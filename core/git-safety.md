# Git safety

- **Load when:** committing or amending, adding files to the index,
  rebasing, resetting or force-pushing a branch, overwriting a tag or a
  branch, pulling or merging, switching branches with local changes, adding
  remotes or upstream tracking, or when a git merge, rebase, cherry-pick or
  am stops on a conflict.
- **Parameters:** `$COMMIT_WORD`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [core/stop-and-ask](../core/stop-and-ask.md)

## Always-on rules
- [GATE] Before a commit or amend: $COMMIT_WORD is in the user's
  instruction for the task, or the commit is a step of an instructed apply
  or a merge whose message the user confirmed, because the user decides
  what enters history. $COMMIT_WORD never authorizes applying patches to a
  publication tree.
- [GATE] Before a branch-structural operation: the user instructed this
  operation itself, even when a workflow seems to need it, because it
  rewrites or mixes branches that other people and trees build on.
- [GATE] Before resolving a conflict: the user saw the conflicted files and
  regions and asked for a resolution, even in auto mode, because a
  resolution decides how two lines of work combine. The one exception is a
  rebase of the agent's own patches during a rework the user asked for.
- [GATE] Before presenting a conflict or proposing its resolution: show
  where it came from, which commits on each side collide and how, and per
  commit how the resolution keeps its intention, auto-merged hunks
  included, because a check against the marked region or an earlier
  adaptation inherits whatever they missed.
- Add files to the index by name or pattern after reading `git status`,
  never with `git add -A` or `git add .`, because a working tree holds files
  that don't belong in the commit.
- Never `git stash`, and never pull or merge into a tree with uncommitted
  changes. Stop and ask instead, because a stash keeps work outside any
  branch and an aborted merge can take uncommitted changes with it.

