`git-funcs`
===========

`git-funcs` is a collection of shell scripts to make keeping an orderly git
branching strategy easier with a team. These functions aim to serve these
principles:

* There should be only one long-lived (hereafter: _primary_) branch in the
	repository
* Features, bug fixes, etc. should be developed in their own branches and
	should be based on the primary branch
* The project may have multiple remote repositories
* The project's primary remote repository and that repository's primary branch
	could be named anything
* The project's primary remote repository and that repository's primary branch
	can change at any time.

The functions are written to run in any POSIX-compliant shell.

Usage
-----

Commands can be added to `git` by placing executables prefixed 'git-' in your
user's `PATH`, and the function name becomes whatever follows the 'git-'
prefix. Installing these functions is achieved by copying the functions and
the helpers file in this repo to a directory in your path or by cloning or
exporting this repo to a directory that you add to your path.

### Common Options

All commands support these flags, but won't necessarily change behavior from
defaults.

#### option `-h` (help)

All commands support a `-h` flag to print help text containing a description of
the command, its options, and arguments.

#### option `-v` (verbose)

The `-v` flag prints additional information, such as the output from
the used `git` commands and whether provided or default values are used. It
is not meant to be used as input to other commands. Overidden by `-q`.

#### option `-q` (quiet)

The `-q` flag prevents the command from printing to stdout. This is useful
when running a command for side effects (like `refresh`) or to use only the
return status as a conditional. Overrides `-v`.

### Settings

These commands will check your git config files when explicit flags and arguments
for an option is not provided. If the configuration key is not set, boolean
settings are assumed to be off. Config settings can be overridden by command
flags and arguments.

#### funcs.primaryRemote and funcs.primaryBranch (local only)

Commands that refer to the primary remote and/or branch will first check if
a remote was passed as an argument. If not, it will then check the
repository's local config file for values in "funcs.primaryRemote" and
"funcs.primaryBranch", and use those values if it finds them.

If the primaries are not found, the `refresh` command will automatically be
run to try to discover the primary remote and branch and save them to the
config. The values can also be set by editing the config file, or by passing
a remote as argument to the `refresh` command after the `-s` flag to set the
remote you specify as the primary.

#### funcs.verbose, funcs.<function name>.verbose

The verbose option (`-v`) can be set globally (in the global or system
config) or locally (in the repository config), and can be set for all
commands (funcs.verbose) or for a single command (funcs.<command
name>.verbose). Settings for single commands override settings for all
functions. This setting can be set to any value `git` accepts as a valid
boolean value.

Functions
---------

### refresh

`git refresh [-e|-q] [-v] [-s|-n] [<remote>]`

Determine the branch that feature branches should be based on. The -e
and -q options and the -s and -n options will override each other, and
whichever is last takes precedence.

If neither -s nor -n are set, refresh will check the repository's git
configuration for a saved primary remote, and if not found, it will save the
primary remote and branch based on the repository's first listed remote.

* `-s <remote>` (save) Save primary remote given as an argument and its primary
	branch to the local config, or if none is provided, save the rediscovered
	default remote and its primary branch.
* `-n` (noop) Do not set primary remote and branch
* `-e` (echo) Print the names of the primary remote and branch when done
* `-q` (quiet) Print nothing to stdout
* `-v` (verbose) Print additional information about how the remote and
branch are determined

If the repository has no remote at all, `refresh` degrades to a
branch-only primary (a warning is printed to stderr) instead of failing. A
repository with no commits yet (unborn HEAD) still errors, since there's
nothing to base a primary on.

### primary

`git primary [-r|-b] [-q|-v] [<remote>]`

Show the remote and branch that feature branches should be based on. If
`<remote>` is provided, it will show the primary branch for that remote instead
of the primary remote. Otherwise, it will first check the repository's
configuration for the primary remote, and if not found, return the first
remote associated with the repository. To change the primary remote, use
`git-refresh -s <remote>`.

* `-r` (remote) Print primary remote only
* `-q` (quiet) Print nothing to stdout
* `-v` (verbose) Print additional information about how the remote and
	branch are determined

In a repository with no remote, the primary is branch-only: the default
and `-b` forms print just the branch name, and `-r` prints nothing.

### co-primary

`git co-primary [-s] [-q|-v] [<remote>]`

Check out the primary branch of the primary remote repository. If a remote is
provided as an argument, the primary branch of that remote will be checked
out instead. Any uncommitted changes will be stashed before checkout and
re-applied after.

