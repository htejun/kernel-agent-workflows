# Dependent patchsets

- **Load when:** posting a patch or patchset that needs another one not yet
  in the target branch, posting a new version of one, or deciding which
  branch pending patches go on.
- **Parameters:** `$SEND_TOKEN`
- **Depends on:** [core/publishing](../core/publishing.md),
  [submit/patchset-posting](../submit/patchset-posting.md)

## Always-on rules
- [GATE] Before posting a dependent patchset: the prerequisite went out
  first and the dependent cover links it on lore, because reviewers can't
  apply or judge the dependent patchset without it.
- Keep pending and unposted patches on topic branches, never on a tree's
  devel or fixes branch, because those branches change only when reviewed
  patches are applied.

## [GATE] Checklist
### Before posting a dependent patchset
- [ ] The prerequisite patch or patchset went out first: sent by the agent
      under its own $SEND_TOKEN
      ([core/publishing](../core/publishing.md#before-publishing)), or sent
      by the user, who confirmed it went out.
- [ ] The prerequisite's Message-ID comes from its sent copy, or from the
      user when the user sent it, and the dependent cover links it as
      `https://lore.kernel.org/r/<msgid>`.

## Procedure
### Posting
1. Post the prerequisite first
   ([submit/patchset-posting](../submit/patchset-posting.md)) and wait until
   it has gone out. Then take its Message-ID.
2. Prepare the dependent patchset on top of the prerequisite. Its cover
   links the prerequisite by its lore URL, never by title alone, so that
   reviewers can find it.
3. Generate the patches as
   [submit/patchset-posting](../submit/patchset-posting.md#posting-a-patchset)
   does, with the target branch as `--base` and the prerequisite's tip as
   the start of the range. git format-patch then adds a
   `prerequisite-patch-id:` line for each prerequisite commit below the
   `base-commit:` line, so that tools can tell which patches must go in
   first:

   ```
   git format-patch <subject options> --base=<target base> -o <dir> <prerequisite tip>..HEAD
   ```

4. Post the dependent patchset.

### Reposting
1. Before posting a new version, check whether each prerequisite has
   merged into the base branch:

   ```
   git log --oneline -F --grep='<prerequisite subject>' <base>
   ```

2. If all of them have merged, base the new version on the base branch
   alone and drop the prerequisite links.

## Customization
- An overlay usually records the naming of topic branches in the user's
  trees.
