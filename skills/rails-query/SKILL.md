---
name: rails-query
description: |
  Answer any question that requires live data from a Rails app using `rails query`
  (built into Rails 8.2+, backported to Rails 7.2/8.0/8.1 by the `rails-query-command` gem):
  counts, lookups, "how many X", ad-hoc investigation, schema exploration, EXPLAIN plans,
  and aggregates — whether locally or against a deployed environment reached through the
  command in `$RAILS_QUERY_CMD` (Heroku, SSH, docker, kubectl, …). Trigger
  broadly whenever the user asks for data from a Rails app's database, mentions
  `rails query` / `bin/rails query` / `rq` / `rails-query-command`, asks to inspect schema,
  tables, or models, wants to run SQL or ActiveRecord against the database, needs an EXPLAIN
  plan, or wants to query production/staging/replica. Prefer this skill over opening a
  Rails console, `heroku run rails console`, SSH-ing to a server, or writing a one-off
  script for data questions — `rails query` is faster, structured (JSON output),
  audit-friendly, and read-only by construction.
---

# rails-query

`rails query` runs read-only queries against a Rails app's database. Input is a single expression, output is a single JSON object on stdout (`{columns, rows, meta}`; errors go to stderr as JSON with a non-zero exit), and writes are blocked at the connection level. With a read replica configured it hits the replica automatically, so it's safe to point at production.

It ships with Rails 8.2+. On Rails 7.2, 8.0 and 8.1 it comes from the [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) gem (`gem "rails-query-command"` in the Gemfile) — same command, same output.

Prefer it over a console, a script, or SSH for any data question. One invocation, structured output, nothing to clean up.

## The flow

### 1. Know where you're running

- **Local** (`bin/rails query`) — development, tests, or anywhere you have the app checked out.
- **Remote** (`rq`) — the deployed app, reached through the command template in `$RAILS_QUERY_CMD`. `rq` is the wrapper script at `scripts/rq` in this skill's directory (`${CLAUDE_PLUGIN_ROOT}/skills/rails-query/scripts/rq`). Call it by its full path.

Check `echo "$RAILS_QUERY_CMD"`. If it's set and the user is asking about a deployed environment (production, staging, "live", the app name in the template), use `rq`. If it's empty, use `bin/rails query` — or, if the user wants remote data, ask them to set it up (see "Remote setup").

If `bin/rails query` says `Could not find command "query"`, the app is older than Rails 8.2 and doesn't have the gem — check `Gemfile.lock` for `rails-query-command` and suggest adding it.

### 2. Decide: Ruby expression or raw SQL

**Default: Ruby (ActiveRecord).** The expression is `eval`'d in the app context — scopes, associations, finders, aggregates all work. Model logic (encryption, default scopes, polymorphism) is honored.

**`--sql`:** raw schema access, cross-table joins without models, or aggregates awkward in AR.

Telltale sign you forgot `--sql`: `SyntaxError: unexpected *; no anonymous rest parameter` — your SQL got parsed as Ruby. Add `--sql` and retry.

### 3. Inspect the schema if it's unfamiliar

`rails query` has three introspection modes that work the same locally and remotely — crucial when you only have remote access:

```bash
bin/rails query schema           # every table
bin/rails query schema users     # columns + indexes + enums + associations for one table
bin/rails query models           # every AR model with its table and associations
```

Enums matter: if a column is an enum, `User.where(status: "active")` works but `User.where(status: 0)` might silently misfire. `schema <table>` tells you.

**Remotely, cache the schema locally before `jq`-ing it** — each round-trip boots a process on the server (a Heroku one-off dyno takes 10-30s+). Dump `schema` / `models` / `schema <table>` to `/tmp/rails-query-cache-<app>/` once, then `jq` against the files. Skip caching for single-query tasks or local runs.

### 4. For expensive queries, `EXPLAIN` first

```bash
bin/rails query explain 'User.where(active: true).order(:created_at).limit(100)'
bin/rails query explain 'SELECT * FROM users WHERE active = 1' --sql
```

Do this before running full-table scans against production.

### 5. Run the query

```bash
bin/rails query 'User.count'
bin/rails query --sql 'SELECT COUNT(*) FROM users'
```

Watch for `"has_more": true` in the response — pagination is truncating. Re-run with `--page N` or raise `--per`. Don't add your own `LIMIT`; see "Pagination".

### 6. Extract what you need

```bash
bin/rails query 'User.count' | jq '.rows[0][0]'                  # single scalar
bin/rails query 'User.pluck(:email)' | jq -r '.rows[][0]'        # column as array
COUNT=$(bin/rails query 'User.count' | jq -r '.rows[0][0]')      # into a shell var
```

Every example works the same with `rq` in place of `bin/rails query`.

## Command reference

```bash
bin/rails query [OPTIONS] '<expression>'
rq [OPTIONS] '<expression>'
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--sql` | off | Treat `<expression>` as raw SQL instead of Ruby/AR |
| `--db <name>` | — | Explicit database config (e.g. `primary_replica`, `analytics`) |
| `-e <env>` | `development` | Environment (`test`, `production`, …). Don't pass it remotely — the server's `RAILS_ENV` is already right |
| `--page N` | 1 | Page number (1-indexed) |
| `--per N` | 100 | Rows per page (max 10000) |

### Special expressions (no `--sql` needed)