## [GATE] Checklist
### Before a commit or amend
- [ ] One of these authorizes it: $COMMIT_WORD in the user's instruction
      for the task, an apply the user instructed of which the commit is a
      step ([maintain/apply-patches](../maintain/apply-patches.md#applying)),
      or a merge commit whose message the user confirmed
      ([maintain/merges](../maintain/merges.md#making-a-merge)). Never
      commit or amend otherwise.
- [ ] For a rework: an instruction that includes $COMMIT_WORD and asks for
      a rework of an unpublished branch covers the commits, amends and
      rebases of that rework.
- [ ] The commit doesn't apply patches to a publication tree. That needs an
      explicit apply instruction
      ([maintain/apply-patches](../maintain/apply-patches.md#always-on-rules)),
      never $COMMIT_WORD.

### Before a branch-structural operation
- [ ] The user instructed this specific operation
      ([the list](#branch-structural-operations)). A workflow step that
      seems to need it doesn't count, and neither does an instruction for
      the work around it ([what instructions mean](#what-instructions-mean)).
- [ ] For a rewrite, the branch was checked on the remote
      ([before rewriting a branch](#before-rewriting-a-branch)).

### Before resolving a conflict
- [ ] The conflict is not in a merge into for-next. That merge is aborted at
      once and never resolved there
      ([maintain/merges](../maintain/merges.md#a-conflict-on-for-next)).
- [ ] For a conflict from a merge, a cherry-pick or an am, or from a rebase
      of anyone else's work: the conflicted files and a short summary of
      each conflicting region were presented, and the user asked for a
      resolution. Until then, no `git add` of a conflicted file, no edits
      to the markers and no commit. Aborting when the user asks is fine.
- [ ] For a conflict while rebasing the agent's own patches during a rework
      the user asked for: every added and removed line of every conflicting
      hunk is accounted for, and each hunk that touches prose the user
      wrote was shown to the user first
      ([rebasing the agent's own patches](#rebasing-the-agents-own-patches)).

### Before presenting a conflict or proposing its resolution
- [ ] Where each conflict came from: the incoming commits, and the target
      commits since `git merge-base --all`, that touch the conflicted code
      or the logic the incoming commits change. Find them per function with
      `git log -L '/^<function>(/,/^}/:<file>'` and read them in full. The
      `git log -L :<function>:<file>` form can match the wrong definition
      and report nothing.
- [ ] How they collide: for each pair, what each commit intended and what
      of the other it breaks, invalidates or depends on, including hunks
      git merged without a marker.
- [ ] A resolution that keeps the intention of every involved commit,
      shown commit by commit.
- [ ] The resolution's commit message explains each conflict: the commits
      on both sides, their intents, and how the resolution satisfies each
      ([maintain/merges](../maintain/merges.md#message-shapes)).
- [ ] No item rests on a clean auto-merge, a line count, or a byte match
      with an earlier adaptation of the same patches.

## Procedure
### Before rewriting a branch
1. Before a rebase, reset or amend that a push will follow, run
   `git ls-remote <remote> <branch>` to see whether the branch is published.
2. If it is published, the rewrite is a branch-structural operation. Stop
   and present the alternatives: a merge, a cherry-pick or a new branch.
3. Before rebasing a patchset, keep the old branch under another name, so
   that the rebased patchset can be compared against it.

### When an apply hits a missing dependency
1. Stop when a patch to be applied depends on a commit the target branch
   lacks. Never back-merge without an instruction for it.
2. Present the gap and the options: a back-merge, a different target branch,
   or deferring the patch.

### When a conflict appears
1. A conflict in a merge into for-next goes to
   [maintain/merges](../maintain/merges.md#a-conflict-on-for-next).
2. A conflict from rebasing the agent's own patches during a rework the
   user asked for goes to
   [rebasing the agent's own patches](#rebasing-the-agents-own-patches).
3. Any other conflict, from `git merge`, `git cherry-pick`, `git am` or a
   rebase of anyone else's work, stops the work, even in auto mode.
4. Present the conflicted files, each conflicting region, and where each
   conflict came from
   ([the checklist](#before-presenting-a-conflict-or-proposing-its-resolution)),
   then wait.
5. If the user asks, abort with the command's `--abort`.
6. When the user asks for a resolution, follow
   [resolving a conflict](#resolving-a-conflict).

### Resolving a conflict
1. Find the commits on each side that touch the conflicted code or the
   logic the incoming commits change, per function, as
   [the checklist](#before-presenting-a-conflict-or-proposing-its-resolution)
   describes. Read their diffs in full.
2. State what each commit intended, and for each pair, what of the other it
   breaks, invalidates or depends on.
3. For every incoming commit, state the behavior it changes and find what
   the target did to that logic since the merge base. Apply the incoming
   edit where the target moved the code. Git merges the moved copy without
   a marker and without the edit.
4. Resolve minimally. Take one side's code verbatim where possible and
   change only what reconciles the two. No restructuring, no renames and no
   cleanups, even provably equivalent ones. Note those as follow-up commits
   for after the merge.
5. Verify behavior, not lines, because a line or count check proves
   nothing. A target that consolidated duplicated logic into one place
   takes the incoming change once. A target that split multiplexed handling
   into several paths takes it on every path that carries the case. Show
   that the merged code has each incoming behavior on every path, including
   where git merged without a marker.
6. Present the resolution commit by commit. For a merge commit, present its
   message with it ([maintain/merges](../maintain/merges.md#merge-messages)).

### Rebasing the agent's own patches
During a rework the user asked for, such as editing, reordering or
splitting an earlier commit, the agent resolves the conflicts of its later
commits in place.
1. Treat each conflicting hunk as a checklist. Account for every added and
   removed line: applied, dropped for a stated reason, or superseded. A
   hunk often spans several unrelated changes that happened to be
   adjacent, and applying only the part that comes to mind drops the rest.
2. Show the user each hunk that touches prose the user wrote before
   resolving it.

## Reference
### Branch-structural operations
Each needs an instruction for that operation, even when a workflow seems to
require it:
- a back-merge between integration branches, such as a fixes branch into a
  devel branch
- `git push --force` or `--force-with-lease` to a published branch
- `git reset --hard` on a branch that was pushed
- a merge into a shared branch, other than the standard for-next merge of an
  apply the user instructed
- a rebase, amend or filter-branch of anything on a remote

### What instructions mean
- "Pull X into Y" means a merge or a fast-forward, never a rebase of X for
  a tidier graph.
- "Apply" never authorizes the branch work around the apply, such as a
  back-merge for a missing dependency.
- "Revert" leaves the mechanism to confirm: a force push or a revert
  commit.

### Forcing and overwriting
- `--force` on a push, reset or checkout, and overwriting a tag or a branch,
  need the user's explicit confirmation every time
  ([core/safe-operations](../core/safe-operations.md#gate-checklist)).
- Published tags are permanent: never move or delete one, because others
  have already fetched it.

### Remotes, pulls and local state
- Never set upstream tracking (`-u`), add a remote (`git remote add`) or
  create a remote-tracking branch without permission.
- A branch that tracks mainline, such as master, only fast-forwards. Report
  a pull into it that would not fast-forward.
- Pulling into any other branch needs permission.
- Work is committed before switching branches, under the commit gate above,
  instead of being stashed.
- Where the work happens, in place or in its own worktree, is in
  [core/safe-operations](../core/safe-operations.md#working-files).

## Pitfalls
- A one-sided conflict can hide an auto-merged copy. One side renamed a
  call inside a function, and the other side had moved that function to
  another file. Git reported the old file as a one-sided conflict and
  merged the moved copy without a marker, still calling the old name.
  Taking the target's side of the conflict alone would have broken the
  build. Reading both sides' commits showed that the rename belonged at the
  function's new home.
- A byte match is a line check too. A proposed resolution once named only
  the commit behind the marked region and was justified by a byte match with
  an earlier adapted copy of the same patches. That copy's misses came
  along with it. The per-commit trace comes first, every time.

## Customization
- `$COMMIT_WORD` sets the word that authorizes a commit.
- An overlay can narrow which tree and branch a commit lands on, and name a
  backup remote that commits get pushed to.
