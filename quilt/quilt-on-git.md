# Quilt on git

- **Load when:** `$PATCH_MGMT` is `quilt` and a task creates, edits,
  refreshes, splits or reorders quilt patches, pushes or pops the stack,
  commits the branch that holds it, or turns the patches into git commits
  or mail for a review, a posting, a pull request or an apply.
- **Parameters:** `$PATCH_MGMT`, `$COMMIT_WORD`, `$PLAN_DIR`, `$SCRATCH`,
  `$WORKTREE_ROOT`, `$KAW_DIR`, `$WIDTHS`, `$BUILD`
- **Depends on:** [core/git-safety](../core/git-safety.md),
  [core/safe-operations](../core/safe-operations.md),
  [core/stop-and-ask](../core/stop-and-ask.md),
  [patches/line-widths](../patches/line-widths.md),
  [code/kernel-idioms](../code/kernel-idioms.md),
  [review/kernel-build](../review/kernel-build.md),
  [review/patchset-review](../review/patchset-review.md),
  [submit/patchset-posting](../submit/patchset-posting.md),
  [submit/single-patch](../submit/single-patch.md),
  [submit/github-pr](../submit/github-pr.md),
  [mail/mail-compose](../mail/mail-compose.md),
  [mail/lore-thread](../mail/lore-thread.md),
  [review/checkpatch](../review/checkpatch.md),
  [debug/event-tracing](../debug/event-tracing.md),
  [test/perf-comparison](../test/perf-comparison.md)

