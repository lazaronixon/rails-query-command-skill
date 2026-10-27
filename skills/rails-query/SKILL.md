---
name: rails-query
description: |
  Answer questions needing live data from a Rails app's database with `rails query`
  (Rails 8.2+, or 7.2–8.1 with the rails-query-command gem): counts, lookups, aggregates,
  investigation, schema/model/enum exploration, EXPLAIN plans, locally or on a deployed
  environment (production, beta, staging…) via remotes in `.rails_query_remote`. Trigger
  whenever the user asks for Rails database data, says "in production" / "in beta" about
  data, mentions `rails query`, wants to inspect tables, columns or models, or run SQL or
  ActiveRecord. Prefer it over a console, server shell or one-off script: read-only, JSON output.
---

# rails-query

`rails query` runs one read-only expression against the app's database and prints one JSON object. Writes are blocked at the connection level and a configured read replica is used, so it is safe on production. Rails 8.2+ ships it; 7.2–8.1 needs `gem "rails-query-command"`.

Use it for every data question instead of a console, runner script or server shell. Never go around the app with a database client and credentials from `DATABASE_URL`, `.env` or `database.yml`: that skips the read-only guard and model logic (enums, scopes, encryption). If `rails query` can't run, say why instead of falling back to a raw connection.

## How to run it

Pass the expression on stdin with a quoted heredoc, locally and on remotes alike:

```bash
bin/rails query - <<'EOF'
recent = User.where(created_at: 30.days.ago..)
{obrien: recent.where(last_name: "O'Brien").count,
 dollar: recent.where("email LIKE ?", "%$%").count}
EOF
```

- **`-` reads the expression from stdin.** The quoted heredoc (`<<'EOF'`) passes it byte for byte, so quotes, `$` and newlines need no escaping. Never put the expression in the command string, where an apostrophe like `O'Brien` breaks the quoting.
- **The result is the last line** starting with `{"columns"`, `{"table"`, `{"error"` or `[{"model"`. Logs often surround it (SQL debug logs locally; runner banners, boot output or an echo of the expression on remotes). No such line means the command failed (boot crash, auth, wrong path): read the rest of the output.

Flags and subcommands go before the `-`. `schema` and `models` take no expression: they replace the `-` and get `< /dev/null` (`schema -` would look for a table named `-`).

| Query | Command | stdin |
|---|---|---|
| Ruby (default) | `bin/rails query -` | heredoc |
| SQL | `bin/rails query --sql -` | heredoc |
| Flags | `bin/rails query --per 500 -`, `bin/rails query --db primary_replica -` | heredoc |
| EXPLAIN | `bin/rails query explain -`, `bin/rails query --sql explain -` | heredoc |
| Schema, models | `bin/rails query schema`, `bin/rails query schema users`, `bin/rails query models` | `< /dev/null` |

Run from the directory with `bin/rails` (in a monorepo, the app's subdirectory); `.rails_query_remote` lives there too.

## Choosing the environment

No environment named: run locally and say the numbers are development data.

Environment named ("in beta", "on prod"): read `.rails_query_remote` and pick the matching remote (`beta`, `app_beta`, `api-beta` all match "beta"). Query only that environment. If none matches, or two could, ask: the wrong environment gives a confidently wrong answer. If the file doesn't exist, explain how to add one (see "Remote setup") rather than improvising remote commands.

### Running a remote

Replace `{args}` in the remote's command with what follows `bin/rails query` in the table above, and run it otherwise exactly as written, with the same heredoc (`<runner>` is the part before the quoted `rails query`):

```bash
<runner> "bin/rails query -" <<'EOF'
User.where(created_at: 30.days.ago..).group(:plan).count
EOF
```

Leave the rest (runner, flags, path) alone. If it looks wrong or doesn't end in `bin/rails query {args}`, say so rather than fixing it silently.

Each remote run boots the app, often 10–30 seconds, so:

- Answer several numbers in **one** expression returning a Hash, as in the first example.
- Prefer the local schema file and `app/models` over a remote `schema` call when the code answers the question; deployed code is usually close to the checked-out branch.

## Writing the expression

**Ruby (ActiveRecord) by default.** It is `eval`'d in the app, so scopes, associations, enums and encryption apply. Multi-line is fine; the last value is the result.

**Keep it simple and readable.** Write the expression the way you'd write it in the app: idiomatic ActiveRecord or plain SQL, named variables for repeated pieces, one clause per line when it grows, no clever one-liners. The user may read or rerun it.

**`--sql`** for raw tables, joins without models, or aggregates awkward in AR. `SyntaxError: unexpected *` means SQL was parsed as Ruby; add `--sql`.

**Check an unfamiliar schema first**, or a guessed column costs a round trip. Read `db/schema.rb` (or `db/structure.sql`) and the model, or run `schema TABLE`, which also lists associations and `enums` (absent when there are none or no model). For enums use `User.where(status: "active")`; `status: 0` may silently misfire.

**Time zones.** Ruby ranges like `30.days.ago..` use the app's zone; SQL dates (`DATE(created_at)`) use the database's, usually UTC. Daily counts can differ near midnight, so say which you used.

### Pagination and large results

Only **relations** and **`--sql` queries** are paginated: `LIMIT per+1` is appended (`per` default 100, max 10000) and `meta.has_more` is `true` when more rows exist. Use `--page N` or a larger `--per`, and order explicitly so pages don't overlap.

Everything else (`pluck`, `to_a`, `map`, `group(...).count`, other Hashes and Arrays) comes back whole with `has_more: false`, however big. On a large table, add `.limit` or count first.

Your own `LIMIT` disables pagination (fine for "the 10 most recent"). In `--sql`, `LIMIT` anywhere, even a subquery, counts as yours, and a trailing `-- comment` swallows the appended `LIMIT`.

### EXPLAIN

Run `explain` before heavy production queries (full scans, unindexed filters, large joins). In Ruby the expression must be a **relation**: `User.where(active: true)` works, `.count` on it fails with `undefined method 'to_sql' for ... Integer`. Explain the relation, then count.

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

- Writes raise `ActiveRecord::ReadOnlyError` (`reading` role if configured, else `while_preventing_writes`). If the user wants data changed, say `rails query` is the wrong tool; don't work around the guard.
- Ruby is still evaluated like `rails runner`; protection covers the database only, so keep to queries: no mailers, jobs, HTTP calls or cache writes.
- Don't read `ENV`, credentials or secrets: the result lands in the transcript.

## Remote setup

`.rails_query_remote` sits next to `bin/rails`. One `name: command` per line; everything after the first `: ` is a shell command ending in `bin/rails query {args}`. Blank lines and `#` comments are ignored.

```
production: <runner for production> "bin/rails query {args}"
beta: <runner for beta> "bin/rails query {args}"
```

`<runner>` runs a command where the app is deployed. It must forward stdin to `bin/rails query` and close it at the end (some runners need a flag for that; some have one that disables it). Name remotes after environments.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `"rows":[[null]]` with `"sql":""`, or with `--sql` an error near ` LIMIT 101` | Nothing arrived on stdin: an empty expression | Put the heredoc on the same command, and make sure the runner forwards stdin |
| Remote hangs with no result | Runner keeps stdin open | Check the runner's stdin flags; for `schema`/`models` use `< /dev/null` |
| `ConnectionNotEstablished` with `--db` | Name not in `database.yml` | Use the exact key from `database.yml` |
| `rails query` unknown command | Rails < 8.2 without the gem, or not deployed | Add `gem "rails-query-command"` and deploy |