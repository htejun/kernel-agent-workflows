# Pull requests to mainline

- **Load when:** preparing or sending a pull request from a tree the user
  maintains to mainline, for fixes during the rc cycle or for the merge
  window, for one tree or several at once.
- **Parameters:** `$TREES`, `$SCRATCH`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md),
  [core/git-safety](../core/git-safety.md),
  [review/kernel-build](../review/kernel-build.md),
  [core/publishing](../core/publishing.md),
  [mail/mail-compose](../mail/mail-compose.md),
  [maintain/branch-model](../maintain/branch-model.md),
  [maintain/merges](../maintain/merges.md)

## Always-on rules
- [GATE] Before creating a pull request tag: the base, the commits, the
  test-merge build, the names and the diffstat were verified, because a
  published tag can never be moved or deleted.
- [GATE] Before sending a pull request: the tag is pushed,
  `git request-pull` exits 0, and the mail body is its output verbatim,
  because mainline pulls exactly what the public tag holds, and a failing
  request-pull means the tag there doesn't match.

## [GATE] Checklist
### Before creating a pull request tag
- [ ] The base branch was identified and checked with
      `git log --oneline -3 <branch>`.
- [ ] Every intended patch is on the branch, checked with `git rev-list`.
- [ ] A clean build of the test-merge branch succeeded
      ([review/kernel-build](../review/kernel-build.md#procedure)).
- [ ] The tag name, the version in the tag description and the base rc
      match ([tag names and subjects](#tag-names-and-subjects)).
- [ ] The test-merge diffstat and commit list came from git
      ([test merge and build](#test-merge-and-build)), never written by
      hand.

### Before sending a pull request
- [ ] `git request-pull` exits 0 for the pushed tag.
- [ ] The mail body is its standard output, verbatim, and the diffstat and
      shortlog in it match the test-merge cross-check.
- [ ] The recipients include the mainline maintainer,
      linux-kernel@vger.kernel.org, the subsystem lists and the MAINTAINERS
      entries for the tree, as $TREES lists them.

## Procedure
### Finding the base
1. Ask the user whether this is a fixes or a merge-window pull request.
2. Update master from mainline
   ([maintain/branch-model](../maintain/branch-model.md#checking-for-pending-fixes)).
3. Find the current rc with
   `git log --oneline master | grep -m1 '^[0-9a-f]* Linux '`.
   `git describe master` can show an older rc when tags weren't fetched.
4. Name the tag and subject after the rc the pull request is based on,
   never the next one ([tag names and subjects](#tag-names-and-subjects)).
   Compare with the tree's previous fixes tag:
   `git tag -l '*fixes*' --sort=-creatordate | head`, then
   `git show <tag> --no-patch`.
5. Find the dependency merges with
   `git log --oneline --merges <branch> ^master` and identify the external
   branch each one merged. Ask the user whether to include each in the
   base and where to fetch it from.
6. With dependencies, build the base: `git checkout -b pr-base-for-X.Y
   master`, then `git fetch <remote> <branch> && git merge FETCH_HEAD` for
   each dependency.
7. Check that `git rev-list <branch> ^pr-base-for-X.Y --no-merges` lists
   exactly the expected patches.

### Test merge and build
Mainline pulls dependency trees separately, so the pull request covers only
what this tree adds on top of them.
1. For each dependency merge, take the merged commit, the merge's second
   parent.
2. `git checkout -b test-merge-<branch> master`, then `git merge <dep-sha>`
   for each dependency, then `git merge <branch>`. Any conflict stops the
   work: present it and wait
   ([core/git-safety](../core/git-safety.md#when-a-conflict-appears)).
3. A conflict between a dependency and master belongs to the dependency
   tree's pull request and stays out of this mail. A conflict from merging
   the pull request branch goes in the mail with how to resolve it, in the
   conflict form of a merge message
   ([maintain/merges](../maintain/merges.md#message-shapes)). Without one,
   the mail says nothing about conflicts.
4. Build the test-merge branch, not the pull request branch
   ([review/kernel-build](../review/kernel-build.md#procedure)). The test
   merge and its build are part of preparing the pull request
   ([core/safe-operations](../core/safe-operations.md#publication-trees)).
5. Generate the cross-check: `git diff --stat <base>..test-merge-<branch>`,
   where the base is the point after the dependency merges, never bare
   master, and `git rev-list <branch> --not master <dep-sha-1>
   <dep-sha-2>`, which leaves out the dependencies' commits. The diffstat
   and shortlog that request-pull puts in the mail must match them, so
   changes from dependency trees never appear in the pull request.

### Writing the tag description
1. Read every commit body, not only the subjects.
2. Write the description ([tag descriptions](#tag-descriptions)) to a file
   under $SCRATCH.
3. Before sending, check with `git log master..<tag>` that every commit is
   reflected.

### Creating and sending
Signing and pushing the tag are preparation, done before asking for the
send word ([core/publishing](../core/publishing.md#always-on-rules)), because
the mail body is request-pull output for the pushed tag.
1. The user signs the tag, since signing needs the user's key:
   `GPG_TTY=$(tty) git tag -s <tag> <branch> -F <description file>`.
2. Push the tag: `git push <remote> <tag>`.
3. Run `git request-pull master <public URL> <tag>`, or with
   `pr-base-for-X.Y` in place of master when the base has dependencies. It
   exits non-zero until the tag reaches the public servers. Run it again
   every 5 seconds for up to 2 minutes
   ([core/waits-and-long-ops](../core/waits-and-long-ops.md#procedure)),
   and save its standard output to a file under $SCRATCH.
4. Build the mail around the saved output: the headers with
   `Subject: [GIT PULL] <subject>`, a greeting, the output verbatim, the
   signature. Diff the embedded part against the saved output.
5. When the base has dependencies, add a note above the request-pull output
   that says what the base consists of, such as "vX.Y + <tree>/<branch>",
   and how to resolve any conflicts.
6. Send it as [core/publishing](../core/publishing.md#who-sends-mail)
   describes, and record it
   ([mailbox/sent-log](../mailbox/sent-log.md#procedure)) when that is
   enabled.

### Several trees at once
1. Update mainline once, then fetch its master into each tree.
2. Run each tree's steps in that tree's own shell calls
   ([core/safe-operations](../core/safe-operations.md#working-across-repositories)).
3. A second-factor token may cover several tag pushes
   ([core/publishing](../core/publishing.md#second-factor-tokens)). The
   overlay says how the user's server behaves.

## Reference
### Tag names and subjects
- Fixes: tag `<prefix>-for-X.Y-rcN-fixes` from the fixes branch
  for-X.Y-fixes, which has no rc in its name. Subject
  `<subsystem>: Fixes for vX.Y-rcN`. With master at vX.Y-rc2, the tag is
  `<prefix>-for-X.Y-rc2-fixes`, not rc3.
- Merge window: while mainline is at X.Y or one of its rcs, tag
  `<prefix>-for-X.(Y+1)` from the devel branch for-X.(Y+1). Subject
  `<subsystem>: Changes for vX.(Y+1)`.
- Published tags are never deleted or moved
  ([core/git-safety](../core/git-safety.md#forcing-and-overwriting)). A
  second pull request for the same rc gets a numbered suffix, such as
  `<prefix>-for-X.Y-rcN-fixes-2`. Its tag description keeps "Fixes for
  vX.Y-rcN", and only the mail subject says "Another fix for vX.Y-rcN".

### Tag descriptions
The mainline maintainer reads hundreds of pull requests and doesn't know a
subsystem's function names, types or jargon. Describe themes in plain,
kernel-generic terms, such as "Improve recovery from misbehaving drivers",
never a list of the patches or the files they touch.
- Three to six bullets, with related patches grouped by kind of bug, not by
  file or area: use-after-frees, locking under recursion or NMI, lifecycle
  bugs in new infrastructure, verifier-time hardening, cross-subsystem
  dependencies, smaller one-liners. A bucket named after a feature ("X
  fallout") hides that the bugs weren't sorted by kind.
- Each bullet takes one to three lines and leads with the bug and its
  visible effect. The fix is a clause, and only when it isn't the obvious
  inverse of the bug: "A race could free an object while deferred work was
  still in flight, closed by flushing the work before the free." No
  mechanism or trigger detail.
- Bullets go in decreasing order of importance, and the less important an
  item, the shorter it is.
- Replace function, type and symbol names with what they do, such as "the
  destroy helper" or "the callback invocations". Keep a symbol only when it
  is the news, such as a new option, type or config.
- Changes to other subsystems pulled through this tree, such as lib, fs or
  mm changes, get their own bullet. The mainline maintainer wants to see
  them.
- No "Misc" bucket. Fold each item into the right bucket or leave it as a
  bare one-liner at the bottom.
- Patches applied late for a specific reason, such as a stable backport,
  are mentioned separately.
- No credits for reviewers or reporters. The commits' tags carry them.
- A header sentence orients the reader when the batch is unusually large
  or crosses subsystems. The bullets deliver what it frames.

## Customization
- `$TREES` sets each tree's tag prefix, the subsystem name in subjects, the
  public URL for request-pull, the remote to push tags to, and the
  recipients of its pull requests.
- An overlay records the greeting and signature of the mail, how the
  user's server handles second-factor tokens, and the dependency trees the
  user's branches usually merge.
