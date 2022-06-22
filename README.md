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

The `-v` flag turns prints additional information, such as the output from
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

### primary

`git primary [-r|-b] [-q|-v] [<remote>]`

Show the remote and branch that feature branches should be based on. If
`<remote>` is provided, it will show the primary branch for that remote instead
of the primary remote. Otherwise, it will first check the repository's
configuration for the primary remote, and if not found, return the first
remote assiciated with the repository. To change the primary remote, use
`git-refesh -s <remote>`.

* `-r` (remote) Print primary remote only
* `-q` (quiet) Print nothing to stdout
* `-v` (verbose) Print additional information about how the remote and
	branch are determined

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
* `-v` (verbse) Print additional information
* `-h` (help) Print help.

### new-branch

`git new-branch [-q|-v] <branch-name> [<remote>]`

Create a new branch with <branch-name> based on the remote's HEAD. If
<remote> is provided, it will use the primary branch for that remote instead
of the primary remote. Otherwise, it will first check the repository's
configuration for the primary remote, and if not found, use the first remote
assiciated with the repository. To change the primary remote, use git-refesh
-s <remote>.

* `-q` (quiet) Supress writing to stdout. Overrides .gitconfig funcs.verbose and funcs.refresh.verbose.
* `-v` (verbose) Print additional information
* `-h` (help) Print help

### resync

`git resync [-v|-q] [<remote>]`

Fetch the latest HEAD from this repository's remote and rebase the local HEAD
on it. If <remote> is provided, it will show the primary branch for that
remote instead of the primary remote. Otherwise, it will first check the
repository's configuration for the primary remote, and if not found, return
the first remote assiciated with the repository. To change the primary
remote, use `git-refesh -s <remote>`."

* `-q` (quiet) Print nothing to stdout. Overrides .gitconfig options
	funcs.verbose and funcs.refresh.verbose.
* `-v` (verbse) Print output from git commands
* `-h` (help) Print help.

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

### merged-into

`git merged-into [-v|-q] <ancestor> <descendent>`

Check if the first branch <ancestor> has been merged into the second branch
<descendent>.  This is accomplished with `git merge-base --is-ancestor`, which
effectively looks for the head of <ancestor> in the commit history of
<descendent>.

As `merge-base` normally does, the exit status will be 0 if the first branch
has been merged into the second or 1 if not. If the `-v` (verbose) flag is set,
it will also print out the result.'

* `-q`, (quiet) Write nothing to stdout and use the exit status instead,
	returning 0 if the first branch is merged into the second or 1 if not.
* `-v` (verbose) Print additional information (default)
* `-h` (help) Print this help.

