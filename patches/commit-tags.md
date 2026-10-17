# Commit tags

- **Load when:** writing or checking the tag block of a commit, choosing
  the commit a Fixes: tag names, deciding on a stable Cc, crediting people
  in tags, or referring to a commit in prose.
- **Parameters:** `$TREES`
- **Depends on:** [core/git-safety](../core/git-safety.md),
  [maintain/apply-patches](../maintain/apply-patches.md),
  [mail/mail-replies](../mail/mail-replies.md)

## Always-on rules
- Refer to a commit in prose as `<sha12> ("subject")`
  ([form](#commit-references)), never by a bare ID, because the subject
  tells the reader what the commit is and lets them check the ID.

## Procedure
### Finding the commit a fix fixes
1. Name the buggy lines and what is wrong, missing or misplaced in them.
2. Run `git blame -L <start>,<end> <file>` on those lines. Follow the lines
   through renames and moves as needed. Never take the commit from
   `git log -S` alone, because -S finds the last commit that touched a
   string, not the one that introduced the bug.
3. Read the diff of the blamed commit and confirm that the bug existed
   there. Blame often stops at a refactor or a move. Walk further back to
   the commit that introduced the pattern, because the tag names the
   introduction, not a later refactor.
4. A bug that needs several conditions at once, such as a missing unlock,
   put or check, exists from the moment all of them coexist. Tag the latest
   commit that completed the combination: the one whose revert removes the
   bug.
5. When those commits came in on parallel branches, "latest" means merge
   topology, not commit dates. Tag the commit that landed last in the
   integration branch where the bug becomes visible, usually mainline. In
   bash:

   ```bash
   # both commits are in the integration branch
   git merge-base --is-ancestor <A> <integration> && git merge-base --is-ancestor <B> <integration>
   # the merge that brought A into the integration branch's first-parent history
   git rev-list --first-parent <integration> |
           awk 'NR == FNR { d[$1]; next } $1 in d' \
               <(git rev-list --ancestry-path <A>..<integration>) - | tail -1
   # the same for B, then: true means B's merge descends from A's, so B landed last
   git merge-base --is-ancestor <merge of A> <merge of B>
   ```

6. Check the answer: reverting the tagged commit must restore correct
   behavior.
7. Write the tag as `Fixes: <sha12> ("subject")`.

### Deciding on a stable Cc
1. Decide whether the fix needs a backport to stable kernels.
2. Find the first release that contains the fixed commit, using the tags
   from mainline:

   ```
   git tag --contains <sha> | grep -E '^v[0-9]+\.[0-9]+$' | sort -V | head -1
   ```

3. No output means that the fixed commit is only in an rc, a devel branch
   or a fixes branch. Add no stable Cc, because stable has no tree to
   backport the fix to when no release contains the bug. The fix goes into
   the coming release through the normal flow.
4. Otherwise add `Cc: stable@vger.kernel.org # vX.Y+`, with the release
   from step 2, to the tag block. It is a tag in the commit message only.
   Don't add the stable list to the mail's recipients by hand.
5. Earlier stable tags of your own show the form your trees use:
   `git log --all --author=<your address> --grep='Cc: stable' --format='%h %s'`.

## Reference
### Tag order
Tags form one block after the description, in this order:

1. `Fixes:`
2. `Cc: stable@vger.kernel.org`
3. `Reported-by:`, each followed directly by the `Closes:` or `Link:` of
   its report, then `Suggested-by:` and other `Link:` tags
4. `Signed-off-by:`, the author's first
5. `Reviewed-by:`, `Acked-by:` and `Tested-by:`
6. `Cc:` for other people

Each `Co-developed-by:` is followed directly by its co-author's
`Signed-off-by:`, among the sign-offs. Review and ack tags go after the
Signed-off-by, never before it. A Fixes: tag goes in the block, never in
the body of the description. checkpatch warns when a Reported-by: isn't
followed by a Closes: or a Link:.

### Link tags
Fetch every Link: URL and confirm that it shows the intended message
before the commit, because a mistyped Message-ID makes a dead link. Prefer
`https://lore.kernel.org/all/<message-id>/`, because a `/lkml/` URL
returns 404 for a message that is only in another list's archive.

### Combined tags
Combine one person's tags. Reported-by: and Tested-by: from one person
become `Reported-and-tested-by:`. A Cc: for a person who also gave a
Reviewed-by: or Acked-by: is dropped, because the review tag already names
them. Match the person by name, not by address, because one person may
review from a different address than the one in the Cc.

### Attribution tags
- `Reported-by:` names the person who reported the bug. For an
  AI-generated report, that is the person who sent it, not the tool. Copy
  the name and address from the reporter's recent commits or mail, because
  a current signature can differ from older ones.
- `Original-patch-by:` credits someone whose diff this specific patch
  derives from. `Co-developed-by:` with their `Signed-off-by:` is for
  shared authorship, and `Suggested-by:` for an idea without a diff.
- `Cc:` is for someone who should see the work but didn't contribute to
  this patch.
- Re-evaluate the tags of every patch when splitting a larger patch, and
  whenever the approach changes. A fix that takes a different approach
  from the diff it started from drops the tags that credited that diff,
  and the diff's author becomes a Reported-by: beside the original
  reporter.
- A small build fix reported by the kernel test robot and folded into a
  larger patch is credited in that patch's vN changelog line, not with
  Reported-by: and Closes:.

### Selftest patches
Whether selftest patches carry Fixes: tags is a policy of the user or the
tree, recorded in the overlay. Under a policy that drops them, never add
one to the user's own selftest patch. When applying a contributor's
selftest patch, strip the tag with `git commit --amend` and note the drop
in the acknowledgment
([mail/mail-replies](../mail/mail-replies.md#acknowledgments-of-applied-patches)).
Which branch a selftest patch goes to is the tree's selftest policy in
`$TREES`
([maintain/apply-patches](../maintain/apply-patches.md#choosing-the-target-branch)).

### Commit references
- The form is `<sha12> ("subject")`: 12 hex digits of the ID, then the full
  subject in quotes and parentheses, with no "commit" in front. This
  command prints it:

  ```
  git log -1 --abbrev=12 --format='%h ("%s")' <commit>
  ```

- checkpatch's GIT_COMMIT_ID check expects "commit" in front and flags this
  form ([review/checkpatch](../review/checkpatch.md#ignored-warnings)).

- A second mention repeats the form or rephrases, such as "that commit".
  It is never a bare ID.

### Fixing a tag typo
Fix a typo in the tags of the newest commit with `git commit --amend`. In
an older commit, rewrite the messages from that commit on:

```
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch --msg-filter '<sed command>' -- <commit>^..HEAD
```

Without the range, filter-branch rewrites every commit reachable from
HEAD. Without the variable, it prints a warning and waits before it
starts. Both forms rewrite history, so the gates of
[core/git-safety](../core/git-safety.md#before-rewriting-a-branch) apply.

## Pitfalls
- `git log -S` returned the last commit that touched the string for a bug
  in which a lock is taken and a callee was later made fallible without
  updating the error path. It missed the commit that completed the bug.
  Use blame and the steps above.
- `git log --ancestry-path <A>..<integration> | tail -1` returns the next
  commit on A's own branch, not the merge that brought A in. Filter by the
  first-parent history as in step 5.
- `git tag --contains` also lists rc tags. A commit that is only in an rc
  is not in a release, so filter for release tags as in the stable steps.

## Customization
- The overlay records the user's tag policies, such as Fixes: tags on
  selftest patches or a different tag order.
- For trees that follow Documentation/process/maintainer-tip.rst, an
  overlay keeps each tag on its own line instead of combining them, because
  combined tags get in the way of automated tag extraction.
- checkpatch reports Original-patch-by: as non-standard. For trees that
  want only standard tags, an overlay credits a derived diff with
  Co-developed-by: or Suggested-by: instead.
- For trees that want submitting-patches.rst's form, an overlay writes
  references as `commit <sha12> ("subject")`.