A quilt-on-git branch starts at the upstream base, and its own commits add
only the series and patch files in `patches/` and the plans in `$PLAN_DIR`.
quilt applies the patches to the working tree. The patch files are the work
product, and the branch's history records them. The other modules describe
patch work with git. [Git terms in other modules](#git-terms-in-other-modules)
gives the quilt form of each.

## Always-on rules
- [GATE] Before editing a source file in a quilt-on-git tree: `quilt top`
  names the patch the edit belongs to, and `quilt add` put the file in it
  before its first change, because quilt diffs against the copy `quilt add`
  saves.
- [GATE] Before recording a patch: the gates the enabled modules set
  before committing a patch passed on it, and only `quilt refresh` wrote
  its diff and diffstat ([patch files](#patch-files)), because the patch
  file is the commit.
- [GATE] Before pushing or popping patches: the user asked for it or for
  the rework, edit, commit or sync it is part of, or it moves only the
  agent's new patches or runs in a worktree the agent made for a check,
  and `quilt diff -z` is empty, because the user may be working on the
  stack.
- [GATE] Before resolving a reject: it came from a rework of the agent's
  own patches that the user asked for, and every line of every hunk is
  accounted for. Any other reject, offset or fuzz stops the work, because a
  hunk applied from memory or by fuzz loses a change without an error.
- [GATE] Before changing git state in a quilt-on-git tree: a commit, pull,
  merge, rebase or checkout needs every patch popped and `git status`
  clean, except for the `patches/` and `$PLAN_DIR` changes a commit records
  as a new commit, never an amend, because git tracks only the base there.
- Change the agent's own patches, those of the user's working stack, only
  through quilt, never by a rebase, cherry-pick or replay of git commits
  made from them, because the next import discards a change made in git.

## [GATE] Checklist
### Before editing a source file in a quilt-on-git tree
- [ ] `quilt top` names the patch this edit belongs to. The user may have
      pushed or popped since the last look.
- [ ] `quilt files` lists the file, or `quilt add <file>` runs now, before
      the first change. A new file is added before it is created, so that
      the patch records it as new. `quilt new` adds no files.
- [ ] Before `quilt add`, the file held no unrecorded change. For a file no
      applied patch touches, `git diff <file>` is empty. A file that holds
      one goes through
      [restoring an edit](#restoring-an-edit-that-belongs-to-no-patch).
- [ ] After an empty patch was pushed, its files were added again. An empty
      patch pushes only from a zero-byte or missing patch file, and it comes
      back with no files.
- [ ] `quilt diff -z` was read, so that unrefreshed edits the user made in
      the file are known and kept.

### Before recording a patch
- [ ] `quilt top` names the patch being refreshed.
- [ ] Every "Before committing a patch" gate of the enabled modules passed
      on the patch's header and its current `quilt diff`
      ([checking a patch](#checking-a-patch-before-recording-it)).
- [ ] The full `quilt diff` was read. Every hunk is justified by the
      patch's purpose, with no moved code and no collateral changes
      ([code/kernel-idioms](../code/kernel-idioms.md#changing-code)).
- [ ] Lines 1 and 2 of the patch file are `From:` and `Subject:`.
- [ ] A direct edit stayed above the diffstat
      ([editing a description](#editing-a-description)).
- [ ] After a function signature changed, the later patches still apply
      ([reworking the stack](#reworking-the-stack)).

### Before pushing or popping patches
- [ ] `quilt top` names the patch on top. Before a pop, tell the user which
      patch that is, so that the stack can go back there.
- [ ] One of these covers the operation: the user asked for this push or
      pop, itself or as a step of a rework or an edit; it is the `quilt pop
      -a` of a commit $COMMIT_WORD authorized or of a sync the user asked
      for; it moves only patches the agent created in the task; or it runs
      in a worktree the agent made under $WORKTREE_ROOT for a check. Never push
      existing patches as a side effect of stacking new work. Ask instead.
- [ ] `quilt diff -z` is empty. Otherwise refresh first, or get the user's
      approval to drop the change.
- [ ] No `-f` follows a "does not remove cleanly" from `quilt pop`
      ([removing an orphan hunk](#removing-an-orphan-hunk)). A patch forced
      on by `quilt push -f` was refreshed before it pops, because `quilt
      pop -f` on it discards what the push applied. Forced operations are
      never chained.
- [ ] No build or test is reading the tree, and the stack moves only as far
      as the next step needs ([working the stack](#working-the-stack)).

### Before resolving a reject
- [ ] The reject came from a rework of the agent's own patches that the
      user asked for, in which an earlier patch was edited, reordered or
      split. Any other reject, and any offset or fuzz outside such a
      rework, stops the work
      ([core/stop-and-ask](../core/stop-and-ask.md#genuine-blockers)).
- [ ] Every hunk in each `.rej` is handled as
      [core/git-safety](../core/git-safety.md#rebasing-the-agents-own-patches)
      describes for a conflicting hunk.
- [ ] The patch is rebuilt from its current text and the `.rej`, never
      recreated from memory.

### Before changing git state in a quilt-on-git tree
- [ ] The operation is a commit, pull, merge, rebase or checkout. Reading
      commands, such as `git show` and `git diff`, need none of this.
- [ ] `quilt top` reports that no patch is applied.
- [ ] `git status --short` lists nothing. For a commit, it lists only paths
      under `patches/` and `$PLAN_DIR`.
- [ ] Every commit, the merge commit of a pull included, has $COMMIT_WORD
      in the user's instruction or is a step of a
      [sync](#syncing-after-an-apply) the user asked for
      ([core/git-safety](../core/git-safety.md#before-a-commit-or-amend)).
- [ ] For a commit: `git diff --cached --stat` lists only paths under
      `patches/` and `$PLAN_DIR`, `textfill check` passes on every patch of
      the series, every patch starts with `From:` and `Subject:`, and the
      commit is a new one, never `--amend`.

## Procedure
### Setting up a tree
1. Create `patches/` at the top of a new tree or worktree before its first
   `quilt new`. quilt takes the nearest directory at or above the current
   one that holds `patches/` or `.pc` as the top of the tree. Without one at
   the new tree's top, a parent directory's series takes the patch, with
   paths that start with the new tree's directory name. Never create an
   empty `.pc`, which makes quilt ask for `quilt upgrade`.
2. After the first `quilt add`, check that `quilt files` prints paths
   relative to the new tree's top.
3. If a parent directory's series took the patch, save the change with
   `git diff` under $SCRATCH, pop the patch in the parent tree and delete
   it there
   ([core/safe-operations](../core/safe-operations.md#before-a-destructive-or-forced-operation)),
   and check that the parent's series file and `.pc/applied-patches` are
   back to their earlier content. Then set up the new tree as step 1 says,
   create the patch, `quilt add` its files and apply the saved diff with
   `git apply`.
4. The kernel's .gitignore ignores `patches` and `.pc`, so patch files are
   staged with `git add -f`.
5. Start an investigation that may yield a fix on a quilt-on-git branch,
   with the diagnostic patches above the fix in the series, so that the fix
   can be refreshed and posted without cutting it out of the probe code by
   hand ([debug/event-tracing](../debug/event-tracing.md#always-on-rules)).

### Creating a patch
1. Check that the patch the new one goes after is on top. `quilt new`
   inserts the new patch right after the top patch.
2. `quilt new <name>`, named as [patch files](#patch-files) says.
3. `quilt add` each file before changing it.
4. Edit, and run the formatter the tree uses for languages that have one,
   such as `cargo fmt` for Rust, before each refresh.
5. Record the patch as
   [checking a patch](#checking-a-patch-before-recording-it) describes.
6. Write the header last, once quilt has written the diff: `From:`,
   `Subject:`, the description and the tags
   ([editing a description](#editing-a-description)).

### Checking a patch before recording it
1. With the patch on top, build a draft from its header and its current
   diff, and check it, passing the three $WIDTHS values. Add
   `--tag NOT_SIGNED_OFF` for a patch that carries that line:

   ```
   { quilt header; quilt diff -p ab --no-index; } > $SCRATCH/<topic>/draft.patch
   $KAW_DIR/tools/textfill check --desc D --comment C --code K $SCRATCH/<topic>/draft.patch
   ```

2. Run `textfill refill` on the draft from the top of the tree. It rewrites
   the flagged comment paragraphs in the source files. A flagged
   description paragraph is fixed in the patch file itself: with the patch
   on top and `quilt diff -z` empty, `textfill refill patches/<name>`
   rewrites it in place. Otherwise fill it with `textfill fill`
   ([patches/line-widths](../patches/line-widths.md#filling-prose)).
3. Run the other checks the enabled modules gate before committing a patch,
   such as the description and comment checklists, on the draft.
4. Read the full `quilt diff`, then run `quilt refresh`.

### Editing a description
1. Edit the patch file above the `---` line. The edit needs no push, pop or
   refresh, because it touches neither the source nor `.pc`, and a later
   refresh keeps the text above the diff.
2. Change only the lines being edited. Never write the whole file anew,
   which puts the diff below the header at risk.
3. Keep the replaced text clear of the diffstat. A diffstat line can end in
   a space, and a tool that strips trailing whitespace from the replaced
   text changes the line. The next refresh no longer recognizes it, keeps
   it as header text and writes a new diffstat below it. To add a header to
   a patch that has none, anchor the edit on the `---` line and at most the
   first few characters of the line after it.
4. Check the patch file itself: run `textfill check patches/<name>` and the
   other checks of
   [checking a patch](#checking-a-patch-before-recording-it) step 3 on it,
   and fix the findings as its step 2 describes.

### Changing a patch's code
1. Bring the patch to the top under
   [the gate](#before-pushing-or-popping-patches).
2. `quilt add` each file the patch doesn't have yet, before changing it.
3. Edit the source, then record the patch as
   [checking a patch](#checking-a-patch-before-recording-it) describes.
4. Refresh only the patch being changed, and leave the later patches
   popped. Carrying a change of context up the stack is a separate
   operation ([reworking the stack](#reworking-the-stack)).
5. A posted patch whose diff changed gets a vN note
   ([submit/patchset-posting](../submit/patchset-posting.md#vn-notes)). Take
   its number from the highest note in both the working copy and the
   committed one, `git show HEAD:patches/<name>`.

### Restoring an edit that belongs to no patch
An edit made before its `quilt add` leaves quilt's saved copy holding the
edited file, and `quilt refresh` reports "Nothing in patch". An empty patch
pushed back from a zero-byte or missing patch file comes back with no
files, so an edit made after the push belongs to no patch at all. Never
refresh the empty patch over such edits.
1. Copy each edited file under $SCRATCH.
2. If the top patch lists the file, `quilt remove <file>` takes it out of
   the patch.
3. Rebuild the file as the stack below the top patch leaves it: its
   content at the base, with the hunks for it from every applied patch
   below the top one, in order. filterdiff comes from patchutils:

   ```
   git show HEAD:<path> > $SCRATCH/<topic>/clean
   for p in $(quilt applied | head -n -1); do
       filterdiff -i '*/<path>' "patches/$p" | patch -p1 $SCRATCH/<topic>/clean
   done
   ```

4. Copy the rebuilt file in, `quilt add` it, copy the saved file back and
   refresh.
5. Check with `quilt diff` that the patch holds exactly the edits.

### Removing an orphan hunk
When `quilt pop` says that a patch "does not remove cleanly", one of its
files holds a change that no applied patch records. A common cause is a
rewrite of the patch file that dropped a hunk: the change stays in the
source, the stack still builds, and only the pop fails.
1. Run `quilt diff -z` to see the change. Never `quilt pop -f` past it,
   which discards it.
2. Save the change under $SCRATCH and revert it in the source, so that the
   patch pops.
3. Push to the patch that should hold the change, `quilt add` the file,
   make the change again and refresh.
4. Check that `quilt diff -z` is empty, that the stack applies without a
   reject, that the build passes and that the pop that failed now
   succeeds.

### Resolving rejects
During a rework of the agent's own patches that the user asked for, such as
an edit, a reorder or a split of an earlier patch, later patches reject on
their way back up. Resolve them in place, one patch at a time.
1. Before popping, when the user has unrefreshed edits, keep the output of
   `quilt diff -z` under $SCRATCH.
2. `quilt push -f <patch>` applies what it can, leaves `.rej` files and
   marks the patch as needing a refresh. The rework instruction covers this
   forced push, so it needs no separate approval under
   [core/safe-operations](../core/safe-operations.md#before-a-destructive-or-forced-operation).
3. Read every `.rej` and the source it targets. Apply each hunk as
   [core/git-safety](../core/git-safety.md#rebasing-the-agents-own-patches)
   describes, copying the user's text from the `.rej` verbatim.
4. Before deleting the `.rej` files, read each against the source again
   and confirm that nothing the user wrote was lost.
5. Refresh the patch before pushing the next one. Offsets within the rework
   are refreshed away the same way.
6. Outside such a rework, a reject, an offset or a fuzz stops the work: an
   import for a posting, an apply, or an untouched stack that rejects after
   the base moved. Report it as
   [core/stop-and-ask](../core/stop-and-ask.md#procedure) describes. Never
   fuzz, force, skip the patch or narrow the scope on your own.

### Splitting, reordering and renaming
1. Never write a patch file by hand. quilt writes every diff.
2. To split a patch, pop to the patch before it, `quilt new` the part, `quilt
   add` its files, apply the part's changes to the source and refresh. Then
   `quilt push -f` the original patch, whose hunks for the part now reject,
   resolve them as [resolving rejects](#resolving-rejects) describes and
   refresh it. The split instruction covers that forced push.
3. To reorder, pop below the patches that move, edit the series file, then
   push them one at a time, refreshing each whose context moved.
4. Rename an identifier one file at a time with explicit edits
   ([core/safe-operations](../core/safe-operations.md#before-editing-a-code-or-patch-file)).
   Across patches, pop to the patch that introduces the name, change it
   there and refresh, then push each later patch, change its uses and
   refresh it.

### Reworking the stack
1. Make each change in the earliest patch that needs it: pop to it, change
   it, refresh, push to the next patch that needs a change, and repeat.
   Never make every change on top and expect it to move down later, because
   quilt has no way to redistribute a change.
2. A fix to code or a comment goes into the patch that introduced it, even
   when that takes more pushes and pops than a fix on top.
   `quilt annotate <file>` shows which applied patch last changed each
   line, and `-P <patch>` stops it at an earlier patch.
3. Build with `$BUILD` after each refresh that changes code, so that a
   break between patches shows at the patch that caused it
   ([review/kernel-build](../review/kernel-build.md#procedure)).
4. When an earlier patch changes a function signature that later patches
   use, pop to the patch that owns the signature, change it and refresh.
   Then push the later patches one at a time. Rejects appear at the call
   sites. Resolve them as [resolving rejects](#resolving-rejects) describes,
   one patch at a time, never several at once.
5. Once the stack is stable again, add a vN note to every posted patch
   whose diff changed in this round
   ([submit/patchset-posting](../submit/patchset-posting.md#vn-notes)).

### Committing the quilt-on-git branch
1. If patches are applied, tell the user which patch is on top, check that
   `quilt diff -z` is empty and run `quilt pop -a`. The commit instruction
   covers this pop.
2. Run `git status --short`. Only paths under `patches/` and `$PLAN_DIR`
   may show. A changed source file stops the commit until its cause is
   found. Never `git checkout -- .` or `git checkout -- patches/` after the
   pop: `quilt pop` already restored the sources, and a checkout of
   `patches/` discards patch changes not yet committed.
3. Check every patch of the series with `textfill check`, and that each
   starts with `From:` and `Subject:`.
4. Stage the patches and the plans by path, deleted patch files included,
   and check what is staged:

   ```
   git add -f patches $PLAN_DIR
   git diff --cached --stat
   ```

   If any other path is staged, unstage everything with `git reset` and
   find out why.
5. Commit with `git commit`, a new commit every time, never `--amend`. The
   patches' own history is in their vN notes, and each commit of the branch
   records one round.
6. Push the branch to the user's backup remote, when one is set up
   ([core/git-safety](../core/git-safety.md#customization)).

### Importing the patches into a git branch
Posting, a pull request and a review need the patches as git commits. `git
quiltimport` makes one commit per patch, with the author from `From:` and
the subject from `Subject:`.
1. Create a branch from the base, named by its branch name, in a new
   worktree under $WORKTREE_ROOT of the tree the commits go to, because the
   import, the build and git format-patch run there and a publication tree
   takes no builds
   ([core/safe-operations](../core/safe-operations.md#publication-trees)):
   `git -C <tree> worktree add -b <branch> $WORKTREE_ROOT/<topic>-post
   <base>`. A new version gets a new branch, such as `<branch>-v2`. Never
   reset a branch that was posted or pushed. An import onto the tree's
   devel, fixes or integration branch is an apply and waits for the user's
   word for it
   ([maintain/apply-patches](../maintain/apply-patches.md#always-on-rules)).
   A new branch for posting the user's own patchset is part of preparing
   the posting and needs no such word.
2. When the series holds more than the patches to import, write a series
   file under $SCRATCH with only those, in order. git quiltimport imports
   every line of the series it gets, skipping comments.
3. Give every patch `From:` and `Subject:`. Without an author, git
   quiltimport doesn't fail: in a shell without a terminal it loops
   forever, printing "No author found". `git quiltimport -n` names the
   patches that lack one. Add missing headers to copies under $SCRATCH,
   never to the patch files.
4. From the top of the tree, import:

   ```
   git quiltimport --patches <patches dir> --series <series file>
   ```

5. Check `git status` and `git log --oneline <base>..`: a clean tree, and
   one commit per patch in series order. A failed import stops at the
   failing patch, with the patches before it committed. That stops the
   work. Never force it through.
6. git quiltimport applies each patch with `git apply -C1`, which accepts
   context that drifted. When the import starts from the quilt-on-git
   branch's base, as for a pull request, compare the result with the
   stack. In a worktree of the quilt-on-git branch that the agent makes
   under $WORKTREE_ROOT for this check, never the user's own, with every
   patch pushed, the source files must match the imported branch. Marking
   the untracked files intent-to-add, by name, makes the files that patches
   create show in the diff:

   ```
   git ls-files -o --exclude-standard -z | xargs -0 git add -N --
   git diff --stat <imported branch> -- . ':!patches' ':!.pc' ':!<plan dir>'
   ```

7. Never fix a commit on the imported branch by a rebase, an amend, a
   cherry-pick or a filter-branch. Fix the patch in the quilt-on-git tree
   and import again onto a branch created afresh from the base.

### Posting a patchset
These steps replace the git steps of
[submit/patchset-posting](../submit/patchset-posting.md) that make and
change commits.
1. Before step 1 of
   [posting a patchset](../submit/patchset-posting.md#posting-a-patchset),
   import the patches to post onto a new branch from the base, in a
   worktree of the tree the posting comes from
   ([importing](#importing-the-patches-into-a-git-branch)). The posting
   steps that follow run in that worktree.
2. For a new version, this replaces steps 4 and 7 of
   [posting a new version](../submit/patchset-posting.md#posting-a-new-version):
   make the changes and the vN notes in the patch files, then import the
   patches onto a new branch for the version from the base. Comparing
   versions with `git range-diff` works on the two imported branches.
3. This replaces step 2 of collecting review tags in
   [submit/patchset-posting](../submit/patchset-posting.md#collecting-review-tags),
   because b4 trailers rewrites commits, not patch files. Add each tag by
   editing the patch file above `---`. Find a reply's patch by its subject,
   comparing `quilt series` and the patches' `Subject:` lines, never by its
   number. Committing the result waits for $COMMIT_WORD.
4. To get new tags into commits imported earlier, edit the patch files and
   import again onto a branch created afresh from the base. Never
   filter-branch or `rebase --exec` the imported commits, because the next
   import regenerates them from the patches. Re-creating a branch that was
   pushed is a branch-structural operation
   ([core/git-safety](../core/git-safety.md#branch-structural-operations)).

### Posting one patch
A single patch from the stack goes out from its patch file, with no branch,
import or `git format-patch`. This replaces step 4 of both procedures in
[submit/single-patch](../submit/single-patch.md#procedure), posting one
patch and re-posting one patch of a patchset.
1. Check that `quilt diff -z` for the patch is empty. If it isn't, ask the
   user, because the change may be meant for the patch.
2. Build the mail file as
   [mail/mail-compose](../mail/mail-compose.md#building-a-mail-file)
   describes: the mail headers, with the subject prefix the posting needs
   and, for a re-post, In-Reply-To and References, then a blank line and
   the patch file from its description on. The patch file's `Subject:`
   moves into the mail's Subject. Its `From:` line is left out when the
   author is the sender, and starts the body otherwise.
3. The file goes out through a sendmail tool, not git send-email
   ([mail/mail-compose](../mail/mail-compose.md#the-git-send-email-command)).

### Opening a GitHub pull request
This replaces step 1 of opening or updating a pull request in
[submit/github-pr](../submit/github-pr.md#opening-or-updating-a-pull-request)
when the pull request branch is made in the repository that holds the
quilt-on-git branch.
1. Commit the quilt-on-git branch
   ([committing](#committing-the-quilt-on-git-branch)), which needs
   $COMMIT_WORD in the user's instruction. Checking out another branch
   takes the committed `patches/` out of the working tree, so the import
   reads the patches from the branch.
2. Create the pull request branch from the base:
   `git checkout -b <pr branch> <base>`. For an update, the branch is
   created afresh from the base. That rewrite is a branch-structural
   operation, and its push is a force-push
   ([core/git-safety](../core/git-safety.md#branch-structural-operations)).
3. Extract the committed `patches/` of the quilt-on-git branch into a new
   directory under $SCRATCH, and give the copies that lack them `From:` and
   `Subject:` headers:

   ```
   d=$(mktemp -d $SCRATCH/<topic>/import.XXXXXX)
   git archive <quilt branch> patches | tar -x -C "$d"
   ```

4. Import from the top of the repository with `--patches "$d/patches"
   --series "$d/patches/series"`, and compare with the stack
   ([importing](#importing-the-patches-into-a-git-branch), steps 2-6).
5. Run step 2 of the same procedure in
   [submit/github-pr](../submit/github-pr.md#opening-or-updating-a-pull-request)
   on the imported branch, with one difference: a formatter finding is
   fixed in the quilt patches, which are committed and imported again. A
   lock-file update is the one fix folded in on the pull request branch,
   because it is generated from the imported tree.
6. Continue with step 3 there.
7. Check out the quilt-on-git branch again.

### Taking a posted patchset into a series
1. Fetch the patchset into $SCRATCH with `b4 am -o $SCRATCH/<topic>
   <msgid>`, and read its thread
   ([mail/lore-thread](../mail/lore-thread.md#procedure)).
2. Split the mbox into one file per patch:
   `git mailsplit -o$SCRATCH/<topic>/split $SCRATCH/<topic>/<name>.mbx`.
3. Each split file keeps the mbox `From` line, the mail headers and a
   Subject with its `[PATCH vN i/M]` prefix. Rewrite each into the form of
   [patch files](#patch-files), under a short name in `patches/`. git
   mailinfo separates the author, the subject without its prefix, the
   message and the patch:

   ```
   git mailinfo $SCRATCH/<topic>/msg $SCRATCH/<topic>/diff < <split file> > $SCRATCH/<topic>/info
   { printf 'From: %s <%s>\nSubject: %s\n\n' "<Author>" "<Email>" "<Subject>"
     cat $SCRATCH/<topic>/msg $SCRATCH/<topic>/diff; } > patches/<name>
   ```

   The three values come from the `Author:`, `Email:` and `Subject:` lines
   of the info file. The next refresh drops the mail signature that ends
   the diff.
4. Add the names to the series file in the patchset's order, never in the
   order of the names, after the applied patches, at the place the user
   wants. Pushing them falls under
   [the gate](#before-pushing-or-popping-patches).

### Syncing after an apply
Only when the user asks, after some patches of the stack were applied to a
public tree. The user's request to sync covers the pull's merge commit,
the final commit and the push.
1. Tell the user which patch is on top, and pop every patch.
2. Check that `git status` is clean for the pull
   ([the gate](#before-changing-git-state-in-a-quilt-on-git-tree)).
   Uncommitted changes in `patches/` or `$PLAN_DIR` stop the sync and go to
   the user.
3. Pull the updated public branch into the quilt-on-git branch.
4. Comment out the applied patches in the series file with `#`.
5. Commit the branch and push it to its backup remote. When every patch was
   applied, the branch has nothing left and can go.

## Reference
### Patch files
- Every patch file starts with `From:` and `Subject:` on lines 1 and 2,
  then a blank line, the description, any vN notes and the tag block, then
  `---`, the diffstat and the diff. git quiltimport and tools/textfill read
  the two header lines.
- A patch not meant for merging, such as a demonstration posted with a
  patchset, carries `NOT_SIGNED_OFF: <reason>` in place of its
  Signed-off-by. The missing sign-off keeps it from being applied, and the
  line says why. tools/textfill treats it as a tag with
  `--tag NOT_SIGNED_OFF`.
- Name a patch with one to a few short lowercase words joined by hyphens,
  such as `fix-foo-leak`, with no number prefix, because the series file
  sets the order and numbers go stale when patches move. The `Subject:`
  carries the title. Never rename an existing series entry.
- The diff and the diffstat change only through `quilt refresh`. Never edit
  a hunk in the patch file, not even whitespace or a wrongly wrapped line,
  and not when the result would be identical, because a hand-edited hunk
  drifts from its line counts and from the source. Never type, trim or
  deduplicate a diffstat. When one looks wrong, find the cause in the
  header, fix that, and let `quilt refresh` write it again.
- Never run `quilt header -r`. It replaces everything above the diff, the
  `---` line and the diffstat included, so the description runs into the
  diff.
- A patch with a header or a diffstat and no hunks doesn't push: patch
  reports "Only garbage was found in the patch input". A refresh of a patch
  with no changes writes such a diffstat, `0 files changed`, so the patch
  no longer pushes once it is popped.

### Working the stack
- Push or pop only when the next concrete step needs the stack in that
  state, such as editing a patch's source or building a code change.
  Restoring a file, editing the series or a description, and bookkeeping
  need no stack movement. Never push again afterwards "to restore the
  working state". Leave the stack where it is.
- Run one quilt operation or build at a time. Never push or pop while a
  build or a test reads the tree, because it then works on an unknown mix
  of patches.
- While iterating on a patch with the user, keep it on top. The user edits
  the source live, and the next refresh records those edits into the top
  patch. No `quilt push -a` to check the whole series while working through
  patches one by one. Check up to the patch at hand.
- `quilt diff` shows the patch. `quilt diff -z` shows the changes not yet
  refreshed into it, and only a non-empty `quilt diff -z` means the patch
  needs a refresh.
- When reporting updated or new patches, prefix each with its number in
  the current series, such as `19 fix-foo-leak`, because the user reviews
  in posting order.

### User-edited prose
- When a description or a comment in a patch differs from what the agent
  last wrote, the user edited it. Never restore the earlier wording, not
  during unrelated edits, splits, reject resolution or rebuilds from `.rej`
  files.
- Copy the current text from the patch or the `.rej` verbatim, never
  retype it from memory.
- When scanning `git diff patches/` for unintended changes, revert only
  structural damage, such as a duplicated diffstat line, never a deletion
  of prose.
- When it is unclear whose wording it is, recover the user's version from
  the session history or the `.rej`, and ask.

### Git terms in other modules
- **Committing a patch**: `quilt refresh` for the code and an edit of the
  patch file for the description, each after
  [checking a patch](#checking-a-patch-before-recording-it). A refresh
  needs no $COMMIT_WORD, which gates only the git commit of the branch.
- **The staged diff**, and `git diff` in a check of a change in progress:
  `quilt diff`, read before the refresh. **Uncommitted changes in a
  patch's files**: `quilt diff -z`.
- **Amending a commit message**:
  [editing a description](#editing-a-description).
- **The patchset's earlier patches**: the patch files before it in `quilt
  series`.
- **A range of commits**, for checkpatch `-g`, `git range-diff`,
  `git format-patch` or the reviewers' worktrees: import the patches first
  ([importing](#importing-the-patches-into-a-git-branch)).
- **Patchset review**: import the stack onto a branch from the base in a
  worktree under $WORKTREE_ROOT and review that range
  ([review/patchset-review](../review/patchset-review.md#tree-state-for-each-reviewer)).
  Record the reviewed head by patch name, because the imported commits
  don't last. A review of one patch takes the patch the user names, or the
  one `quilt top` names.
- **checkpatch on a patch file in `patches/`**: through stdin, as
  [review/checkpatch](../review/checkpatch.md#procedure) step 2 gives for
  files git tracks.
- **Width checks of a patchset**: `textfill check` over the patch files of
  the series, which carry `From:` and `Subject:`
  ([patches/line-widths](../patches/line-widths.md#toolstextfill)).
- **A debug commit**: a debug patch above the fix in the series.
- **The agent's own patches**: the patches of the user's working stack, as
  opposed to contributors' patches and published history.
- **Rebasing the agent's own patches during a rework**: reworking the
  stack, with rejects in place of conflicts
  ([resolving rejects](#resolving-rejects)). Because the agent's own
  patches change only through quilt, the conflict exception of
  [core/git-safety](../core/git-safety.md#rebasing-the-agents-own-patches)
  never reaches a git rebase of them.
- **Keeping the old branch before a rework**: commit the quilt-on-git
  branch, or copy `patches/` under $SCRATCH.
- **Building the base without the change**, such as for a
  [before/after comparison](../test/perf-comparison.md#beforeafter-comparison):
  a separate worktree at the base commit. Popping the whole stack in place
  for it needs the user's instruction for that pop and the push after it.
- **Committing before switching branches**: every patch refreshed and
  popped, and the quilt-on-git branch committed.

## Pitfalls
- A fresh worktree had no `patches/`. `quilt new` there found a parent
  directory's `.pc` and added the patch to the user's own series, with
  paths prefixed by the worktree's name. Create `patches/` first.
- A patch that had never been refreshed, with no patch file yet, was popped
  and pushed back. It came back with no files, the edits made afterwards
  belonged to no patch, and the refresh reported nothing in the patch. Run
  `quilt files` after such a push.
- A substitution meant for one function name also matched a longer name
  that contained it, in two patch files at once. Rename with explicit
  edits, one file at a time.
- A reject spanned a validation check, a helper and a rewritten kerneldoc
  comment that happened to be adjacent. Only the remembered part was
  applied by hand, and the refresh after it put back the original
  kerneldoc over the user's rewrite. Account for every line of the `.rej`.
- An import for a posting applied seven of eight patches. The eighth was
  skipped, then pushed through with a three-way apply, a raised fuzz factor
  and a force, instead of stopping at the first failure.
- A pop and push of the whole stack ran while a build read the tree. The
  build compiled an unknown mix of patches and had to be redone.

## Customization
- `$PATCH_MGMT` set to `quilt` says that the user keeps patchsets this
  way, and `$PLAN_DIR` names the plan directory committed with the
  patches.
- The procedures assume that `quilt refresh` writes a diffstat and
  `a/` `b/` paths without timestamps or Index lines, such as with
  `QUILT_REFRESH_ARGS="--diffstat --no-timestamps --no-index -p ab"` in
  `~/.quiltrc`.
- An overlay usually names the quilt-on-git tree and its worktrees, the
  backup remote, the identity for headers added at import, and how a fresh
  worktree's quilt state is seeded when the user's directories nest.
