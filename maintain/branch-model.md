# Branch model

- **Load when:** choosing the branch a patch or change goes to, working out
  the current release cycle or rc, naming a devel or fixes branch, or
  checking maintained trees for fixes not yet sent to mainline.
- **Parameters:** `$TREES`, `$TREE_STATUS`, `$TREE_STATE`
- **Depends on:** [core/safe-operations](../core/safe-operations.md)

## Always-on rules
- Take the release cycle from $TREE_STATE and branch names from the tree's
  own clone in $TREES, never from memory or from a same-named branch in a
  scratch tree or worktree, because the rc changes every week and a stale
  copy of a branch hides work that already landed.

## Procedure
### Finding the active branch
1. Read $TREE_STATE. If it is missing, or its `last_checked_at` is more
   than 24 hours old, run $TREE_STATUS to refresh it. With `--no-net`, the
   tool reuses the cached release data instead of asking kernel.org.
2. Report any anomaly the state shows before going on
   ([anomalies](#anomalies)).
3. Take `kernel.devel_cycle` as X.Y.
   - While `kernel.rc` is 1 or more, fixes go to for-X.Y-fixes and new
     development goes to for-X.(Y+1).
   - When `kernel.rc` is 0, X.Y is released and the merge window for the
     next version is open. for-X.Y-fixes closed at the release. Fixes go to
     the next version's fixes branch, for-X.(Y+1)-fixes, which opens from
     the tip of for-X.(Y+1) once its merge-window pull request is sent.
   - For a tree on another preset, use the branches its $TREES entry names
     instead ([presets](#presets)).
4. Confirm the branch in the tree's own clone with `git -C <tree> log
   --oneline -3 <branch>`. `trees.<name>.fixes_branch` in the state names
   the tree's active fixes branch.
5. Before basing work on a branch, compare its tip in the tree's own clone
   with the base the working tree claims. Search the tree's own clone for a
   mechanism before implementing it again.
6. If the state disagrees with the tree, the tree wins. Refresh the state
   and find out why the two differ.

### Checking for pending fixes
1. Update each tree's master from mainline. With a local mainline clone,
   pull it, then run `git -C <tree> fetch <mainline clone> master:master`.
2. List the fixes mainline doesn't have yet: `git -C <tree> log --oneline
   master..for-X.Y-fixes`.
3. Repeat for every maintained tree in $TREES that has a fixes branch for
   the cycle. Each tree's commands run in their own shell call
   ([core/safe-operations](../core/safe-operations.md#before-a-shell-command-that-targets-a-repository)).

## Reference
### Branch lifecycle
The default preset names branches after the kernel version they are for.
- for-X.Y collects development for the X.Y merge window. Mainline merges it
  during the merge window, and the branch closes.
- for-X.Y-fixes opens when the X.Y merge-window pull request is sent and
  collects fixes until X.Y is released.
- During the X.Y rc cycle, for-X.(Y+1) collects development for the next
  merge window.
- for-X.Y and for-X.Y-fixes are never open at the same time.
- for-next merges the open branches for linux-next. It is an integration
  branch that never enters mainline.

### Presets
A tree's $TREES entry names its preset. Anything else that differs per tree
goes in the entry too.
- `for-X.Y`: the lifecycle above. Pull request tags start with the tree's
  tag prefix, and $TREE_STATUS assumes this preset.
- `patch-queue`: patches move through a chain of staging branches before a
  pull request, as in the mm tree. There a patch enters mm-new, moves to
  mm-unstable and then to mm-stable
  (Documentation/mm/damon/maintainer-profile.rst). Patches outside the
  subsystem that the maintainer also carries travel on a parallel pair,
  mm-nonmm-unstable and mm-nonmm-stable, and fixes for the current rc on
  hotfix branches such as mm-hotfixes-stable. Each pull request comes from a
  date-stamped tag of a stable branch, named `<branch>-YYYY-MM-DD-HH-MM`.
  The entry names the branch new patches enter, the branch fixes enter and
  the branches pull requests come from. $TREE_STATUS doesn't cover this
  preset.

### The state file
$TREE_STATUS writes $TREE_STATE, and nothing else announces its contents:
anomalies reach the user only when an agent reads the state.
- `kernel.devel_cycle` is the mainline version X.Y in development or just
  released. `kernel.rc` is its rc number, or 0 once X.Y is released and the
  next merge window is open. `kernel.released` and `kernel.latest_stable`
  go with them, and `kernel.fetched_at` is when they came from kernel.org.
- `last_checked_at` is the time of the last full refresh.
- `trees.<name>` holds each tree's `fixes_branch`, `baseline_tag` (its
  last pull request tag), `unsent_count` (fixes branch commits since that
  tag that mainline doesn't have) and `stale_count` (those committed more
  than `--stale-days` ago, a week by default).

The repository's tools/kernel-tree-status takes its trees from `--tree NAME
PATH [TAG_PREFIX]` options or from its config file, never from $TREES
itself. List the maintained trees of $TREES in the config file once, or put
the `--tree` options into $TREE_STATUS; without either, the tool stops with
"no trees given". `--mainline` names the branch that tracks mainline,
`master` by default. The tool runs `git fetch --tags` in each tree unless
given `--no-fetch`. It updates `last_checked_at` only when the release data
came from kernel.org and every git command succeeded, and exits 1 when
anything failed. `--json` prints the full report.

### Anomalies
When reading the state, mention each anomaly in one line before going on,
once per session:
- `stale_count` above 0: "<tree> has <N> unsent fixes older than a week in
  <fixes branch>."
- `kernel.rc` at 6 or later and any `unsent_count` above 0: "rc<N>, <tree>
  still has <M> unsent fixes."
- `last_checked_at` more than 48 hours old: "The state was last refreshed
  <when>, refreshing." Then run $TREE_STATUS.

Otherwise say nothing about the state.

## Pitfalls
- A scratch tree held a stale branch with the same name as the devel
  branch. Work based on it reimplemented a feature that the real branch
  already had. Branch names and tips come from the tree's own clone.

## Customization
- `$TREES` lists each tree's own clone, preset, tag prefix and, for the
  patch-queue preset, its branch names. It can also name a local mainline
  clone to fetch master from.
- `$TREE_STATUS` and `$TREE_STATE` point at another status command and its
  state file.
- An overlay records branch naming conventions in the user's trees.
