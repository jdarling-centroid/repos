# updateRepos New Project Plan

## Goal

Build `updateRepos`, a portable Bash tool that clones or updates GitHub
repositories from one or more organizations using SSH.

The project is a new implementation. Do not preserve earlier script structure
unless it still fits this plan.

## Runtime Requirements

- Support macOS and Ubuntu.
- Support macOS Bash 3.2.
- Use Bash with `#!/bin/bash`.
- Use Git for clone and pull operations.
- Treat GitHub CLI (`gh`) as optional unless `--github-user` is supplied.
- Use SSH clone URLs only.
- Do not assume `rg` is installed.

## Repository Layout

Target project structure:

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

## Core Behavior

`updateRepos` reads repository definitions, resolves each to a GitHub remote and
local destination, then processes every repository.

For each resolved repository:

- If the local Git repository exists, run `git pull --ff-only`.
- If the local path exists but is not a Git repository, skip it with `WARN:`.
- If the local path does not exist, clone it by SSH.
- Failed `git pull`, `git clone`, or parent-directory creation is non-fatal for
  the run. Print `ERROR:` to stderr, increment the failed count, and continue.
- If any repository fails, print the summary and exit `1`.
- If no repositories fail, exit `0`.

## Data Model

Each repository resolves to:

```text
org
repo
localPath
```

Internal representation may use a tab-delimited string:

```text
org<TAB>repo<TAB>localPath
```

Examples:

```text
core-enterprise<TAB>coalition-app<TAB>coalition-app
core-enterprise<TAB>coalition-app<TAB>c-app
another-org<TAB>platform-tools<TAB>tools/platform
```

## Clone Root

`--clone-root <path>` controls where repositories are cloned or updated.

Rules:

- If omitted, clone root defaults to the current working directory.
- It may appear anywhere on the command line.
- It may only be supplied once.
- It controls local paths only.
- It does not override explicit `--org` values.
- The default CLI org is the basename of the final clone root.

Local checkout path:

```text
<clone-root>/<localPath>
```

Do not clone into:

```text
<clone-root>/<org>/<repo>
```

## Command-Line Interface

Supported options:

```text
--clone-root <path>
--org <name>
--repo <value>
--repos-file <path>
--github-user <name>
--dry-run
-h, --help
```

### `--org`

Sets the current CLI org for later `--repo` values.

Rules:

- May appear multiple times.
- A later `--org` applies to later CLI `--repo` values.
- Does not affect repo file parsing.
- Repo file org sections do not affect CLI org.

### `--repo`

May be supplied multiple times.

Supported forms:

```text
repo
repo=localPath
org/repo
org/repo=localPath
```

Interpretation:

```text
repo                 -> currentCliOrg/repo -> <clone-root>/repo
repo=localPath       -> currentCliOrg/repo -> <clone-root>/localPath
org/repo             -> org/repo           -> <clone-root>/repo
org/repo=localPath   -> org/repo           -> <clone-root>/localPath
```

Examples:

```bash
./updateRepos --repo myrepo
./updateRepos --repo myrepo=otherpath
./updateRepos --repo myorg/myrepo
./updateRepos --repo myorg/myrepo=myotherpath
```

### `--repos-file`

May be supplied multiple times.

Rules:

- Each file is parsed independently.
- Files do not inherit CLI org.
- Files must define org sections with `[org]`.
- File org sections do not leak into CLI parsing or other files.
- Supplying any `--repo` or `--repos-file` skips the default file.

### `--github-user`

Runs GitHub CLI account switching before repository processing.

Rules:

- If omitted, do nothing.
- If supplied, require `gh`.
- Missing `gh` is fatal.
- Resolve `gh` by first checking for an executable `gh` next to `updateRepos`,
  then falling back to `gh` from `PATH`.
- Run:

  ```bash
  gh auth switch --user <name>
  ```

- Do not run `gh auth setup-git`.
- Do not change clone URLs.
- Do not use HTTPS.

### `--dry-run`

Dry-run uses the same control flow as live mode. The only difference is that
`runCommand` prints each command instead of executing it.

## Repository File Format

Repository files use INI-like org sections:

```text
# Default repositories for updateRepos.
[core-enterprise]

coalition-app=c-app
aws-data-lake
shared-sftp
```

Supported lines:

```text
# comment

[org-name]
repo
repo=localPath
```

Rules:

- Blank lines are ignored.
- Lines where the trimmed line starts with `#` are ignored.
- `[org-name]` starts an org section.
- Whitespace inside section brackets is allowed and trimmed.
- `repo` uses local path `repo`.
- `repo=localPath` uses local path `localPath`.
- Whitespace around `=` is allowed and trimmed.
- Repo files do not support `org/repo`.
- Repo files do not support `org/repo=localPath`.
- Repo files do not support `org=<name>`.
- Repo entries before the first `[org-name]` section are fatal errors.
- Empty sections, such as `[]` or `[   ]`, are fatal errors.
- Empty repo names are fatal errors.
- Empty local paths are fatal errors.
- Inline comments are not supported.
- Repo file parse errors must include filename and line number.

Example:

```text
[core-enterprise]
coalition-app = c-app
aws-data-lake

[ another-org ]
platform-tools = tools/platform
```

Resolved entries:

```text
core-enterprise<TAB>coalition-app<TAB>c-app
core-enterprise<TAB>aws-data-lake<TAB>aws-data-lake
another-org<TAB>platform-tools<TAB>tools/platform
```

## Default Repository File

If no explicit `--repo` or `--repos-file` is supplied, load:

```text
<clone-root>/repos
```

Rules:

- If the default file does not exist, fail with `FATAL:`.
- If the default file produces no repository entries, fail with `FATAL:`.
- If the default file has invalid syntax, fail with `FATAL:`.

