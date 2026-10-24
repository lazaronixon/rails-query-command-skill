---
name: rails-query
description: |
  Answer any question that needs live data from a Rails app's database using `rails query`
  (Rails 8.2+, or Rails 7.2–8.1 with the rails-query-command gem): counts, "how many X",
  lookups, aggregates, ad-hoc investigation, schema and model exploration, enums, and
  EXPLAIN plans, either locally or against a deployed environment (production, beta,
  staging…) through named remotes in the app's `.rails_query_remote`. Trigger broadly
  whenever the user asks for data from a Rails database, says "in production" / "in beta"
  about data, mentions `rails query` or `bin/rails query`, wants to inspect tables, columns
  or models, or wants to run SQL or ActiveRecord against the database. Prefer this over a
  remote console, a shell on the server, or a one-off script for data questions: it is
  read-only by construction and returns structured JSON.
---

# rails-query

`rails query` runs one read-only expression against the app's database and prints one JSON object. Writes are blocked at the connection level, and a configured read replica is used automatically, so it is safe to point at production. It ships with Rails 8.2+; on Rails 7.2–8.1 the app needs `gem "rails-query-command"`.

Use it instead of a console, a runner script or a shell on the server for any data question: one invocation, structured output, nothing to clean up. Don't go around the app with a database client and credentials from `DATABASE_URL`, `.env` or `database.yml`: that skips the read-only guard and the models' logic (enums, scopes, encryption). If `rails query` can't run, say why instead of falling back to a raw connection.

## How to run it

Pass the expression on stdin with a quoted heredoc, locally and on remotes alike:

```bash
bin/rails query - 2>&1 <<'RUBY'
recent = User.where(created_at: 30.days.ago..)
{obrien: recent.where(last_name: "O'Brien").count,
 dollar: recent.where("email LIKE ?", "%$%").count}
RUBY
```

- **`-` reads the expression from stdin**, and the quoted heredoc (`<<'RUBY'`, `<<'SQL'`) delivers it byte for byte: no shell parses it, so quotes, `$` and newlines need no escaping. Write the expression exactly as it should run. Never put it in the command string, where one apostrophe in `O'Brien` breaks the quoting.
- **`2>&1`**: results go to stdout but errors (`{"error": ...}`) go to stderr. Without it a failed query looks like empty output.
- **The result is the last line** starting with `{"columns"`, `{"table"`, `{"error"` or `[{"model"`. Logs often surround it, even locally (development logs SQL at debug level; remotes add runner banners and boot output, and some runners echo the expression back first), so pick that line out. No such line means the command itself failed (boot crash, auth, wrong path): read the rest of the output for why.

Flags and subcommands go before the `-`. `schema` and `models` take no expression, so they replace the `-` and get `< /dev/null` instead of a heredoc (`schema -` would look for a table named `-`).

| Query | Command | stdin |
|---|---|---|
| Ruby (default) | `bin/rails query -` | heredoc |
| SQL | `bin/rails query --sql -` | heredoc |
| Flags | `bin/rails query --per 500 -`, `bin/rails query --db primary_replica -` | heredoc |
| EXPLAIN | `bin/rails query explain -`, `bin/rails query --sql explain -` | heredoc |
| Schema, models | `bin/rails query schema`, `bin/rails query schema users`, `bin/rails query models` | `< /dev/null` |

Run from the project root: the Rails app's directory, the one with `bin/rails` (in a monorepo, the app's subdirectory). Its `.rails_query_remote` lives there too.

## Choosing the environment

- **Local** (development data): `bin/rails query ...` as above.
- **Remote** (a deployed environment): the command from `.rails_query_remote`.

When the user names an environment ("in beta", "on prod"), read `.rails_query_remote` and pick the remote whose name matches (`beta`, `app_beta`, `api-beta` all match "beta"). Query only the environment the user asked about. If none matches, or two could, ask instead of guessing: production and beta data are not interchangeable, and the wrong one produces a confidently wrong answer. When the user names no environment, local is the default; say that the numbers are development data.

If there is no `.rails_query_remote`, tell the user how to add one (see "Remote setup") rather than improvising remote commands by hand.

### Running a remote

Each line of `.rails_query_remote` is `name: command`, and the command ends in `bin/rails query {args}`. Replace `{args}` with what follows `bin/rails query` in the table above, and run the command otherwise exactly as written, with the same heredoc and `2>&1`. Here `<runner>` stands for the part of the remote's command before the quoted `rails query`:

```bash
<runner> "bin/rails query -" 2>&1 <<'RUBY'
User.where(created_at: 30.days.ago..).group(:plan).count
RUBY
```

`{args}` becomes `--sql -` for SQL, `explain -` for EXPLAIN, and `schema users` for schema, with `< /dev/null` instead of a heredoc. Leave the rest of the command alone: runner, flags, path. If it looks wrong, or doesn't end in `bin/rails query {args}`, say so rather than fixing it silently.

Each remote run usually boots the app in a fresh process, often 10–30 seconds. Plan before running:

