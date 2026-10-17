# Merges

- **Load when:** a merge between branches is about to be made or committed,
  a merge into for-next conflicts, or a merge commit message needs writing,
  including one for a back-merge.
- **Parameters:** `$IDENTITY`, `$WIDTHS`
- **Depends on:** [core/git-safety](../core/git-safety.md),
  [patches/line-widths](../patches/line-widths.md),
  [maintain/branch-model](../maintain/branch-model.md)

When a conflict stops the work and how to resolve one is in
[core/git-safety](../core/git-safety.md#when-a-conflict-appears). This
module covers conflicts on for-next, making a merge and merge messages.

## Always-on rules
- [GATE] Before committing a merge: the cross-side check ran, every
  adaptation it found was resolved from the commits on both sides and
  verified on every path, marked by git or not, and the user confirmed
  the message unless it is git's default one for for-next, because a
  resolution read from the 3-way diff alone can build and still drop a
  change.
- Never resolve a conflict on for-next. Abort, resolve in the devel branch
  and merge that into for-next, because for-next is rebuilt freely and must
  stay a plain union of the real branches.

## [GATE] Checklist
### Before committing a merge
- [ ] When both sides have new commits, the cross-side check ran
      ([core/git-safety](../core/git-safety.md#before-a-merge-where-both-sides-have-new-commits)),
      and the merged code keeps every commit's behavior on every path,
      including adaptations git merged without a marker and code the
      target moved to another file
      ([core/git-safety](../core/git-safety.md#resolving-a-conflict)).
- [ ] The resolution changes only what reconciles the two sides.
- [ ] A merge into for-next holds no conflict resolution
      ([a conflict on for-next](#a-conflict-on-for-next)).
- [ ] The target has commits the source lacks. Otherwise the target is
      fast-forwarded instead of merged ([making a merge](#making-a-merge)).
- [ ] The user confirmed the message. A for-next merge with git's default
      message needs no confirmation.

## Procedure
### A conflict on for-next
1. Before the merge of a branch into for-next, run the cross-side check,
   a clean merge included: a fixes branch meets the devel branch's commits
   first there. A merge that only pulls commits a checked merge already
   brought together, such as a devel branch whose fixes merge was checked,
   needs none. When the check finds an adaptation, don't make the merge.
   When the merge conflicts, run `git merge --abort` at once, without
   waiting to be asked. Either way for-next holds no resolution.
2. Present the merge that resolves the conflict in the devel branch
   instead. When a fixes branch conflicts with the devel branch, that is
   the fixes branch merged into the devel branch
   ([maintain/branch-model](../maintain/branch-model.md#branch-lifecycle)).
   That merge is a branch-structural operation
   ([core/git-safety](../core/git-safety.md#branch-structural-operations)).
3. On the user's go, make that merge, resolve there
   ([core/git-safety](../core/git-safety.md#resolving-a-conflict)), and
   commit with a confirmed message.
4. Merge the devel branch into for-next, which is then clean.

### Making a merge
1. If the target has no commits the source lacks (`git log TARGET ^SOURCE`
   prints nothing), fast-forward instead of making a merge commit with
   `--no-ff`.
2. Run the cross-side check
   ([core/git-safety](../core/git-safety.md#cross-checking-both-sides-of-a-merge))
   and present its result.
3. Merge without committing (`git merge --no-commit`) unless it is a
   for-next merge with git's default message.
4. Draft the message, with a Conflicts bullet for every adaptation the
   check found, present it and wait for the user's confirmation. The
   authorization for the merge doesn't approve its message.
5. Commit with `git commit -F <file>`.

## Reference
### Merge messages
- Before writing one, read the user's recent merge commits in the tree for
  its current conventions: `git log --all --merges --author=<name>`, and
  `git log --merges -i --grep=conflict` for the conflict notes.
- The subject is `Merge branch 'SOURCE' into TARGET`. Where the tree's
  recent merges carry the subsystem prefix, so does the subject:
  `subsys: Merge branch 'SOURCE' into TARGET`.
- The body names only the commits that matter to the merge's purpose: the
  dependencies it pulls and the commits that conflicted. Never list every
  incoming commit.
- Resolving conflicts is one of the merge's goals. State it as part of the
  purpose, never buried further down.
- A back-merge into a devel branch always carries a Signed-off-by with
  $IDENTITY. A routine back-merge with no conflicts may have an empty body
  and only the Signed-off-by.
- A for-next merge takes git's default message, with no body and no
  Signed-off-by.
- Fill prose and bullets to the patch description width in `$WIDTHS`
  ([patches/line-widths](../patches/line-widths.md#filling-prose)). Indented
  commit reference lines don't wrap. A quoted subject in prose may wrap.
- Use `[1]` and `[2]` references only when the text must refer back to the
  sides.

### Message shapes
A dependency pull names the commits it needs and why. Several dependencies
go in an indented list:

```
subsys: Merge branch 'SOURCE' into TARGET

Pull to receive:

 <sha12> ("subject")
 <sha12> ("subject")

for <upcoming work> and to resolve the conflicts with <target-side work> on
TARGET.

Conflicts:

- <function or file>: <sha12> ("subject") <does X>, while <sha12>
  ("subject") on TARGET <did Y>. <The resolution in one sentence.>

Signed-off-by: $IDENTITY
```

A single dependency goes inline, as
`Pull to receive <sha12> ("subject") as a dependency for <what needs it>.`
or, in the compact form,
`Pull to receive <sha12> ("subject") for <purpose>.`:

```
subsys: Merge branch 'SOURCE' into TARGET

Pull to receive <sha12> ("subject") as a dependency for <what needs it>.

Signed-off-by: $IDENTITY
```

A conflict note is the body of a routine merge that only documents its
conflicts. When no single commit stands for the target side, describe it in
prose, such as "the code reorganization and API updates on TARGET". Each
entry ends with the resolution: "Resolved by <action>.", "Include both." or
"Take changes from both."

The conflicts form a list:
- `Conflicts:` on its own line, a blank line, then one `- ` bullet per
  conflict in commit order, with continuation lines indented two spaces.
  To fill a bullet, fill it as a paragraph with a two-space prefix, then
  turn the first line's indent into `- `.
- Each bullet opens with the function or file, names the commits on both
  sides with `<sha12> ("subject")` and what each intended, then states how
  the resolution keeps each intention: "Kept both, the wait after the
  clear, so that ..." or "Dropped the old file's side and applied the
  rename at the new home." Every adaptation the cross-side check found
  gets a bullet, marked by git or not
  ([core/git-safety](../core/git-safety.md#before-a-merge-where-both-sides-have-new-commits)).

## Customization
- An overlay can record the tree's merge subject convention and example
  merge commits to follow.