* `-s` (stash) Automatically stash uncommitted changes before checking out the
	branch and pop after
* `-q`, (quiet) Print nothing to stdout. Overrides .gitconfig options
	funcs.verbose and funcs.refresh.verbose.
* `-v` (verbose) Print additional information
* `-h` (help) Print help.

With no remote, checks out the local primary branch without fetching (a
warning is printed to stderr). Passing an explicit `<remote>` in a
remoteless repository is still an error.

### new-branch

`git new-branch [-b <base>] [-q|-v] <branch-name> [<remote>]`

Create a new branch with <branch-name> based on the remote's HEAD. If
<remote> is provided, it will use the primary branch for that remote instead
of the primary remote. Otherwise, it will first check the repository's
configuration for the primary remote, and if not found, use the first remote
associated with the repository. To change the primary remote, use git-refresh
-s <remote>.

With `-b <base>`, the new branch is based on `<base>` instead of the
primary, and `branch.<branch-name>.funcsBase` is set to `<base>` to record
it. `<base>` may be a local branch (a stacked diff), any other ref — tag,
remote-tracking ref, sha — (a pinned base), or `none` (pinned with no base,
so the branch is never rebased; it still starts from the primary). Use `.`
or `@` for `<base>` to mean the current branch. `-b` cannot be combined with
a `<remote>` argument, except `-b none`. See "Branch bases" below.

* `-b <base>` (base) Base the new branch on `<base>` and record it as the
	branch's base.
* `-q` (quiet) Supress writing to stdout. Overrides .gitconfig funcs.verbose and funcs.refresh.verbose.
* `-v` (verbose) Print additional information
* `-h` (help) Print help

### new-tree

`git new-tree [-q|-v] [-P] [-d <base-dir>] [-p <path>] <branch>`

Create a worktree for `<branch>`. If the branch already exists locally, the
worktree attaches to it. Otherwise the branch is created from the refreshed
primary remote's HEAD (as `git new-branch` does). To base on a non-primary
remote, first set it with `git refresh -s <remote>`.

By default the worktree lands at `<base>/<branch>`, where `<base>` is the
first of: the `-p` path (used verbatim), `-d <base-dir>`, the local config
value `funcs.worktree.dir`, or `<repo-parent>/wtree/<repo-name>`. Relative
bases are anchored at the repo toplevel, not the caller's CWD. Missing parent
directories are created. Branch names with `/` (such as
`feature/PROJ-123_title`) segment naturally into subdirectories.

After creating the worktree, entries from two sources are populated into
the new worktree from the main worktree, at the same relative path:

1. A committed `.worktree-populate` file at the repo root (team-wide
	 default). One entry per line, `<mode> <path>`, where `<mode>` is
	 `copy` or `link`. `#` starts a comment; blank lines are ignored.
2. The multi-value local config keys `funcs.worktree.copyPath` and
	 `funcs.worktree.linkPath` (per-clone personal overlay). Add entries
	 with `git config --local --add funcs.worktree.copyPath <path>`.

Both sources are unioned; duplicate destinations trigger a "destination
already exists" warning without overwriting. Files and directories both
work. Missing sources are handled per `funcs.worktree.onMissing` (config
entries) or the manifest's `!onMissing` directive (`.worktree-populate`
entries), each of `warn` (default; stderr + skip), `silent` (skip
quietly), or `block` (stderr + tear down the worktree + exit non-zero).
Absolute paths and paths containing `..` are rejected. Use `copy` for
per-worktree editable state (e.g. `.env`), `link` for shared read-mostly
artefacts. Skip all population for one invocation with `-P`. See
`CONFIG.md` for the manifest format and full config reference.

On success the absolute path of the new worktree is printed to stdout, so it
can be captured by a shell wrapper to `cd` into.

* `-d <base-dir>` Override the worktree base directory for this run. Ignored
	when `-p` is set.
* `-p <path>` Use `<path>` verbatim as the worktree path; skip base-dir
	composition.
* `-P` (no-populate) Skip populating configured `funcs.worktree.copyPath` and
	`funcs.worktree.linkPath` entries into the new worktree.
* `-q` (quiet) Suppress writing to stdout.
* `-v` (verbose) Print additional information.
* `-h` (help) Print this help.

### resync

`git resync [-v|-q] [<remote>]`

