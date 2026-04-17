# Bash Coding Standard

This standard applies to `updateRepos` and any future Bash scripts in this
repository.

## Runtime Contract

- Support macOS and Ubuntu.
- Support macOS Bash 3.2.
- Use Bash explicitly with `#!/bin/bash`.
- Do not require GNU-only command options.
- Keep dependencies limited to standard shell tools and Git.
- Do not require `rg`.
- Treat GitHub CLI (`gh`) as optional unless `--github-user` is supplied.
- For `--github-user`, prefer an executable `gh` next to `updateRepos`, then
  fall back to `gh` from `PATH`.

## Script Safety

- Start scripts with:

  ```bash
  set -o errexit -o pipefail -o nounset
  ```

- Include an `ERR` trap that reports line, command, and exit code.
- Do not use destructive commands unless explicitly requested.
- Do not create the clone root automatically.

## Formatting

- Use two-space indentation.
- Use ASCII text.
- Keep lines readable.
- Use `camelCase` function names.
- Use `UPPER_SNAKE_CASE` globals.
- Use `camelCase` local variables.

## Structure

- Keep context globals near the top.
- Keep helpers small and focused.
- Prefer guard clauses over nested `else` blocks.
- Keep parsing, validation, initialization, execution, and summary steps obvious
  in the main flow.

## Input Handling

- Trim flag values.
- Strip carriage returns from text input.
- Repo files use `[org]` sections.
- Repo files may contain `repo=localPath` entries.
- Skip blank lines.
- Skip lines where the trimmed line starts with `#`.
- Do not support inline comments unless explicitly requested.
- Repo file parse errors must include filename and line number.
- Fail clearly when an explicit file argument does not exist or is unreadable.

## Repositories

- Clone with SSH only:

  ```text
  git@github.com:<org>/<repo>.git
  ```

- Do not introduce HTTPS clone behavior.
- Store resolved entries as `org`, `repo`, and `localPath`.
- Sort and dedupe exact resolved entries before processing.
- Fail on duplicate local destination paths.
- If no repositories are supplied by flags, load `<clone-root>/repos`.
- If no repositories are found, fail with `FATAL:`.

## Command Execution

- Build commands with arrays.
- Execute commands through a shared helper.
- In dry-run mode, the helper prints instead of executing.
- Per-repo `mkdir`, `git clone`, and `git pull` failures are non-fatal:
  print `ERROR:` to stderr, increment failed count, and continue.

## Output

- Warnings start with `WARN:`.
- Per-repo command failures start with `ERROR:` and go to stderr.
- Fatal validation failures start with `FATAL:` and go to stderr.
- Help text may remain on stdout.
- Summary includes updated, cloned, skipped, and failed counts.

## Validation

At minimum, run:

```bash
bash -n updateRepos
./updateRepos --dry-run --org my-org --repo app
./updateRepos --dry-run --repos-file ./repos
```

When changing argument parsing or repository loading, also validate:

```bash
./updateRepos --dry-run --repo my-org/app=my-local-app
./updateRepos --dry-run --clone-root /path/that/does/not/exist
./updateRepos --dry-run --repos-file /path/that/does/not/exist
```
