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
| `funcs.merged-into.verbose` | git-merged-into |
| `funcs.new-tree.verbose` | git-new-tree |

## Worktree Options (`git new-tree`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `funcs.worktree.dir` | string | *(none)* | Base directory for worktrees. Required unless an absolute path or `-d` is provided. |

### Example

```ini
[funcs]
    primaryRemote = origin
    primaryBranch = main
    verbose = false

[funcs "worktree"]
    dir = /home/user/worktrees/myproject

[funcs "refresh"]
    echo = true
    verbose = true
```

### Setting options

```sh
git config --local funcs.worktree.dir ~/worktrees/myproject
```
