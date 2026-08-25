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

## Branch Bases (stacked diffs and pinned branches)

| Key | Type | Description |
|-----|------|-------------|
| `branch.<name>.funcsBase` | string | What local branch `<name>` is maintained against. Read by `git resync` (what to rebase onto), `git stack` (what to show), and `git sweep` (what to re-point children to, and what not to delete). Set by `git pin` on the current branch, by `git new-branch -b <base>` at creation, or by hand with `git config --local branch.<name>.funcsBase <value>`. |

`funcsBase` has four possible answers, and every branch has one:

| Value | Meaning |
|-------|---------|
| *unset* | **The remote primary.** The default, and what nearly every branch should be. `git resync` rebases the branch onto the freshly-fetched primary. |
| a local branch name | **A stack parent.** The branch is a stacked diff on top of another feature branch. `git resync` restacks the whole stack: root onto the primary, each descendant onto its parent. |
| any other ref | **A pinned base.** A tag, a remote-tracking ref (`origin/release/2.4`), or a sha. `git resync` rebases the branch onto that ref instead of the primary, and does not cascade above it. If the ref is remote-tracking on a remote other than the primary one, resync fetches that remote too. |
| `none` | **No base.** The branch is never rebased onto anything. `git resync` reports it and exits 0, so it stays safe to run unconditionally. Use this for a branch that must not pick up trunk. |

The literal string `none` (any case) is the no-base sentinel. A local branch
actually named `none` must be recorded as `refs/heads/none`.

### Pinned branches

Pinning is a repository-configuration decision, not a per-push judgment call.
Once a branch is pinned, the rest of the workflow follows automatically:

- `git resync` prints a notice and leaves the branch alone. It is still
  correct — and still expected — to run it before every push and PR.
- The `git-workflow` plugin's pre-push hook stops requiring a recent fetch on
  that branch, because there is nothing to be fresh against.
- `git sweep` never deletes it, even when the primary contains it.
- Branches stacked *on* a pinned branch still restack onto it normally, and
  inherit its base if it is later swept or skipped over.

```sh
# never pull trunk into this branch again
git pin

# or track a frozen release line instead of trunk
git pin origin/release/2.4

# or create one already pinned
git new-branch -b none vendor/patched-deps

# undo
git pin -u
```

`git pin` is the front end for this key; the equivalent raw form is
`git config --local branch.<name>.funcsBase <value>`.

### Dangling bases

If a `funcsBase` value stops resolving, how it was written decides what
happens. A bare name (`feature/parent`) is treated as a stack parent that was
deleted by hand: `git resync` unsets the stale link with a warning and the
branch falls back to the primary. A qualified value (`refs/...` or
`<remote>/...`) was a deliberate pin, so it is reported but never rewritten,
and the branch is left unrebased — silently reverting a pinned branch to
following trunk is the one thing the pin exists to prevent. `git stack` warns
about both and modifies neither.

Git removes the whole `branch.<name>` section — including `funcsBase` — when
the branch is deleted, so no manual cleanup is needed in the common case.

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
| `funcs.pin.verbose` | git-pin |
| `funcs.sweep.verbose` | git-sweep |

## Sweep Options (`git sweep`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `funcs.sweep.pr` | bool | `true` | Consult GitHub for squash- and rebase-merged branches when the tree comparison says unmerged. A branch is swept on this evidence only if a merged pull request exists for it *and* the local branch carries no commits beyond the one GitHub merged, so work added after the merge is always kept. Costs one batched `gh pr list` call per sweep; degrades silently to tree-comparison-only when `gh` is missing, unauthenticated, or the remote isn't GitHub. Override per-run with `-p` / `-P`. |

## Worktree Options (`git new-tree`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `funcs.worktree.dir` | string | `<repo-parent>/wtree/<repo-name>` | Base directory for worktrees created by `git new-tree`. Overridable per-run with `-d <dir>` or `-p <path>`. |
| `funcs.worktree.copyPath` | string (multi-value) | none | Paths (relative to the main worktree) copied recursively into each new worktree after creation. Set one entry per invocation with `git config --local --add funcs.worktree.copyPath <path>`. Files and directories both work. Missing sources are handled per `funcs.worktree.onMissing` (see below). Existing destinations are never overwritten. Absolute paths and paths containing `..` are rejected. Skip per-run with `git new-tree -P`. These entries are a per-clone overlay on top of `.worktree-populate` (see below). |
| `funcs.worktree.linkPath` | string (multi-value) | none | Like `copyPath`, but creates an absolute symlink pointing back at the source in the main worktree instead of copying. Use for shared caches or large read-mostly artefacts where a single source of truth is wanted. Avoid for files you edit per-worktree (`.env` and friends — use `copyPath` for those). |
| `funcs.worktree.onMissing` | enum: `warn` \| `silent` \| `block` | `warn` | What to do when a `copyPath` / `linkPath` source doesn't exist in the main worktree. `warn` prints to stderr and skips; `silent` skips quietly; `block` prints to stderr, tears down the just-created worktree (removes it, deletes the branch if it was newly created, and rolls back any parent directory `git new-tree` created), and exits non-zero. Applies only to entries sourced from local git config; entries from `.worktree-populate` follow the manifest's own `!onMissing` directive if set, otherwise fall back to this value. |

### `.worktree-populate` (committed manifest)

If a file named `.worktree-populate` exists at the repo root, `git
new-tree` reads it in addition to the `funcs.worktree.copyPath` /
`linkPath` config entries. This is the way to share a worktree file list
with the team — commit the manifest and every clone automatically
populates the same files into new worktrees.

Format: one entry per line, `<mode> <path>`, where `<mode>` is `copy` or
`link` and `<path>` is a repo-root-relative path (files or directories).
Mode and path are separated by any run of whitespace (spaces or tabs);
leading and trailing whitespace on the line is trimmed. Lines starting
with `#` are comments; blank lines are ignored. Comments after a value
on the same line (`copy .env  # local db`) are also stripped — so `#`
cannot appear literally in a path.

`<path>` must be relative and cannot contain `..` segments; absolute
paths and paths that would escape the worktree are rejected with a
warning and skipped (same guard as `funcs.worktree.copyPath`
config entries). Any first token that is not `copy`, `link`, or the
`!onMissing` directive (below) is treated as an unknown directive —
warned to stderr with `.worktree-populate:<lineno>` and skipped, so the
rest of the manifest still applies.

A `!onMissing <value>` directive line sets the missing-source behavior
for entries listed in the manifest, where `<value>` is `warn`, `silent`,
or `block` (same semantics as the `funcs.worktree.onMissing` config
key). It overrides the config default for manifest-sourced entries only;
config-sourced entries continue to follow `funcs.worktree.onMissing`.
Position doesn't matter — the whole manifest uses the resolved value.
If the directive is set more than once with different values, the last
wins and a warning is printed. If absent, the manifest inherits
`funcs.worktree.onMissing` (which itself defaults to `warn`).

```
# Team defaults for every worktree.
!onMissing block
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
