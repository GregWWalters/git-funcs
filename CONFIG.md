# Configuration Options

All options are set in the repository's `.git/config` under the `[funcs]`
section. Use `git config --local` to set them.

## Global Options

| Key | Type | Description |
|-----|------|-------------|
| `funcs.primaryRemote` | string | The primary remote (set automatically by `git refresh`) |
| `funcs.primaryBranch` | string | The primary branch (set automatically by `git refresh`) |
| `funcs.verbose` | bool | Enable verbose output for all commands |
| `funcs.echo` | bool | Echo the primary branch after operations |

## Stacked-diff Options

| Key | Type | Description |
|-----|------|-------------|
| `branch.<name>.funcsBase` | string | The stack parent of local branch `<name>` — another local branch this one is based on, instead of the primary. Set automatically by `git new-branch -b <base>`; consumed by `git resync` (restacks the whole stack), `git stack` (shows it), and `git sweep` (re-points children when a parent is deleted). A branch with no `funcsBase` is implicitly based on the primary. Git automatically removes the whole `branch.<name>` section — including this key — when the branch is deleted, so no manual cleanup is needed in the common case. If the named parent branch is deleted by hand (bypassing `git sweep`), the link goes dangling; `git resync` repairs it (unsets it, with a warning) the next time it walks the stack, and `git stack` warns without modifying it. |

## Per-Command Verbose

Each command checks its own verbose key before falling back to
`funcs.verbose`. The per-command key overrides the global one.

| Key | Commands affected |
|-----|-------------------|
| `funcs.refresh.verbose` | git-refresh |
| `funcs.refresh.echo` | git-refresh |
| `funcs.primary.verbose` | git-primary |
| `funcs.co-primary.verbose` | git-co-primary, git-new-branch, git-resync |
| `funcs.up.verbose` | git-up |
| `funcs.contains.verbose` | git-contains |
| `funcs.new-tree.verbose` | git-new-tree |
| `funcs.stack.verbose` | git-stack |

## Worktree Options (`git new-tree`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `funcs.worktree.dir` | string | `<repo-parent>/wtree/<repo-name>` | Base directory for worktrees created by `git new-tree`. Overridable per-run with `-d <dir>` or `-p <path>`. |
| `funcs.worktree.copyPath` | string (multi-value) | none | Paths (relative to the main worktree) copied recursively into each new worktree after creation. Set one entry per invocation with `git config --local --add funcs.worktree.copyPath <path>`. Files and directories both work. Missing sources warn to stderr and are skipped; existing destinations are never overwritten. Absolute paths and paths containing `..` are rejected. Skip per-run with `git new-tree -P`. These entries are a per-clone overlay on top of `.worktree-populate` (see below). |
| `funcs.worktree.linkPath` | string (multi-value) | none | Like `copyPath`, but creates an absolute symlink pointing back at the source in the main worktree instead of copying. Use for shared caches or large read-mostly artefacts where a single source of truth is wanted. Avoid for files you edit per-worktree (`.env` and friends — use `copyPath` for those). |

### `.worktree-populate` (committed manifest)

If a file named `.worktree-populate` exists at the repo root, `git
new-tree` reads it in addition to the `funcs.worktree.copyPath` /
`linkPath` config entries. This is the way to share a worktree file list
with the team — commit the manifest and every clone automatically
populates the same files into new worktrees.

Format: one entry per line, `<mode> <path>`, where `<mode>` is `copy` or
`link` and `<path>` is a repo-root-relative path (files or directories).
Lines starting with `#` are comments; blank lines are ignored. Comments
after a value on the same line (`copy .env  # local db`) are also
stripped.

```
# Team defaults for every worktree.
copy .env
copy .env.local
copy .vscode
link shared/big-model.bin
```

The manifest and config entries are unioned; both are safe to use in the
same repo. Duplicate destinations trigger the existing "already exists"
warning — no clobbering.

### Example

```ini
[funcs]
    primaryRemote = origin
    primaryBranch = main
    verbose = false

[funcs "worktree"]
    dir = ../wtree/myproject
    copyPath = .env
    copyPath = .env.local
    linkPath = shared/big-model.bin

[funcs "refresh"]
    echo = true
    verbose = true
```

### Setting options

```sh
git config --local funcs.worktree.dir ~/worktrees/myproject
```