Fetch the latest HEAD from this repository's remote and rebase the local HEAD
on it. If <remote> is provided, it will show the primary branch for that
remote instead of the primary remote. Otherwise, it will first check the
repository's configuration for the primary remote, and if not found, return
the first remote associated with the repository. To change the primary
remote, use `git-refresh -s <remote>`."

* `-q` (quiet) Print nothing to stdout. Overrides .gitconfig options
	funcs.verbose and funcs.refresh.verbose.
* `-v` (verbose) Print output from git commands
* `-h` (help) Print help.

With no remote, prints a warning to stderr and exits 0 without attempting
to sync.

What a branch is rebased onto is its recorded base (see "Branch bases"
below): the remote primary by default, a parent branch if it is a stacked
diff, a pinned ref if one is recorded, or nothing at all if it is pinned
with `funcsBase = none` — in which case `resync` prints a notice and exits 0
without touching the branch. That last case is deliberate: `git resync`
stays safe to run unconditionally on every branch, so "should I resync?"
never becomes a judgment call.

If the current branch is part of a stack, `resync` restacks the whole stack
instead of just the current branch: the root is rebased onto whatever it is
maintained against, then every descendant is rebased onto its (possibly
moved) parent, in BFS order. A parent that has already landed in the stack's
target — including via a squash- or rebase-merge — is skipped over rather
than replayed, and the child's `branch.<name>.funcsBase` is re-pointed past
it, inheriting the parent's own base (so a pinned stack stays pinned). On a
rebase conflict partway through the cascade, the conflicting branch is left
mid-rebase (resolve or abort as usual); re-running `git resync` afterwards
finishes the rest of the stack.

### stack

`git stack [-q|-v]`

Read-only. Print the tree of branches the current branch is stacked with, as
recorded by `branch.<name>.funcsBase`. The first line is what the stack is
maintained against — the primary, a pinned ref, or `(no base)`. The tree
follows, root first, one branch per line, indented two spaces per depth.

If the current branch isn't part of a stack, prints
`<branch> (not stacked; base: <remote>/<primary>)` instead — or
`<branch> (pinned; no base)` / `<branch> (not stacked; pinned base: <ref>)`
for a pinned branch.

Markers: ` *` the current branch; ` [merged]` the branch is fully contained
in the stack's target (incl. squash/rebase merges); ` [base missing]` the
branch's recorded base no longer resolves (a bare parent name is repaired
automatically the next time `git resync` runs on this stack; a pinned ref is
left alone — `git stack` itself never modifies anything).

* `-q` (quiet) Print nothing to stdout.
* `-v` (verbose) Print additional information.
* `-h` (help) Print this help.

### up

`git up [-q|-v] [<remote/branch>]`

Fetch the primary remote and rebase the local HEAD on it. If no
arguments are provided, it will use the saved primary and branch, but a remote
or remote and branch (or other valid ref) to pull from can be provided as
arguments to be passed to `git pull`. Any submodules will then be updated.

* `-q` '(quiet) Suppress output from git commands. Overrides .gitconfig
	options funcs.verbose and funcs.refresh.verbose.'
* `-v` '(verbose) Print output from git commands'
* `-h` '(help) Print this help'

With no remote, prints a warning to stderr, skips the pull, and still
updates submodules.

### label

`git label [<ref>]`

Generate a label that can be used as a build tag or in an application to
indicate which version of code the build is based on. The label is the branch
or tag name and the shortened commit hash, separated by a colon. This
combination identifies the ref needed to build the project again, and whether
a deployed build came from the expected branch (e.g. Is production running
a release? Is QA running a build of the main branch, or a deployed feature
branch? Is the build of the main branch in Staging the latest commit?)

The label is generated for the current HEAD, unless another ref is given
as the first argument.

* `-h` Print this help

### contains

`git contains [-v|-q] [<target>] [<branch>]`

Check whether all of `<branch>`'s changes are already present in `<target>`.
Returns success when a merge of `<branch>` into `<target>` would be a no-op.
Unlike `git merge-base --is-ancestor`, this also recognizes squash-merges and
rebase-merges by simulating the merge with `git merge-tree --write-tree` and
comparing the resulting tree to `<target>`'s tree.

If `<target>` is omitted, the primary branch is used. If `<branch>` is omitted,
the current HEAD is used. With no arguments at all, answers "did my current
branch land in primary?"

Exit status is 0 when `<target>` contains `<branch>`, 1 when it does not
(including when a merge would conflict). If the `-v` (verbose) flag is set,
it also prints the result.

