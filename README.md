# rails-query-command

A Claude Code plugin that teaches Claude to answer data questions about a Rails application using the [`rails query`](https://github.com/rails/rails/pull/57156) command — locally, or against a deployed environment through **named remotes you configure in a config file** (Heroku, SSH, docker, kubectl, …).

It's a fork of [lewispb/rails-query-skill](https://github.com/lewispb/rails-query-skill), adapted to:

- Work on **Rails 7.2, 8.0 and 8.1** via the [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) gem (and on Rails 8.2+ natively).
- Run remotely through **named command templates in your app's `.rails_query_remote`**. The skill is Markdown only: no scripts to install, just `bash` on Claude's side.

## What it does

When you ask Claude something like:

- "How many active users do we have in production?"
- "Show me the orders table structure."
- "What models reference `accounts`?"
- "Explain the query plan for this scope."

…Claude reaches for `rails query` instead of opening `heroku run rails console`, SSHing to a server, or writing a one-off script. Queries are structured (JSON output), read-only by construction (writes blocked at the connection level), and safe to run against production with a read replica.

## Installation

```shell
claude plugin marketplace add lazaronixon/rails-query-command-skill
claude plugin install rails-query-command@rails-query-command-skill
```

Or, from inside a Claude Code session:

```
/plugin marketplace add lazaronixon/rails-query-command-skill
/plugin install rails-query-command@rails-query-command-skill
/reload-plugins
```

## Requirements

- **`rails query` in the app.** Rails 8.2+ has it built in. On Rails 7.2–8.1 add the gem and deploy:

  ```ruby
  gem "rails-query-command"
  ```

## Remote setup

Create `.rails_query_remote` at the root of your project (the Rails app, next to `bin/rails`), with one `name: command` per line. Each project has its own: in a monorepo, each Rails app (e.g. `apps/api`) gets its own file. Each command is the full command that runs `rails query`, taken literally (everything after the first `: `), and it ends in `bin/rails query {args}`:

```
# Example
production: heroku run -a platform-api-main -x "apps/api/bin/rails query {args}"
beta: heroku run -a platform-api-beta -x "apps/api/bin/rails query {args}"

# Other runners
ssh: ssh deploy@app.example.com "/srv/app/current/bin/rails query {args}"
k8s: kubectl exec -i deploy/web -- sh -c "bin/rails query {args}"
docker: docker exec -i app sh -c "bin/rails query {args}"
```

Blank lines and lines starting with `#` are ignored. Commit it to share remotes with your team, or add it to `.gitignore` to keep them personal.

Name remotes after the environment (`production`, `beta`) so Claude can match "in beta" or "in production" to the right one. Keep `bin/rails query {args}` at the end of the command string, and let the command pass its stdin through: no `-n` on `ssh`, `-i` on `docker exec` and `kubectl exec`.

Claude reads the file, picks the remote for the environment you named, replaces `{args}` (`-` for a Ruby expression, `--sql -`, `explain -`, `schema users`, …) and runs the command, with the expression on stdin. `rails query -` reads the expression from stdin, so it never passes through a shell: quotes, `$`, parens and newlines arrive intact, and expressions can span several lines:

```bash
heroku run -a platform-api-beta -x "apps/api/bin/rails query -" <<'EOF'
recent = User.where(created_at: 30.days.ago..)
{obrien: recent.where(last_name: "O'Brien").count, total: recent.count}
EOF
```

Output is left alone: runner banners, boot logs and the JSON result come through as the runner prints them, with its exit status.

## Auto mode

If auto mode blocks production runs with `[Production Reads]`, add this to `~/.claude/settings.json`:

```json
{
  "autoMode": {
    "allow": [
      "$defaults",
      "Rails Query: `bin/rails query` blocks writes, so production data queries aren't production reads unless they read secrets."
    ]
  }
}
```

## How triggering works

The skill is described broadly so Claude invokes it whenever a task would naturally be solved by querying a Rails database. Try:

> "How many users registered last week in production?"

> "What's in the `mailboxes` table?"

> "Explain this query: `Post.published.limit(100)`"

## Further reading

- [rails/rails#57156](https://github.com/rails/rails/pull/57156) — the PR that added `rails query`.
- [rails-query-command](https://github.com/lazaronixon/rails-query-command) — the backport gem.
- [lewispb/rails-query-skill](https://github.com/lewispb/rails-query-skill) — the original skill this is forked from.

## License

MIT