| Expression | What it returns |
|------------|-----------------|
| `schema` | Every table name |
| `schema <table>` | Columns, indexes, enums, and associations for that table |
| `models` | Every AR model with its table and associations |
| `explain <expr>` | `EXPLAIN` plan. Pair with `--sql` for raw SQL |
| `-` (or piped stdin) | Read expression from stdin — useful for long multi-line SQL |

## Pagination

`rails query` paginates automatically. It internally appends `LIMIT per+1` to detect whether more rows exist, which drives `meta.has_more`.

**Don't add your own `LIMIT`.** If the SQL already contains `LIMIT`, the command won't add one — which suppresses the truncation detector. For raw SQL, omit `LIMIT` and let pagination handle it.

```bash
bin/rails query --sql 'SELECT id, email FROM users ORDER BY id' --page 2
bin/rails query --per 500 'User.order(:id)'
```

Always order explicitly when paginating — without `ORDER BY`, page 2 can overlap page 1.

## Remote setup

`$RAILS_QUERY_CMD` is the full command that runs `rails query` on the deployed app. `rq` puts the query arguments (expression, subcommand, flags) where `%s` is. Without `%s`, they're appended to the end.

```bash
# Heroku (monorepo, app in apps/api)
export RAILS_QUERY_CMD='heroku run --no-tty --exit-code --app platform-api-main "cd apps/api && bin/rails query %s"'

# Others
export RAILS_QUERY_CMD='ssh deploy@app.example.com "cd /srv/app/current && bin/rails query %s"'
export RAILS_QUERY_CMD='kubectl exec -i deploy/web -- sh -c "bin/rails query %s"'
export RAILS_QUERY_CMD='docker exec -i app sh -c "bin/rails query %s"'
```

Where `%s` sits decides the quoting. Inside `"…"` or `'…'`, `rq` assumes the string goes to a remote shell (`heroku run`, `ssh`, `sh -c`) and quotes for that shell and for yours. Unquoted, only your local shell parses it. So for remote commands, keep `%s` inside the quoted command string. Use `--no-tty` and `--exit-code` with `heroku run` so output isn't mangled and failures propagate.

Then:

```bash
rq 'User.count'
rq --sql 'SELECT COUNT(*) FROM users'
rq schema users
```

`rq` keeps only the JSON result on stdout. The runner's banners (`Running … on ⬢ app… up, run.1234`) and boot logs go to stderr, so piping to `jq` just works. Add `2>/dev/null` if the noise gets in the way.

### Quoting

There's nothing special to do. `rq` quotes the expression for every layer (local shell, Heroku CLI/ssh, the remote `bash -c`), so parens, `*`, quotes, `$`, `&` and newlines arrive intact. Quote the way you would locally:

```bash
rq --sql "SELECT id FROM users WHERE email = 'alice@example.com'"
rq 'User.where(email: "alice@example.com").pick(:id)'
```

Don't hand-assemble `heroku run "… rails query '…'"` commands yourself — that's where quoting breaks. Always go through `rq`.

## Common patterns

```bash
# Counts and aggregates
bin/rails query 'User.where(active: true).count'
bin/rails query 'User.group(:role).count'

# Lookups
bin/rails query 'User.find_by(email: "alice@example.com")&.as_json'

# Joins and scopes — Ruby form reuses existing scopes/encryption
bin/rails query 'Post.published.joins(:author).where(authors: { verified: true }).count'

# Schema-first discovery
bin/rails query schema orders | jq '.columns[] | {name, type, null}'
bin/rails query models | jq '.[] | select(.table_name == "accounts") | .model'
```

## Safety model

- Writes are blocked at the connection level via `while_preventing_writes` or (when configured) `connected_to(role: :reading)`. `INSERT` / `UPDATE` / `DELETE` raises instead of executing.
- With a read replica configured, queries hit the replica automatically.
- `--db <name>` overrides the connection (e.g. `--db primary_replica`, `--db analytics`).
- Ruby expressions are as trusted as `rails runner` — the write block covers the database, not arbitrary Ruby. Never run expressions with side effects (mailers, jobs, HTTP calls, `update_column` tricks). Use `--sql` when you only need SQL.

`ActiveRecord::ReadOnlyError` (or `Write query attempted while in readonly mode`) means the safety net fired — rework the expression to be read-only.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `SyntaxError: unexpected *` | SQL passed without `--sql` | Add `--sql` |
| `Could not find command "query"` | Rails < 8.2 without the gem, or the remote isn't deployed with it yet | Add `gem "rails-query-command"` and deploy |
| `rq: no JSON in output` | Boot crash, wrong directory, or runner error | Read the stderr above it; re-run with `RQ_DEBUG=1` to see the exact command |
| `bin/rails: No such file or directory` remotely | Wrong `cd` in `$RAILS_QUERY_CMD`, or no binstub | Fix the `cd`, or use `rails query %s` |
| Expression arrives mangled or `syntax error near unexpected token` | `%s` isn't inside the quoted remote command string | Move `%s` inside the `"…"` passed to the remote shell |
| `ActiveRecord::ReadOnlyError` | Expression tried to write | Rework to be read-only |
| `LIMIT 101` in `meta.sql` unexpectedly | Default pagination probe | Expected — drives `meta.has_more` |
| `has_more: true` but you wanted all rows | Default per-page hit | Raise `--per` (max 10000) or paginate with `--page` |
| `ActiveRecord::ConnectionNotEstablished` with `--db` | Database key not in `database.yml` | Check the env's `database.yml` for the exact key |

## Further reading

- [rails/rails#57156](https://github.com/rails/rails/pull/57156) — the PR that added `rails query`
- [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) — the backport gem for Rails 7.2–8.1