* `-q` (quiet) Write nothing to stdout and use the exit status instead.
* `-v` (verbose) Print additional information (default).
* `-h` (help) Print this help.

#### Deprecated: `merged-into`

`git merged-into [<ancestor>] [<descendant>]` is kept as a thin
backwards-compat shim that forwards to `git contains` with the arguments
flipped and prints a deprecation warning. Prefer `git contains` in any
new invocation.

### sweep

`git sweep [-n] [-w] [-f] [-p|-P] [-t <target>] [-q|-v]`

Find and delete local branches whose changes are fully present in the
target branch. Unlike `git branch --merged`, this detects rebase-merges
and squash-merges by simulating a merge and comparing the resulting tree.

That tree comparison only holds while the target still merges the branch
cleanly. Once the target advances past a squash-merged branch the simulation
conflicts, and the branch reads as unmerged forever even though every line of
it shipped. So when the comparison fails, GitHub is asked whether a merged
pull request exists for the branch; it is swept only if it carries no commits
beyond the commit that was merged, so anything pushed after the merge keeps
the branch alive. Disable with `-P` or `funcs.sweep.pr = false`. It costs one
batched `gh` call per sweep and no-ops when `gh` is unavailable or the remote
isn't GitHub.

For each deleted branch, prints the branch name and head commit to stdout.
Branches checked out in worktrees are skipped unless `-w` is set.

Worktree removal refuses to discard uncommitted or untracked files, so a
single stray `.idea/` strands an otherwise-sweepable worktree; the warning
names how many local changes are in the way, and `-f` overrides. With `-w`,
worktrees with a detached `HEAD` are swept too — holding no branch ref, they
are invisible to the branch walk and otherwise accumulate indefinitely. Only
the worktree goes; the commit stays reachable via the reflog. The main
worktree is never touched.

If a deleted branch is the stack parent (`branch.<name>.funcsBase`) of other
local branches, each child is re-pointed to the deleted branch's own base
(or un-stacked, if it had none), and a re-point notice is printed to stderr
(respect `-q`) — or, in dry-run mode, what would be re-pointed.

Branches with a pinned base (`funcsBase` set to `none` or to a ref rather
than a local branch) are never swept: they are deliberately maintained off
the primary, so the target containing them does not mean they have landed.

* `-n` (dry-run) Show what would be deleted without deleting.
* `-w` (worktrees) Also remove worktrees for merged branches, and sweep
	merged detached-`HEAD` worktrees.
* `-f` (force) Remove worktrees even when they hold uncommitted or untracked
	files. Implies `-w`.
* `-p` (pull requests) Consult GitHub for squash-merged branches (default
	when `gh` is available).
* `-P` (no pull requests) Compare trees only; never call GitHub.
* `-t <target>` Branch or ref to compare against (default: primary
	remote/branch).
* `-q` (quiet) Suppress informational output.
* `-v` (verbose) Print additional information.

### deep-commit-all

`git deep-commit-all [-q] [-n] [-p] [-m] <commit-message>`

Commit all modified files in this repo and all submodules. Provide a commit
message, and all modified files in all submodules will be committed with
that message. All modified files in this repo will then be committed with
the updated submodules using the same message.

* `-n` (dry-run) Print what results would be, but do not commit.
* `-p` (push) Push changes after committing.
* `-m` Provide commit message as an option instead of a positional parameter.
* `-q` (quiet) Silence output.

## Branch bases

Every branch is maintained against something. That something is recorded in
`branch.<name>.funcsBase` in local git config, and it is the single question
`resync`, `stack`, and `sweep` all consult:

| `funcsBase` | The branch is maintained against | `git resync` |
| --- | --- | --- |
| *unset* | the remote primary | rebases onto freshly-fetched trunk |
| a local branch | that branch — a stacked diff | restacks the whole stack |
| any other ref | a pinned base (tag, `origin/release/2.4`, sha) | rebases onto that ref |
| `none` | nothing — pinned | reports it, changes nothing, exits 0 |

The default (unset) is right for nearly every branch, and you never set it by
hand. The three sections below are the non-default cases.

Whatever the base, **`git resync` remains the one command to run** when you
start work, before every push, and before opening or updating a PR. It reads
the base and does the right thing, including doing nothing. There is never a
reason to decide branch-by-branch whether to skip it.

### Stack a branch on another branch (stacked diffs)

