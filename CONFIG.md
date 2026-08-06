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

### Example

```ini
[funcs]
    primaryRemote = origin
    primaryBranch = main
    verbose = false

[funcs "worktree"]
    dir = ../wtree/myproject

[funcs "refresh"]
    echo = true
    verbose = true
```

### Setting options

```sh
git config --local funcs.worktree.dir ~/worktrees/myproject
```
