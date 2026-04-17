# Agent Instructions

This repository contains a Bash tool for cloning and updating GitHub
repositories by SSH.

Build and maintain the project from [PLAN.md](PLAN.md). Treat the current
implementation as disposable if it conflicts with the plan.

## Primary Goal

Keep `updateRepos` simple, portable, and predictable.

Do not add new behavior unless the user explicitly asks for it. When the request
is ambiguous, update or ask about the plan before implementation.

## Non-Negotiable Behavior

- Clone URLs must remain SSH-only:
  `git@github.com:<org>/<repo>.git`
- Do not switch clone behavior to HTTPS.
- Do not add authentication behavior beyond what is requested.
- Use `--github-user`, not `--github-account`.
- `--github-user` requires `gh`; missing `gh` is fatal only when that flag is
  supplied.
- When `--github-user` is supplied, use `gh` next to `updateRepos` first when
  present and executable, otherwise use `gh` from `PATH`.
- Use `--clone-root`, not `--root`.
- Default clone root is the current working directory.
- Repo files use `[org]` sections.
- Repo files may contain `repo=localPath` aliases.
- Repo files do not support `org=...`.
- Repo files do not support `org/repo`.
- Repo entries before the first `[org]` section are fatal errors.
- If no explicit repositories are supplied, load `<clone-root>/repos`.
- If explicit repositories are supplied, skip the default `repos` file.
- Exact duplicate resolved entries are deduped.
- Different entries with the same local path are fatal errors.
- Warnings use `WARN:`.
- Per-repo command failures use `ERROR:` and continue.
- Fatal validation failures use `FATAL:` and stop before processing.

## Scope Control

Avoid scope creep. In particular, do not add:

- HTTPS clone support.
- Credential-helper rewrites.
- New config formats unless requested.
- Network calls for validation.
- Destructive cleanup.
- Git branch management.
- Automatic creation of the clone root.
- New dependencies.
- `rg` or other tools that are not guaranteed to exist.

Useful changes are still scope creep when they are not requested.

## Coding Standards

Follow:

- `standards/coding/bash.md`
- `PLAN.md`

Important rules:

- Use Bash, not POSIX `sh`.
- Keep macOS Bash 3.2 compatibility.
- Do not assume `rg` is installed.
- Use two-space indentation.
- Use `set -o errexit -o pipefail -o nounset`.
- Keep the `ERR` trap.
- Use arrays for command construction.
- Route executable commands through `runCommand`.
- In dry-run mode, `runCommand` prints instead of executing.
- Trim file and flag inputs.
- Skip blank lines and full-line comments in repo files.
- Include filename and line number for repo file parse errors.
- Prefer small focused helpers.

## Review Expectations

When reviewing changes, prioritize:

1. Behavior regressions against `PLAN.md`.
2. Accidental clone URL changes.
3. Portability issues on macOS and Ubuntu.
4. Unrequested feature additions.
5. Missing validation.
6. Complexity that can be removed.

For code review responses, lead with findings and include file/line references.

## Validation Checklist

Run these before reporting success:

```bash
bash -n updateRepos
./updateRepos --dry-run --org my-org --repo app
./updateRepos --dry-run --repo my-org/app=my-local-app
./updateRepos --dry-run --repos-file ./repos
```

Also validate error paths when relevant:

```bash
./updateRepos --dry-run --clone-root /path/that/does/not/exist
./updateRepos --dry-run --repos-file /path/that/does/not/exist
./updateRepos --dry-run --repo '   '
```

Do not run live clone or pull operations unless the user asks for it.
