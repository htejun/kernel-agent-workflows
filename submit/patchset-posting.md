# Patchset posting

- **Load when:** preparing or posting a patchset or a new version of one,
  including RFC and resend postings, or collecting review tags from a
  previous posting.
- **Parameters:** `$SUBJECT_STYLE`, `$TREES`, `$SENT_LOG`, `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [core/git-safety](../core/git-safety.md),
  [patches/patch-descriptions](../patches/patch-descriptions.md),
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md),
  [patches/cover-letters](../patches/cover-letters.md),
  [patches/commit-tags](../patches/commit-tags.md),
  [review/checkpatch](../review/checkpatch.md),
  [review/patchset-review](../review/patchset-review.md),
  [review/kernel-build](../review/kernel-build.md),
  [mail/lore-thread](../mail/lore-thread.md),
  [mail/mail-send](../mail/mail-send.md),
  [mail/thread-roots](../mail/thread-roots.md),
  [mail/sent-log](../mail/sent-log.md),
  [submit/dependent-patchsets](../submit/dependent-patchsets.md)

## Always-on rules
- [GATE] Before posting a patch or patchset: checkpatch ran, the patchset
  review ran unless the user said "skip review", every description passed
  its checklist, and the recipients come from MAINTAINERS and $TREES,
  because a posted patch can't be recalled.

## [GATE] Checklist
### Before posting a patch or patchset
- [ ] checkpatch ran on every patch, as
      [review/checkpatch](../review/checkpatch.md#procedure) describes.
- [ ] The patchset review ran on the commits being posted, as
      [review/patchset-review](../review/patchset-review.md#running-a-milestone-review)
      describes, unless the user said "skip review".
- [ ] Every description passed the checklist in
      [patches/patch-descriptions](../patches/patch-descriptions.md#checking-a-draft),
      and every comment the patchset adds passed the one in
      [patches/comments-kerneldoc](../patches/comments-kerneldoc.md#checking-comments-in-a-patch).
- [ ] The recipients come from MAINTAINERS and $TREES
      ([recipients](#recipients)).
- [ ] The subjects follow $SUBJECT_STYLE ([prefixes](#subject-prefixes)),
      and one generated Subject was read.
- [ ] The send meets the gate in
      [mail/mail-send](../mail/mail-send.md#before-a-public-send), which covers
      threading, the mail files, the self-copy and the send token.

## Procedure
### Posting a patchset
1. Find the base with `git log --oneline --graph`. The cover names the
   target branch. The commits may sit on prerequisites above it
   ([submit/dependent-patchsets](../submit/dependent-patchsets.md)).
2. Build every stage, as
   [review/kernel-build](../review/kernel-build.md#procedure) describes.
3. Generate the patches into a new, empty directory, so that no file from
   an earlier run gets sent. `--base` records the base commit, so that
   reviewers and CI know what the patches apply to:

   ```
   git format-patch -v N --cover-letter --base=<base> -o <dir> <base>..HEAD  # standard, v2 on
   git format-patch -n --base=<base> -o <dir> <base>..HEAD                   # cover, never -v
   ```

4. Run checkpatch on the generated patches
   ([review/checkpatch](../review/checkpatch.md#procedure)) and the patchset
   review on the commits
   ([review/patchset-review](../review/patchset-review.md#running-a-milestone-review)).
5. Write the cover letter as
   [patches/cover-letters](../patches/cover-letters.md#procedure) describes,
   with the [subject prefix](#subject-prefixes) and the
   [recipients](#recipients). With `standard`, fill in the generated cover. With
   `cover`, write it as its own mail file in the same directory.
6. If the cover names a branch in the user's own public tree, push the
   branch there before asking for the send token, so that the link works
   when the mail arrives. A push to a shared remote is publishing and needs
   the send token
   ([mail/mail-send](../mail/mail-send.md#what-counts-as-publishing)). Try
   the push first, because an earlier second-factor code often still works.
   On an authentication error, get a new code as
   [mail/mail-send](../mail/mail-send.md#second-factor-tokens) describes,
   and never wait at an interactive prompt.
7. Send the cover and every patch in one git send-email invocation, as
   [mail/mail-send](../mail/mail-send.md#sending-patches-with-git-send-email)
   describes.
8. Record the posting in $SENT_LOG, as
   [mail/sent-log](../mail/sent-log.md#procedure) describes.

### Posting a new version
1. Look up the previous version in $SENT_LOG: its Message-ID, base, branch and
   patch count. Fetch its whole thread, as
   [mail/lore-thread](../mail/lore-thread.md#procedure) describes, to confirm
   what was sent. The patch count may differ from the local branch.
2. Collect the review tags and the feedback
   ([collecting review tags](#collecting-review-tags)).
3. If the previous version depended on other patches, check whether they
   have merged
   ([submit/dependent-patchsets](../submit/dependent-patchsets.md#reposting)).
4. Create the version's branch from the previous version's branch, and
   rebase it onto the base branch, named by its branch name, if the base
   moved:

   ```
   git branch <branch>-v2 <branch>
   git rebase <base> <branch>-v2
   ```

   A conflict in that rebase is handled as
   [core/git-safety](../core/git-safety.md#rebasing-the-agents-own-patches)
   describes. Never reset or reuse the previous version's branch, because
   the previous cover links it.
5. Make the changes, build
   ([review/kernel-build](../review/kernel-build.md#procedure)), and run the
   tests the subsystem uses.
6. Compare each patch with its previous version
   ([comparing versions](#comparing-versions)). Only real changes go into
   the vN notes and the cover's changelog.
7. Add a [vN note](#vn-notes) to every patch that changed.
8. Continue with step 3 of [posting a patchset](#posting-a-patchset). The cover
   reuses the previous version's body, with the changelog and the links to
   earlier versions on top, as
   [patches/cover-letters](../patches/cover-letters.md#procedure) describes. A
   full new version starts a new thread
   ([mail/thread-roots](../mail/thread-roots.md#procedure)), and its record goes
   under the patchset's existing entry in $SENT_LOG
   ([mail/sent-log](../mail/sent-log.md#procedure)).

### Collecting review tags
1. Read the previous version's whole thread for tags and feedback, as
   [mail/lore-thread](../mail/lore-thread.md#procedure) describes. To read
   the replies one by one, split the thread mbox into a directory, which
   must exist first:

   ```
   mkdir -p "$SCRATCH/<topic>/v2-mails" && git mailsplit -o"$SCRATCH/<topic>/v2-mails" <thread.mbox>
   ```

2. Add the tags to the new version's branch with `b4 trailers -u -F
   <msgid>`, where `<msgid>` is the previous version's cover. It rewrites
   the commits, so it waits for the commit gate in
   [core/git-safety](../core/git-safety.md#before-a-commit-or-amend).
3. Check the result against the replies: every Reviewed-by, Acked-by and
   Tested-by sits on the right patch, and none is missing. A reply's
   subject number, "Re: [PATCH 04/13]" for patch 4, counts in the previous
   version. Find that patch in the new branch by its subject, because
   patches added, dropped or moved since then change the numbers.
4. Order the tags and combine them with any Cc tag of the same person as
   [patches/commit-tags](../patches/commit-tags.md#reference) gives.

## Reference
### Subject prefixes
- With `$SUBJECT_STYLE` `standard`, `git format-patch -v N --cover-letter`
  marks the cover `[PATCH vN 0/M]` and each patch `[PATCH vN i/M]`. A tree
  that wants its name in the subject gets it through `--subject-prefix`:
  `--subject-prefix='PATCH net-next' -v 2` gives `[PATCH net-next v2 i/M]`.
- With `cover`, the version, the target `tree/branch` and any patchset
  marker go on the cover subject only, such as `[<MARKER> v2 tree/branch]`,
  where `<MARKER>` is the cover's leading word, `PATCH` unless an overlay
  names another. The patches come from `git format-patch -n` and keep
  `[PATCH i/M]`. A single patch is `[PATCH tree/branch]`.
- `--subject-prefix`, `--rfc` and `-v` are git format-patch options. git
  send-email ignores `--subject-prefix` without an error, and the patches
  go out without the marker. When a prefix is wrong, regenerate the
  patches.
- A patchset marker, such as RFC or a custom tag, goes on the cover subject
  only, edited by hand, and the patches keep `[PATCH i/M]`. When the user
  wants every patch marked as a request for comments, `--rfc` gives `[RFC
  PATCH i/M]`. Never use `--subject-prefix=MARKER` for a marker, because it
  gives `[MARKER i/M]` and drops PATCH.
- A resend of an unchanged patch or patchset puts RESEND after PATCH:
  `[PATCH RESEND]` and `[PATCH v2 RESEND]`, or with `cover`, `[PATCH RESEND
  tree/branch]` and `[<MARKER> RESEND tree/branch]`. Never `[RESEND
  PATCH]`. A changed repost is a new version.

### Versions
A version number marks a repost of a patch that has not been applied. Once
a patch is applied, a follow-up change is a new patch with a Fixes tag
pointing at the applied commit, and it carries no version.

### vN notes
- A patch that changed since its previous posting gets a note at the end of
  its description, above the whole tag block, with a blank line on each
  side:

  ```
  <description>

  v2: Drop the static from foo_scratch and declare it in foo.h (Name).

  Signed-off-by: ...
  Reviewed-by: ...
  ---
  ```

- The note is one line naming the mechanical change. It gives no
  rationale.
- Credit a suggestion or a report by name in parentheses, `(Name)`, never
  with markers such as `(R)` or `(P)`. Several changes take `v2:` on its own
  line and one `- <change> (Name).` line each.
- When a reviewer's report drove one change in a new version, not the whole
  patch, the credit is `(Name)` in the note, not Reported-by, which credits
  reporting the original problem. Add a trailing `Cc:` tag for the reviewer and
  copy them on the mail, so that they see the result. A build fix from the
  kernel test robot is credited as
  [patches/commit-tags](../patches/commit-tags.md#attribution-tags) describes.
- A patch with no changes gets no note, only its new tags. A patch new in
  this version gets no note either. The cover's changelog records it.
- The number counts the patch's own revisions, never the patchset's
  version, because a reviewer reads v3 as the third version of the patch
  and looks for a v2 change. A patch first changed for the v3 posting gets
  `v2:`. A patch changed for both v2 and v3 carries `v2:` and `v3:`. Notes
  run contiguous from v2, so a patch never carries v3 without v2. Before
  writing a note, find the patch's highest note in its current and
  previous versions and take the next number, or v2 when it has none.
- Several notes in one patch run ascending, `v2:` then `v3:`, one
  paragraph each. Only the cover lists versions descending.
- Some maintainers want version notes below the `---` line, where they show
  in the mail and stay out of the commit. Check the target tree's
  convention before placing a note, and ask when unsure.

### Comparing versions
`git range-diff` pairs each patch of the new version with its previous
version and shows the difference between the two:

```
git range-diff <old-base>..<prev> <base>..<new>
```

`=` marks an unchanged patch, and `!` a changed one, with the difference
below it. `<` and `>` mark patches found only in the previous or only in
the new version. A heavily changed patch can show up as such a pair, and
a higher `--creation-factor` pairs it again. A change in context lines
alone comes from a moved base and is not a change to the patch.

### Recipients
- The MAINTAINERS entries for the files a patchset touches name its
  maintainers, reviewers and lists. `scripts/get_maintainer.pl` collects
  them, or the entries can be read directly. Take addresses from
  MAINTAINERS, because people change them.
- Add the lists and people that $TREES names for the tree.
- A new version goes to the previous version's actual recipients, taken
  from its sent copy, plus everyone who reviewed it and everyone a vN note
  credits.

## Customization
- `$SUBJECT_STYLE` picks the subject form, and `$TREES` the lists and
  people each tree copies.
- An overlay usually names the cover's leading word for `cover`, the trees
  that want vN notes below `---`, whether recipients come from
  get_maintainer.pl or from reading MAINTAINERS, and the public tree's push
  details.
