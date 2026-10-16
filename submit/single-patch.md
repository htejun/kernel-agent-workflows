# Single patch posting

- **Load when:** posting one patch on its own, re-posting one patch of a
  posted patchset as a reply, or posting a patch that a tree has already
  applied.
- **Parameters:** `$SUBJECT_STYLE`, `$SENT_LOG`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [patches/commit-tags](../patches/commit-tags.md),
  [mail/lore-thread](../mail/lore-thread.md),
  [mail/mail-send](../mail/mail-send.md),
  [mail/thread-roots](../mail/thread-roots.md),
  [mail/sent-log](../mail/sent-log.md),
  [submit/patchset-posting](../submit/patchset-posting.md)

A single patch goes out without a cover letter. The posting gate and the
recipients are in [submit/patchset-posting](../submit/patchset-posting.md).

## Procedure
### Posting one patch
1. Check the commit: its author, subject and Signed-off-by, and its tags as
   [patches/commit-tags](../patches/commit-tags.md) describes.
2. Check that the tree has no uncommitted changes in the files the patch
   touches. If it has, ask the user what to do, because they may be edits
   meant for the patch.
3. Decide whether the patch is a new version or a new patch
   ([versions](../submit/patchset-posting.md#versions)).
4. Generate it with `git format-patch -1 -o <dir> <commit>`. With
   `$SUBJECT_STYLE` `standard`, add `-v N` for a new version. With `cover`,
   add `--subject-prefix='PATCH tree/branch'`, with the version after PATCH
   for a new version.
5. If a tree has already applied the patch, add `Applied to tree/branch.`
   on its own line right after the `---` line, before the diffstat. Text
   there shows in the mail and stays out of the commit.
6. Pass the posting
   [gate](../submit/patchset-posting.md#before-posting-a-patch-or-patchset).
7. Send the patch as
   [mail/mail-send](../mail/mail-send.md#sending-patches-with-git-send-email)
   describes, on the thread root that
   [mail/thread-roots](../mail/thread-roots.md#procedure) gives, and record it
   in $SENT_LOG as [mail/sent-log](../mail/sent-log.md#procedure) describes.

### Re-posting one patch of a patchset
Send an updated patch of a posted patchset, such as one with review fixes
or a bug fix, as a reply to the patch it replaces.
1. Fetch the patchset's thread, as
   [mail/lore-thread](../mail/lore-thread.md#procedure) describes. Take the
   patch's Message-ID from the mbox with a mail parser, such as Python's email
   module, never from memory
   ([core/evidence-discipline](../core/evidence-discipline.md)).
2. Collect the recipients: every To and Cc of the original patch and of
   the replies in its thread, plus anyone the change credits. Drop nobody.
3. Add a [vN note](../submit/patchset-posting.md#vn-notes) for the change.
4. Generate the patch with the original subject, changing only `PATCH` to
   `PATCH vN`. The `NN/MM` and the rest of the subject stay character for
   character, and no tree tag is added. Thread it as
   [mail/thread-roots](../mail/thread-roots.md#procedure) gives for a patch
   re-posted from a patchset:

   ```
   git format-patch -1 --subject-prefix='PATCH v2 04/13' --in-reply-to='<msgid>' -o <dir> <commit>
   ```

5. Pass the posting
   [gate](../submit/patchset-posting.md#before-posting-a-patch-or-patchset) and
   send the patch as
   [mail/mail-send](../mail/mail-send.md#sending-patches-with-git-send-email)
   describes.
