---
name: rails-query
description: |
  Answer any question that requires live data from a Rails app using `rails query`
  (built into Rails 8.2+, backported to Rails 7.2/8.0/8.1 by the `rails-query-command` gem):
  counts, lookups, "how many X", ad-hoc investigation, schema exploration, EXPLAIN plans,
  and aggregates — whether locally or against a deployed environment reached through the
  named remotes in the project's `.rails_query_remote` (Heroku, SSH, docker, kubectl, …). Trigger
  broadly whenever the user asks for data from a Rails app's database, mentions
  `rails query` / `bin/rails query` / `rq` / `rails-query-command`, asks to inspect schema,
  tables, or models, wants to run SQL or ActiveRecord against the database, needs an EXPLAIN
  plan, or wants to query production/staging/replica. Prefer this skill over opening a
  Rails console, `heroku run rails console`, SSH-ing to a server, or writing a one-off
  script for data questions — `rails query` is faster, structured (JSON output),
  audit-friendly, and read-only by construction.
---

# rails-query

`rails query` runs read-only queries against a Rails app's database. Input is a single expression, output is a JSON object (`{columns, rows, meta}`, or `{"error": …}` on failure), and writes are blocked at the connection level. Locally the result goes to stdout and errors to stderr with a non-zero exit; remotely `rq` does the same, keeping only the JSON and dropping the server's boot logs (see "Remote setup"). With a read replica configured it hits the replica automatically, so it's safe to point at production.

It ships with Rails 8.2+. On Rails 7.2, 8.0 and 8.1 it comes from the [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) gem (`gem "rails-query-command"` in the Gemfile) — same command, same output.

Prefer it over a console, a script, or SSH for any data question. One invocation, structured output, nothing to clean up.

## The flow

### 1. Know where you're running

- **Local** (`bin/rails query`) — development, tests, or anywhere you have the app checked out.
- **Remote** (`rq --remote NAME`) — a deployed app, reached through a named remote in the config file. `rq` is the wrapper script at `scripts/rq` in this skill's directory (`${CLAUDE_PLUGIN_ROOT}/skills/rails-query/scripts/rq`). Call it by its full path.

If the user is asking about a deployed environment (production, beta, staging, "live", an app name), run `rq --remotes` to list the configured remotes and pick the one that matches — e.g. "live" → `production`. If several could match or none does, ask which one. If there's no `.rails_query_remote`, ask the user to set it up (see "Remote setup"). Otherwise use `bin/rails query`.

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

**Remotely, cache the schema locally before reading it** — each round-trip boots a process on the server (a Heroku one-off dyno takes 10-30s+). Dump `schema` / `models` / `schema <table>` to `/tmp/rails-query-cache-<app>/` once, then read the files. Skip caching for single-query tasks or local runs.

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

### 6. Read the result

Read the JSON directly. `rows` holds one array per row, in `columns` order:

```json
{"columns":["result"],"rows":[[202485]],"meta":{"row_count":1,"page":1,"per_page":100,"has_more":false,"sql":"User.count"}}
```

A single value (`User.count`) is `rows[0][0]`; a plucked column (`User.pluck(:email)`) is the first element of each row.

Every example works the same with `rq --remote NAME` in place of `bin/rails query`.

## Command reference

```bash
bin/rails query [OPTIONS] '<expression>'
rq --remote NAME [OPTIONS] '<expression>'
rq --remotes
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

Remotes live in `.rails_query_remote` at the root of the project — the Rails app, next to `bin/rails` (in a monorepo, e.g. `apps/api/`). Each project has its own. `rq` only looks in the current directory, so run it from the project root, the same place you run `bin/rails query`. It has one `name: command` per line; blank lines and lines starting with `#` are ignored. The command is everything after the first `: `, the full command that runs `rails query` on the deployed app, taken literally — no extra shell quoting around it. It must contain `{args}`: `rq` puts the query arguments (expression, subcommand, flags) there.

```
# Heroku (monorepo, app in apps/api)
production: heroku run --no-tty --exit-code --app platform-api-main "cd apps/api && bin/rails query {args}"
beta: heroku run --no-tty --exit-code --app platform-api-beta "cd apps/api && bin/rails query {args}"

# Other runners — name each after its environment (production, staging, …)
ssh: ssh -n deploy@app.example.com "cd /srv/app/current && bin/rails query {args}"
k8s: kubectl exec deploy/web -- sh -c "bin/rails query {args}"
docker: docker exec app sh -c "bin/rails query {args}"
```

Name remotes after the environment (`production`, `beta`) so it's obvious which one a question refers to — the file belongs to one project, so the app name isn't needed. Names may use letters, digits, `_`, `.` and `-`. Commit the file to share remotes with the team, or add it to `.gitignore` to keep them personal.

Where `{args}` sits decides the quoting. Inside `"…"` or `'…'`, `rq` assumes the string goes to a remote shell (`heroku run`, `ssh`, `sh -c`) and quotes for that shell and for yours. Unquoted, only your local shell parses it. So for remote commands, keep `{args}` inside the quoted command string. With `heroku run`, add `--exit-code` so a failed query exits non-zero, and `--no-tty` so it never attaches your terminal.

Then:

```bash
rq --remotes
rq --remote beta 'User.count'
rq -r production --sql 'SELECT COUNT(*) FROM users'
rq -r beta schema users
```

`rq` prints only the JSON result, once the remote exits — the runner's banners and the server's boot logs are dropped. On success it goes to stdout; on failure the JSON error goes to stderr. `rq` exits with the remote command's status. Most failures, including errors while the app boots (bad credentials, a missing ENV var), come back from `rails query` as that JSON error. When there's no JSON at all — a crash `rails query` couldn't report, a wrong `cd`, a runner error — `rq` prints everything the remote printed to stderr instead and exits non-zero; the backtrace or the runner's error is in there.

### Quoting

There's nothing special to do. `rq` quotes the expression for every layer (local shell, Heroku CLI/ssh, the remote `bash -c`), so parens, `*`, quotes, `$`, `&` and newlines arrive intact. Quote the way you would locally:

```bash
rq -r production --sql "SELECT id FROM users WHERE email = 'alice@example.com'"
rq -r production 'User.where(email: "alice@example.com").pick(:id)'
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
bin/rails query schema orders
bin/rails query models
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
| No JSON in `rq`'s output | Boot crash, wrong directory, or runner error | Read the output — the error is there |
| `rq: no .rails_query_remote …` / `rq: unknown remote` | Remote not configured | Add a `name: command` entry to `.rails_query_remote` at the project root and run `rq` from there; `rq --remotes` lists what's there |
| `bin/rails: No such file or directory` remotely | Wrong `cd` in the remote's command, or no binstub | Fix the `cd`, or use `rails query {args}` |
| Expression arrives mangled or `syntax error near unexpected token` | `{args}` isn't inside the quoted remote command string | Move `{args}` inside the `"…"` passed to the remote shell |
| `ActiveRecord::ReadOnlyError` | Expression tried to write | Rework to be read-only |
| `LIMIT 101` in `meta.sql` unexpectedly | Default pagination probe | Expected — drives `meta.has_more` |
| `has_more: true` but you wanted all rows | Default per-page hit | Raise `--per` (max 10000) or paginate with `--page` |
| `ActiveRecord::ConnectionNotEstablished` with `--db` | Database key not in `database.yml` | Check the env's `database.yml` for the exact key |

## Further reading

- [rails/rails#57156](https://github.com/rails/rails/pull/57156) — the PR that added `rails query`
- [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) — the backport gem for Rails 7.2–8.1
