# updateRepos

`updateRepos` is a portable Bash tool for cloning and updating GitHub
repositories by SSH.

This project is implemented from [PLAN.md](PLAN.md). The current `repos` file
is a sample repository definition file.

## Repository Layout

```text
.
├── AGENTS.md
├── PLAN.md
├── README.md
├── repos
├── standards/
│   ├── README.md
│   └── coding/
│       └── bash.md
└── updateRepos
```

## Requirements

- Bash
- Git
- SSH access to the target GitHub organization
- Optional: GitHub CLI (`gh`) when using `--github-user`

The script is intended to run on macOS and Ubuntu. Do not assume `rg` is
installed.

## Repositories File

The default repository file path is:

```text
<clone-root>/repos
```

The checked-in `repos` file is a sample template:

```text
# Sample repository definitions for updateRepos.
#
# Copy this file to your clone root as "repos" and replace the sample org and
# repo names with real values.
#
# Format:
#   [org-name]
#   repo-name
#   repo-name=relative/local/path

[your-org]

# Clone into <clone-root>/api
api

# Clone into <clone-root>/web-app
frontend=web-app

# Clone into a relative subdirectory.
worker=services/worker

[another-org]

# Multiple org sections are allowed.
shared-tools

# The same repo name from a different org is allowed only when the local path is unique.
api=another-org-api
```

Format:

- Blank lines are ignored.
- Lines where the trimmed line starts with `#` are ignored.
- `[org-name]` starts an organization section.
- `repo` clones to `<clone-root>/repo`.
- `repo=localPath` clones to `<clone-root>/localPath`.
- Whitespace around `=` is allowed and trimmed.
- Whitespace inside `[ org ]` is allowed and trimmed.
- Repo entries before the first `[org]` section are fatal errors.
- Inline comments are not supported.

## Usage

```bash
./updateRepos [options]
```

Options:

```text
--clone-root <path>    Directory where repositories are cloned or updated.
--org <name>           Set organization for later --repo values.
--repo <value>         Add a repository; may be repeated.
--repos-file <path>    Read repositories from a file; may be repeated.
--github-user <name>   Switch GitHub CLI user before git operations.
--dry-run              Print commands instead of executing them.
-h, --help             Show help.
```

`--repo` supports:

```text
repo
repo=localPath
org/repo
org/repo=localPath
```

## Examples

Use `<clone-root>/repos`. By default, clone root is the current directory:

```bash
./updateRepos
```

Preview work without running commands:

```bash
./updateRepos --dry-run
```

Use a specific clone root:

```bash
./updateRepos --clone-root ~/my-projects
```

Process explicit repositories:

```bash
./updateRepos --org my-org --repo app --repo api=services/api
```

Use explicit org/repo syntax:

```bash
./updateRepos --repo my-org/app --repo other-org/tooling=tools/tooling
```

Read repositories from a custom file:

```bash
./updateRepos --repos-file ./team-repos
```

Switch GitHub CLI user before processing:

```bash
./updateRepos --github-user my-github-user
```

When `--github-user` is used, `updateRepos` looks for `gh` next to the
`updateRepos` script first, then falls back to `gh` from `PATH`.

## Operational Notes

- Clone URLs are SSH-only:

  ```text
  git@github.com:<org>/<repo>.git
  ```

- Existing Git repositories are updated with `git pull --ff-only`.
- Existing paths that are not Git repositories are skipped with `WARN:`.
- Per-repo clone, pull, and parent-directory creation failures print `ERROR:`
  and continue.
- Fatal validation errors print `FATAL:` and stop before repository processing.
- Summary includes updated, cloned, skipped, and failed counts.
- If any repo fails, the script exits `1` after the summary.

## Validation

Syntax check:

```bash
bash -n updateRepos
```

Dry-run examples:

```bash
./updateRepos --dry-run
./updateRepos --dry-run --org my-org --repo app
./updateRepos --dry-run --repos-file ./repos
```
