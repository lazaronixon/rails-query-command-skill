# Changelog

All notable changes to this plugin are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions match `.claude-plugin/plugin.json`.

## [0.6.2] - 2026-10-24

### Changed

- Remote commands end in `bin/rails query {args}`. `SKILL.md` and the README no longer offer plain `rails query {args}` as an alternative.

## [0.6.1] - 2026-10-24

### Fixed

- `SKILL.md` and the README say a remote command ends in `bin/rails query {args}` or `rails query {args}`, and no longer claim `bundle exec rails` works.

## [0.6.0] - 2026-10-24

### Changed

- The skill is Markdown only. Claude reads `.rails_query_remote` itself and runs the remote's command.
- Claude replaces `{args}` with `-`, or with flags and subcommands (`--sql -`, `explain -`, `schema users`), and pipes the expression in on stdin.
- The expression goes to `rails query -` on stdin, from a quoted heredoc, so no shell quoting is involved and expressions can span several lines.
- The runner's output and exit status come back as is, instead of being filtered down to the JSON line. Claude picks the result line out of runner banners and boot logs.
- Local runs use the same quoted heredoc on stdin as remotes, and every run adds `2>&1` so errors, which go to stderr, aren't lost. Claude picks the result line out of the logs locally too.
- `SKILL.md` rewritten: picking the remote for "in beta" / "in production", keeping remote runs few, result shapes, and never bypassing `rails query` with a raw database connection.
- `SKILL.md` covers what is and isn't paginated (`pluck`, `to_a` and `group.count` come back whole), the `LIMIT` and trailing `-- comment` traps in `--sql`, `explain` needing a relation rather than a `.count`, enums, time zones in daily counts, and checking the schema before guessing columns.
- `SKILL.md` is generic to any Rails app and names no runner: remotes are shown as `<runner> "bin/rails query {args}"`. The README keeps the Heroku, SSH, docker and kubectl examples.
- The README's Heroku examples use `heroku run -a APP -x`.

### Removed

- The `rq` wrapper (`bin/rq`), with its `--remote`/`--remotes` options and flag validation.

## [0.5.0] - 2026-10-23

### Changed

- Remote runs print only the JSON result: to stdout on success, to stderr on failure. Runner banners and boot logs are dropped.
- Without a JSON line, `rq` prints everything the remote printed to stderr and never exits 0.

## [0.4.0] - 2026-10-23

### Changed

- `rq` collects everything the remote prints and prints it once it exits, then exits with the remote's status. The remote gets an empty stdin.
- Every remote command must contain `{args}`; the `$1` placeholder and argument appending are gone.
- `.rails_query_remote` is read from the current directory only: each project keeps its own file at its root.
- Docs name remotes after the environment (`production`, `beta`) and no longer use `jq`.

### Removed

- `--debug`.

## [0.3.0] - 2026-10-22

### Changed

- `.rails_query_remote.yml` is renamed to `.rails_query_remote`, with one `name: command` per line taken literally after the first `: `. No more YAML parsing.

## [0.2.0] - 2026-10-22

### Added

- Named remotes: `rq --remote NAME` looks the command up in `.rails_query_remote.yml`, and `rq --remotes` lists them. Without `--remote`, `rq` runs `bin/rails query` locally.

### Changed

- `--debug` replaces `RQ_DEBUG`.

### Removed

- The `RAILS_QUERY_REMOTE` environment variable.

## [0.1.1] - 2026-10-22

### Changed

- The remote command lives in `RAILS_QUERY_REMOTE` (renamed from `RAILS_QUERY_COMMAND`), with `{args}` marking where the query arguments go.
- `rq` quotes the expression for the remote shell and the local one, replacing the base64 pipe.

## [0.1.0] - 2026-10-21

### Added

- Initial release, forked from [lewispb/rails-query-skill](https://github.com/lewispb/rails-query-skill): the Kamal alias is replaced by a command template in `$RAILS_QUERY_COMMAND` (Heroku, SSH, docker, kubectl), targeting the [rails-query-command](https://github.com/lazaronixon/rails-query-command) gem for Rails 7.2–8.1.
- The `rq` wrapper, which base64-encodes the expression for remote runs, keeps only the JSON result on stdout and propagates exit codes.