If any explicit `--repo` or `--repos-file` is supplied, do not load the default
file.

## Local Path Validation

Local paths may include relative subdirectories.

Reject local paths that:

- Are empty.
- Are absolute paths.
- Contain `..` path segments.
- End in `/`.
- Contain backslashes.

Examples:

```text
app=app-local          # allowed
app=clients/app-local  # allowed
app=/tmp/app           # rejected
app=../app             # rejected
app=clients/../app     # rejected
app=clients/app/       # rejected
app=clients\app        # rejected
```

## Org And Repo Validation

Keep org and repo validation minimal.

Rules:

- Reject empty org values.
- Reject empty repo values.
- Reject `/` in repo-file repo names because repo files do not support
  `org/repo`.
- Do not add broad GitHub character validation.
- Let Git report remote names outside the parser's responsibility.

## Sorting, Deduplication, And Collisions

Before repository processing:

1. Sort resolved entries.
2. Deduplicate exact resolved entries silently.
3. Validate that no two different entries map to the same local path.

Exact duplicates are allowed:

```text
core-enterprise<TAB>app<TAB>app
core-enterprise<TAB>app<TAB>app
```

Local path collisions are fatal:

```text
org-a<TAB>project1<TAB>shared
org-b<TAB>project2<TAB>shared
```

This is also fatal:

```text
org-a<TAB>project1<TAB>project1
org-b<TAB>project1<TAB>project1
```

Collision errors must identify the conflicting local path and entries.

## Clone And Update Commands

For each resolved entry:

```text
org<TAB>repo<TAB>localPath
```

Clone URL:

```text
git@github.com:<org>/<repo>.git
```

Existing Git repo:

```bash
git -C <clone-root>/<localPath> pull --ff-only
```

Clone missing repo:

```bash
git -C <clone-root> clone git@github.com:<org>/<repo>.git <localPath>
```

Do not validate existing repo origins. If a user changes `origin`, that is
user-owned state.

If `<localPath>` contains subdirectories and the target repo path does not
exist, create parent directories before cloning:

```bash
mkdir -p <clone-root>/<parent>
```

Failed `mkdir -p`, `git clone`, and `git pull` are per-repo failures:

- Print `ERROR:` to stderr.
- Increment failed count.
- Continue to next repository.

## Output Rules

- Warnings start with `WARN:`.
- Per-repo command failures start with `ERROR:` and go to stderr.
- Fatal errors start with `FATAL:` and go to stderr.
- Help text may remain on stdout.
- Fatal validation errors stop before repository processing.
- Per-repo failures continue processing and produce exit code `1` after summary.

Summary should include:

```text
updated:
cloned:
skipped:
failed:
```

## Implementation Steps

1. Build the project as a clean implementation from this plan.
2. Implement `updateRepos` with strict Bash mode and macOS Bash 3.2 compatibility.
3. Parse global flags and ordered repo-source events.
4. Enforce single `--clone-root`.
5. Resolve CLI repo entries.
6. Parse repo files with `[org]` sections and `repo=localPath` support.
7. Load `<clone-root>/repos` only when no explicit repo sources exist.
8. Sort and dedupe exact entries.
9. Validate local path collisions.
10. Run optional `gh auth switch --user`.
11. Process each repo with non-fatal per-repo failures.
12. Update `README.md`, `AGENTS.md`, `standards/`, and `repos`.
13. Run validation.

## Validation Plan

Syntax:

```bash
bash -n updateRepos
```

Default file:

```bash
./updateRepos --dry-run
```

Command-line repo forms:

```bash
./updateRepos --dry-run --repo myrepo
./updateRepos --dry-run --repo myrepo=otherpath
./updateRepos --dry-run --repo myorg/myrepo
./updateRepos --dry-run --repo myorg/myrepo=myotherpath
```

Command-line org switching:

```bash
./updateRepos --dry-run --org org-a --repo app --org org-b --repo tools
```

Repository file org sections:

```bash
./updateRepos --dry-run --repos-file ./repos
```

File org scope does not leak:

```bash
./updateRepos --dry-run --repos-file ./test-repos --org org-a --repo final
```

Dedupe:

```bash
./updateRepos --dry-run --org org-a --repo app --repo app
```

Duplicate local path collision:

```bash
./updateRepos --dry-run --org org-a --repo app --org org-b --repo app
```

Expected: fatal error before processing.

Repository file destination aliases:

```text
[core-enterprise]
coalition-app=c-app
aws-data-lake
```

Expected dry-run:

```text
git -C <clone-root> clone git@github.com:core-enterprise/coalition-app.git c-app
git -C <clone-root> clone git@github.com:core-enterprise/aws-data-lake.git aws-data-lake
```

Repository file subdirectory aliases:

```text
[core-enterprise]
coalition-app=clients/c-app
```

Expected dry-run:

```text
mkdir -p <clone-root>/clients
git -C <clone-root> clone git@github.com:core-enterprise/coalition-app.git clients/c-app
```

Repository file duplicate local path:

```text
[org-a]
project1=shared

[org-b]
project2=shared
```

Expected: fatal error before processing.

Error paths:

```bash
./updateRepos --dry-run --clone-root /path/that/does/not/exist
./updateRepos --dry-run --clone-root /tmp/a --clone-root /tmp/b
./updateRepos --dry-run --repos-file /path/that/does/not/exist
./updateRepos --dry-run --repo '   '
./updateRepos --dry-run --github-user someone
```

Per-repo command failure behavior should be validated with controlled failing
commands or inaccessible repos when appropriate.

Use portable shell tools for validation and inspection. Do not rely on `rg`.