For work that builds on a feature branch that hasn't landed yet, so each
piece can be reviewed as its own PR.

```sh
git new-branch feature/api            # parent: off freshly-fetched trunk
# ...commit, push, open PR...

git new-branch -b . feature/ui        # child: stacked on the current branch
# `-b .` (or `-b @`) means "the branch I'm on"; `-b feature/api` is the same
# thing spelled out. Never `git checkout -b` here — that loses the link.
```

From then on:

```sh
git stack      # show the tree and what it's based on (read-only)
git resync     # restacks the WHOLE stack, from any branch in it
```

`resync` rebases the root onto fresh trunk, then each descendant onto its
(now moved) parent, in order. Run it from anywhere in the stack; it is still
the single command before every push and PR.

Open the child's PR against its parent, not trunk:

```sh
gh pr create --base feature/api --draft
```

When the parent's PR merges, `resync` notices — including squash- and
rebase-merges, where the parent's commits are in trunk under different
SHAs — skips past it instead of replaying its commits, and re-points the
child's base to whatever the parent was based on. Then retarget the PR:

```sh
git resync
gh pr edit <number> --base main
```

If a rebase conflicts partway through, that branch is left mid-rebase:
resolve and `git rebase --continue` (or `--abort`), then re-run `git resync`
to finish the rest of the stack.

### Track a release line instead of trunk (pinned base)

For a hotfix or backport that must sit on a frozen line, not trunk.

```sh
# at creation — starts from that ref and records it
git new-branch -b origin/release/2.4 hotfix/2.4.1

# or for a branch that already exists
git config --local branch.hotfix/2.4.1.funcsBase origin/release/2.4
```

`git resync` now rebases onto `origin/release/2.4` instead of trunk, and
fetches that ref's remote first — including when it lives on a remote other
than the primary one. It does not cascade above the pin: trunk is simply not
part of this branch's story. Branches stacked on it restack onto it as
usual.

The base can be any commit-ish — a tag (`v2.4.0`), a remote-tracking ref, or
a raw SHA. Prefer a fully-qualified ref (`refs/tags/v2.4.0`) for anything
that might later be deleted; see "When a base stops resolving".

Open its PR against the same line:

```sh
gh pr create --base release/2.4 --draft
```

### Never follow anything (pinning)

For a branch that must not pick up trunk at all — a long-lived release
branch, a vendored fork, anything deliberately frozen.

```sh
# pin an existing branch
git config --local branch.release/2.4.funcsBase none

# or create one already pinned (it still starts from fresh trunk)
git new-branch -b none vendor/patched-deps
```

```
$ git resync
notice: 'release/2.4' has no base (funcsBase = none); not rebasing.
$ echo $?
0
```

Pinning is a decision you record once, on the repository — not a judgment
call you re-make at each push. Everything else then follows on its own:

- `git resync` reports and stops. Keep running it; it is still correct.
- The `git-workflow` Claude Code plugin's pre-push hook, if you use it,
  stops demanding a recent fetch on that branch — there is nothing to be
  fresh against.
- `git sweep` never deletes it, even once trunk contains it.
- Branches stacked *on* it still restack onto it normally, and inherit the
  pin if it is later swept or skipped over — a pinned stack stays pinned.

`none` (any case) is the sentinel for "no base". A local branch actually
named `none` must be written `refs/heads/none`.

### Inspecting and undoing

```sh
git stack                                        # what is this branch based on?
git config --get branch.<name>.funcsBase         # the raw value
git config --local --unset branch.<name>.funcsBase   # back to following trunk
git config --local branch.<name>.funcsBase <new-base>  # re-point it
```

Git deletes the whole `branch.<name>` section — `funcsBase` included — when
the branch is deleted, so there is nothing to clean up in the normal case.

### When a base stops resolving

How the value was written decides what happens, because the two cases want
opposite repairs:

- **A bare name** (`feature/api`) is read as a stack parent someone deleted
  by hand. `git resync` unsets the stale link with a warning and the branch
  goes back to following trunk — which is what you want when the parent
  landed and was cleaned up.
- **A qualified value** (`refs/...` or `<remote>/...`) is read as a
  deliberate pin that vanished. It is reported but **never** rewritten, and
  the branch is left unrebased. Silently reverting a pinned branch to
  following trunk is the one thing the pin exists to prevent — fix or unset
  it yourself.

`git stack` warns about both and modifies neither.

See [CONFIG.md](CONFIG.md) for the config-key reference.
