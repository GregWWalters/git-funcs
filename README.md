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

With `-b <base>`, the new branch is based on the local branch `<base>`
instead of the primary, and `branch.<branch-name>.funcsBase` is set to
`<base>` to record the stacking relationship (this is how you start a
stacked diff). Use `.` or `@` for `<base>` to mean the current branch.
`-b` cannot be combined with a `<remote>` argument. See `git stack` and the
"Stacked diffs" section below.

* `-b <base>` (base) Base the new branch on local branch `<base>` and record
	it as the stack parent.
* `-q` (quiet) Supress writing to stdout. Overrides .gitconfig funcs.verbose and funcs.refresh.verbose.
* `-v` (verbose) Print additional information
* `-h` (help) Print help

### new-tree

`git new-tree [-q|-v] [-d <base-dir>] [-p <path>] <branch>`

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

On success the absolute path of the new worktree is printed to stdout, so it
can be captured by a shell wrapper to `cd` into.

* `-d <base-dir>` Override the worktree base directory for this run. Ignored
	when `-p` is set.
* `-p <path>` Use `<path>` verbatim as the worktree path; skip base-dir
	composition.
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

If the current branch is part of a stack (see "Stacked diffs" below),
`resync` restacks the whole stack instead of just the current branch: the
root is rebased onto the fetched primary, then every descendant is rebased
onto its (possibly moved) parent, in BFS order. A parent that has already
landed in primary — including via a squash- or rebase-merge — is skipped
over rather than replayed, and the child's `branch.<name>.funcsBase` is
re-pointed past it. On a rebase conflict partway through the cascade, the
conflicting branch is left mid-rebase (resolve or abort as usual); re-running
`git resync` afterwards finishes the rest of the stack.

### stack

`git stack [-q|-v]`

Read-only. Print the tree of branches the current branch is stacked with, as
recorded by `branch.<name>.funcsBase`. Root first, one branch per line,
indented two spaces per depth beneath the implicit primary base.

If the current branch isn't part of a stack, prints
`<branch> (not stacked; base: <remote>/<primary>)` instead.

Markers: ` *` the current branch; ` [merged]` the branch is fully contained
in primary (incl. squash/rebase merges); ` [base missing]` the branch's
recorded parent no longer exists (repaired automatically the next time
`git resync` runs on this stack — `git stack` itself never modifies
anything).

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

`git sweep [-n] [-w] [-t <target>] [-q|-v]`

Find and delete local branches whose changes are fully present in the
target branch. Unlike `git branch --merged`, this detects rebase-merges
and squash-merges by simulating a merge and comparing the resulting tree.

For each deleted branch, prints the branch name and head commit to stdout.
Branches checked out in worktrees are skipped unless `-w` is set.

If a deleted branch is the stack parent (`branch.<name>.funcsBase`) of other
local branches, each child is re-pointed to the deleted branch's own parent
(or un-stacked, if it had none), and a re-point notice is printed to stderr
(respect `-q`) — or, in dry-run mode, what would be re-pointed.

* `-n` (dry-run) Show what would be deleted without deleting.
* `-w` (worktrees) Also remove worktrees for merged branches.
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

