# rails-query-command

A Claude Code plugin that teaches Claude to answer data questions about a Rails application using the [`rails query`](https://github.com/rails/rails/pull/57156) command — locally, or against a deployed environment through **any command you configure in an env var** (Heroku, SSH, docker, kubectl, …).

It's a fork of [lewispb/rails-query-skill](https://github.com/lewispb/rails-query-skill), adapted to:

- Work on **Rails 7.2, 8.0 and 8.1** via the [`rails-query-command`](https://github.com/lazaronixon/rails-query-command) gem (and on Rails 8.2+ natively).
- Run remotely through **a command template in `$RAILS_QUERY_CMD`**, plus a small wrapper (`rq`) that makes remote quoting a non-issue.

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

- **`jq`** on your PATH for the skill's extraction patterns (`jq '.rows[0][0]'`).

## Remote setup

Set `RAILS_QUERY_CMD` to a command template. It's the full command that runs `rails query`; `%s` marks where the query arguments go:

```bash
export RAILS_QUERY_CMD='heroku run --no-tty --exit-code --app platform-api-main "cd apps/api && bin/rails query %s"'
```

Other examples:

```bash
export RAILS_QUERY_CMD='ssh deploy@app.example.com "cd /srv/app/current && bin/rails query %s"'
export RAILS_QUERY_CMD='kubectl exec -i deploy/web -- sh -c "bin/rails query %s"'
export RAILS_QUERY_CMD='docker exec -i app sh -c "bin/rails query %s"'
```

Keep `%s` inside the quoted command string for remote commands, so `rq` knows a remote shell will parse it too.

Put it in your shell profile, a `.envrc` (direnv), or Claude Code's `settings.json` `env` block so Claude sees it.

Claude then runs queries through the bundled wrapper:

```bash
rq 'User.count'
rq --sql 'SELECT COUNT(*) FROM users'
rq schema users
```

`rq` (`skills/rails-query/scripts/rq`):

- **quotes the expression for every layer**: your shell, and the remote shell when `%s` sits inside the quoted command string. So parens, `*`, quotes, `$`, `&` and newlines arrive intact;
- validates flags (`--sql`, `--db`, `--page`, `--per`, `-e`) and table names;
- keeps **only the JSON result on stdout** — `heroku run` banners and boot logs go to stderr;
- propagates the exit code (use `--exit-code` with `heroku run`);
- falls back to local `bin/rails query` when `RAILS_QUERY_CMD` is unset.

| Variable | Purpose |
|----------|---------|
| `RAILS_QUERY_CMD` | Command that runs `rails query`; `%s` is replaced with the query arguments (appended if absent) |
| `RQ_DEBUG=1` | Print the expression and the final command to stderr |

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
