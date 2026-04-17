# updateRepos

`updateRepos` clones missing GitHub repositories and updates existing ones.

It is meant for keeping a local folder of related GitHub repos current. It uses
SSH clone URLs only:

```text
git@github.com:<org>/<repo>.git
```

## Requirements

- Bash
- Git
- SSH access to the GitHub repositories you want to clone
- GitHub CLI (`gh`) only if you use `--github-user`

The script supports macOS and Ubuntu, including the older Bash version that
ships with macOS.

## Quick Start

Preview what would happen:

```bash
./updateRepos --dry-run --org my-org --repo app
```

Clone or update that repo:

```bash
./updateRepos --org my-org --repo app
```

Use a repo list file:

```bash
./updateRepos --repos-file ./repos
```

By default, repos are cloned under the current directory. Use `--clone-root` to
choose a different destination:

```bash
./updateRepos --clone-root ~/projects --repos-file ./repos
```

## Repo Lists

If you run `./updateRepos` without `--repo` or `--repos-file`, it reads:

```text
<clone-root>/repos
```

The file is grouped by GitHub organization:

```text
[my-org]
api
frontend=web-app
worker=services/worker

[another-org]
shared-tools
api=another-org-api
```

Each entry is either:

```text
repo
repo=local/path
```

For example, `frontend=web-app` clones `git@github.com:my-org/frontend.git`
into `<clone-root>/web-app`.

Rules:

- Blank lines are ignored.
- Full-line comments starting with `#` are ignored.
- Repo entries must appear under an `[org]` section.
- Inline comments are not supported.
- Local paths must be relative.
- Local paths cannot contain `..`, backslashes, or end with `/`.
- Repo list files do not support `org/repo`; use `[org]` sections instead.

## Command-Line Repos

`--repo` is useful for one-off runs:

```bash
./updateRepos --org my-org --repo app
./updateRepos --org my-org --repo frontend=web-app
./updateRepos --repo my-org/app
./updateRepos --repo my-org/frontend=web-app
```

When `--repo app` does not include an organization, the current `--org` value is
used. If no `--org` was supplied, the basename of the clone root is used.

Supplying any `--repo` or `--repos-file` skips the default `<clone-root>/repos`
file.

## GitHub Users

Use `--github-user` when you need to switch the active GitHub CLI account before
processing repos:

```bash
./updateRepos --github-user my-user --repos-file ./repos
```

This runs:

```bash
gh auth switch --user my-user
```

When this flag is used, `updateRepos` first looks for an executable `gh` next to
the script, then falls back to `gh` from `PATH`. It does not run
`gh auth setup-git`, and it does not change clone URLs to HTTPS.

## What It Does

For each configured repo:

- If the local path is already a Git repository, it runs `git pull --ff-only`.
- If the local path exists but is not a Git repository, it prints `WARN:` and
  skips it.
- If the local path does not exist, it clones the repo.

Clone, pull, and parent-directory creation failures print `ERROR:` and the run
continues with the next repo. Fatal validation problems print `FATAL:` and stop
before any repo is processed.

## Options

```text
--clone-root <path>    Directory where repositories are cloned or updated
--org <name>           Set organization for later --repo values
--repo <value>         Add a repository; may be repeated
--repos-file <path>    Read repositories from a file; may be repeated
--github-user <name>   Switch GitHub CLI user before git operations
--dry-run              Print commands instead of running them
-h, --help             Show help
```

## Checks

Run these before changing the script:

```bash
bash -n updateRepos
./updateRepos --dry-run --org my-org --repo app
./updateRepos --dry-run --repo my-org/app=my-local-app
./updateRepos --dry-run --repos-file ./repos
```