- Answer several numbers in **one** expression that returns a Hash, as in the first example.
- Read `db/schema.rb` and `app/models` locally instead of a remote `schema` call when the code answers the question. Deployed code is usually close to the checked-out branch.

## Writing the expression

**Ruby (ActiveRecord) by default.** The expression is `eval`'d in the app, so scopes, associations, enums, encryption and default scopes all apply. It can span several lines; the value of the last one is the result.

**`--sql`** for raw table access, joins without models, or aggregates awkward in AR. If you see `SyntaxError: unexpected *`, SQL was parsed as Ruby; add `--sql`.

**Look at the schema first when it's unfamiliar.** Guessing a column name costs a round trip (`column users.last_name does not exist`). Read `db/schema.rb` and the model, or run `schema TABLE`, which also lists enums and associations. For an enum column, `User.where(status: "active")` is right, while `User.where(status: 0)` may silently misfire; `schema TABLE` reports enums under `enums` (absent when the table has none, or no model).

**Time zones.** Ruby ranges like `30.days.ago..` use the app's time zone; raw SQL dates (`DATE(created_at)`) are in the database's, usually UTC. Day-by-day counts can differ between the two near midnight, so say which you used.

### Pagination and large results

Only **relations** and **`--sql` queries** are paginated: the command appends `LIMIT per+1` (default `per` 100, max 10000) and sets `meta.has_more` to `true` when there are more rows. Fetch the rest with `--page N`, or raise `--per`. Order explicitly when paging, or pages can overlap.

Everything else comes back whole, with `has_more: false` no matter how big it is: `pluck`, `to_a`, `map`, `group(...).count` and other Hashes and Arrays. On a large table, add a `.limit` or count first.

Your own `LIMIT` turns pagination off: the query runs as written and `has_more` stays `false`. That's fine for "the 10 most recent" questions. Two `--sql` traps: the word `LIMIT` anywhere (even in a subquery) counts as yours, and a trailing `-- comment` swallows the appended `LIMIT`, so keep comments off the last line.

### EXPLAIN

Run `explain` before heavy queries against production (full scans, unindexed filters, large joins). With Ruby, the expression must be a **relation**: `explain` calls `to_sql` on it, so `User.where(active: true)` works and `User.where(active: true).count` fails with `undefined method 'to_sql' for ... Integer`. Explain the relation, then run the count.

## Output shapes

```json
{"columns": ["id", "email"], "rows": [[1, "a@x.com"]],
 "meta": {"row_count": 1, "query_time_ms": 4.2, "page": 1, "per_page": 100, "has_more": false, "sql": "SELECT ..."}}
```

| Expression's value | `columns` | `rows` |
|-------|-----------|--------|
| Relation (`User.where(...)`) | selected columns | one per record, paginated |
| Single record | attribute names | one row |
| Hash (`group(:x).count`, `{a: 1, b: 2}`) | `["key", "value"]` | one per pair |
| Array of records | attribute names | one per record |
| Other Array (`pluck`) | `column_0`, … | one per element |
| Scalar (`count`, `sum`) | `["result"]` | `[[value]]` |

## Safety model

- Writes are blocked: queries use the `reading` role when configured, otherwise `while_preventing_writes`. `INSERT`/`UPDATE`/`DELETE`, `delete_all` and `update!` raise `ActiveRecord::ReadOnlyError`. If the user wants data changed, `rails query` is the wrong tool: say so instead of looking for a way around the guard.
- Ruby expressions are still evaluated as Ruby (as trusted as `rails runner`). Write protection covers the database, not arbitrary code, so keep expressions to queries: no mailers, jobs, HTTP calls or cache writes.

## Remote setup

`.rails_query_remote` sits next to `bin/rails`; in a monorepo, each Rails app has its own. One `name: command` per line; everything after the first `: ` is a shell command ending in `bin/rails query {args}`, which you replace per query (`-`, `--sql -`, `schema users`). Blank lines and `#` comments are ignored.

```
production: <runner for production> "bin/rails query {args}"
beta: <runner for beta> "bin/rails query {args}"
```

`<runner>` is whatever runs a command where the app is deployed. `bin/rails query {args}` must end the command string, and the runner must pass its stdin on to `bin/rails query` and close it at the end: some runners need a flag to forward stdin, and some have a flag that disables it. Name remotes after environments so requests like "in beta" map to them directly.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `"rows":[[null]]` with `"sql":""`, or with `--sql` an error near ` LIMIT 101` | Nothing arrived on stdin: an empty expression | Put the heredoc on the same command, and make sure the runner forwards stdin |
| Remote hangs with no result | Runner keeps stdin open | Check the runner's stdin flags; for `schema`/`models` use `< /dev/null` |
| `ConnectionNotEstablished` with `--db` | Name not in `database.yml` | Use the exact key from `database.yml` |
| `rails query` unknown command | Rails < 8.2 without the gem, or not deployed | Add `gem "rails-query-command"` and deploy |