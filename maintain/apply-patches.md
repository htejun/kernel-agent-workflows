# Applying patches

- **Load when:** the user asks to apply posted patches, one patch or a
  patchset from a mailing list or the user's inbox, to a tree they maintain,
  including the for-next merge, the push and the acknowledgment that
  follow.
- **Parameters:** `$TREES`, `$IDENTITY`, `$SCRATCH`, `$WORKTREE_ROOT`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [core/git-safety](../core/git-safety.md),
  [core/publishing](../core/publishing.md),
  [patches/commit-tags](../patches/commit-tags.md),
  [review/patchset-review](../review/patchset-review.md),
  [review/kernel-build](../review/kernel-build.md),
  [mail/lore-thread](../mail/lore-thread.md),
  [mail/mail-replies](../mail/mail-replies.md),
  [mail/mail-compose](../mail/mail-compose.md),
  [mail/thread-roots](../mail/thread-roots.md),
  [maintain/branch-model](../maintain/branch-model.md),
  [maintain/merges](../maintain/merges.md)

## Always-on rules
- Run `git am` or any other patch tool, in any tree, only when the user
  asked for that operation, and apply to a publication tree only on
  "apply" or an unambiguous equivalent such as "put it in for-X.Y-fixes".
  A posted patch waits for that word, because the user decides what enters
  a tree.
- [GATE] Before applying a patch: the thread was read, the patches were
  reviewed, no other tree took them, and the user approved the patches and
  the target branch, because undoing a wrong apply on a published branch
  takes a force push or a revert.
- [GATE] Before pushing applied patches: every commit message was read in
  full and has exactly one Signed-off-by from $IDENTITY, plus the author's
  on a contributor patch, and the build passed, because a missing
  committer sign-off on a public branch violates the DCO.
- Push the applied branches without a pause to confirm, because the apply
  instruction covers the push and the server rejects a push that doesn't
  fast-forward.
- Acknowledge every apply, the user's own patches included, under the gate
  of [core/publishing](../core/publishing.md#before-publishing), because
  the thread learns where the patches went only from the acknowledgment.

## [GATE] Checklist
### Before applying a patch
- [ ] The user asked for this apply ([always-on rules](#always-on-rules)).
      When unsure, ask.
- [ ] Every message of the thread was read, from people and bots alike
      ([mail/lore-thread](../mail/lore-thread.md#always-on-rules)), and the
      patches were reviewed
      ([review/patchset-review](../review/patchset-review.md#procedure)).
- [ ] No other tree took the patches
      ([cross-tree duplicates](#cross-tree-duplicates)).
- [ ] The user approved the presented patches, tags and target branch
      ([choosing the target branch](#choosing-the-target-branch)).

### Before pushing applied patches
- [ ] `git -C <tree> log --oneline -3 <target>` shows the applied subjects
      on the intended branch of the intended tree.
- [ ] Each commit's full message was read after its last amend. It has
      exactly one `Signed-off-by: $IDENTITY`, and a contributor patch still
      has its author's Signed-off-by.
- [ ] Each commit's author matches its first Signed-off-by, or the user
      was told about the mismatch
      ([Signed-off-by and author](#signed-off-by-and-author)).
- [ ] The build of the target branch passed
      ([review/kernel-build](../review/kernel-build.md#before-testing-posting-or-applying-a-code-change)).

## Procedure
### Applying
1. Fetch the whole thread ([mail/lore-thread](../mail/lore-thread.md#procedure))
   and summarize its reviews.
2. Review the patches
   ([review/patchset-review](../review/patchset-review.md#procedure)).
3. Check for duplicates in other trees
   ([cross-tree duplicates](#cross-tree-duplicates)). If another tree took
   a patch, stop and tell the user. Never apply it.
4. Choose the target branch
   ([choosing the target branch](#choosing-the-target-branch)).
5. Present the patch titles, the tags collected from the thread, a summary
   of the reviews and the target branch. Wait for the user's approval.
6. Make the changes the reviews call for, if the user wants them. A change too
   big to describe in the acknowledgment becomes a new version posted as a reply
   instead
   ([mail/mail-replies](../mail/mail-replies.md#acknowledgments-of-applied-patches)).
7. Save the mbox under $SCRATCH without carriage returns
   ([mail files](#mail-files)) and split it, for example with
   `git mailsplit`.
8. Apply one patch at a time: apply it, fix it up, then apply the next. Use
   `git am` for the user's own patches and `git am --signoff` for contributor
   patches ([Signed-off-by and author](#signed-off-by-and-author)). Fix typos
   and tags with `git commit --amend`. A conflict stops the work
   ([core/git-safety](../core/git-safety.md#when-a-conflict-appears)).
9. After each apply, check the author and the author date, and read the
   whole message with `git show --format=%B HEAD`. Never grep or awk the
   message in place of reading it.
10. Add the review tags from the thread with `git commit --amend`, in the order
    [patches/commit-tags](../patches/commit-tags.md#reference) gives, and follow
    the tree's policy on Fixes tags in selftest patches there. Read the whole
    message again after every fix-up.
11. Check the tree and the branch with `git -C <tree> log --oneline -3
    <target>`.
12. Build the target branch in the tree with the tree's own config
    ([review/kernel-build](../review/kernel-build.md#procedure)). Building a
    just-applied patch is part of publishing
    ([core/safe-operations](../core/safe-operations.md#publication-trees)).
13. On the for-X.Y preset, merge the target branch into for-next with a plain
    merge and git's default message, such as
    `git checkout for-next && git merge <target>`, after the cross-side
    check that
    [maintain/merges](../maintain/merges.md#a-conflict-on-for-next) calls
    for. If the check finds an adaptation or the merge conflicts, follow
    that section. A merge
    into the devel branch is a back-merge and needs the user's instruction
    ([core/git-safety](../core/git-safety.md#branch-structural-operations)).
    A tree on another preset updates the branches its $TREES entry names
    instead ([maintain/branch-model](../maintain/branch-model.md#presets)).
14. Verify the [GATE] checklist, then push the target branch and for-next:
    `git push <remote> <target> for-next`. Don't fetch and compare first.
    Stop only for a genuine error: a rejected push, a conflict, or an
    authentication prompt such as a second-factor token
    ([core/publishing](../core/publishing.md#second-factor-tokens)).
15. Prepare the acknowledgment right after the push
    ([the acknowledgment](#the-acknowledgment)), build it as
    [mail/mail-compose](../mail/mail-compose.md) describes, and send it as
    [core/publishing](../core/publishing.md#who-sends-mail) describes.
16. Remove one-shot worktrees
    ([removing a one-shot worktree](#removing-a-one-shot-worktree)).

### Removing a one-shot worktree
A worktree and local branch made under $WORKTREE_ROOT only to review, build
or test the patches of an apply are removed without asking, because a
leftover copy of a branch goes stale and later passes for the current one
([maintain/branch-model](../maintain/branch-model.md#pitfalls)). This
never covers a tree the user works in or the publication tree itself.
1. Remove them once the acknowledgment went out or was handed to the user,
   or right away when the review ends without an apply.
2. Check first that `git -C <worktree> status --short` prints nothing
   (ignored build output doesn't count), and that `git log
   <target>..<branch>` and `git diff <target> <branch>` show only the
   pre-fixup copies of the applied patches.
3. Run `git -C <tree> worktree remove <worktree>` without `--force`. It
   removes a worktree that holds only ignored files and refuses one with
   files that appeared after the check. Then run `git -C <tree> branch -D
   <branch>` and check that both are gone.
4. Unexpected changes, unique commits that weren't pushed, or a branch that
   exists on a remote stop the cleanup. Report them instead.
5. Notes and logs stay under $SCRATCH.

## Reference
### Cross-tree duplicates
A patch lands in exactly one tree, even when its subject prefix names
several subsystems. The prefix says what the patch touches, and another
tree that needs the patch gets it by back-merge. A duplicate is never
acceptable, not even a textually identical one.
- A patch from a series aimed at another tree is presumed taken there. The
  cover letter says the series is based on that tree's branch, or the
  series is addressed To that tree's maintainer.
- Search the mail for that tree's applied notification of this specific
  patch, such as the automated "+ <name>.patch added to <branch>" mail
  that a patch-queue tree sends.
- An ack posted on this subsystem's list doesn't route the patch through
  this tree.

### Choosing the target branch
Resolve branch names in the tree's own clone
([maintain/branch-model](../maintain/branch-model.md#finding-the-active-branch)).
Never assume a target. Verify it.
- With a Fixes tag, run `git -C <tree> branch --contains <fixed commit>`
  and apply where the fixed commit lives: the fixes branch when mainline
  has the commit, the devel branch when only that branch has it.
- Without a Fixes tag, the patch is development and goes to the devel
  branch.
- A patch that changes only selftests goes to the branch the tree's
  selftest policy in $TREES names, Fixes tag or not. Without a policy, ask.
- When the target branch lacks a commit the patch depends on, stop and follow
  [core/git-safety](../core/git-safety.md#when-an-apply-hits-a-missing-dependency).

### Signed-off-by and author
- A contributor patch gets the committer's Signed-off-by from
  `git am --signoff`. Without it, the applied commit violates the DCO.
- The user's own patch is applied with plain `git am` and keeps its single
  author Signed-off-by. `--signoff` would add a second one when review
  tags follow the first.
- When `git am` fails on a contributor patch and the commit is made by
  hand, use `git commit --author="Name <address>"` and write the
  committer's Signed-off-by into the message file. Then read the message
  and check that it is there.
- After `git am`, the commit's author must match the patch's first
  Signed-off-by. Mail relays often rewrite From. On a mismatch, tell the
  user.

### Mail files
- A mail saved with carriage returns once produced a commit whose author
  date was the time of the apply instead of the mail's Date. Strip
  carriage returns before applying, and after each apply check that the
  author date matches the Date header.
- Restore a wrong date with `git commit --amend --no-edit --date="<Date
  header>"` and check that `git show HEAD | grep -c $'\r'` prints 0.

### The acknowledgment
- It is prepared right after the push.
- It threads on the original posting
  ([mail/thread-roots](../mail/thread-roots.md#procedure)).
- Its recipients are the original posting's To and Cc plus everyone who
  replied in the thread, because it answers the whole thread while
  threading on the original. Nobody is dropped. The reviewer being
  addressed may go in To.
- Its wording, including how it reports changes made while applying, is in
  [mail/mail-replies](../mail/mail-replies.md#acknowledgments-of-applied-patches).

### A wrong apply
When a patch was applied without the instruction or to the wrong branch, report
the branch, its HEAD and whether it was pushed. Never reset it on your own: the
user chooses the undo
([core/git-safety](../core/git-safety.md#what-instructions-mean)).

## Pitfalls
- A patch from a series aimed at another tree was applied here too.
  Removing the duplicate took a force push of a published branch. Search
  for the other tree's notification before every apply.
- A contributor patch reached a published branch without the committer's
  Signed-off-by, and a reviewer on the list flagged the DCO violation. Read
  every message after every apply and amend.

## Customization
- `$TREES` sets each tree's remote, for-next branch and selftest policy.
- An overlay records per-contributor handling the user wants during an
  apply, such as extra checks, the user's own working trees that stay
  untouched after an apply, and the local mail search for applied
  notifications.
