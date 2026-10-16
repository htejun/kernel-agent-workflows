# Cover letters

- **Load when:** writing or revising the cover letter of a patchset,
  including the changelog of a new version.
- **Parameters:** `$PLAN_DIR`, `$SENT_LOG`
- **Depends on:** [patches/patch-descriptions](../patches/patch-descriptions.md)

## Always-on rules
- Nothing local goes into a cover letter or anything else an outside
  reader sees: machine names and sizes, local config options, the names of
  local test suites, tests held back from the posting, or what ran on which
  machine, because the reader can't use any of it.

## Procedure
### Writing a cover letter
1. For a new version, v2 and later, open with the changelog: a
   `vN (<summary>):` line followed by bullets of what changed in this
   version ([Changelog](#changelog)).
2. Below the changelog, link every earlier version, one per line, newest
   first, each as `vN: https://lore.kernel.org/r/<message-id>`. Changelog
   first, links after, and never "vN of <link>" phrasing.
3. Write the body like a description
   ([patches/patch-descriptions](../patches/patch-descriptions.md#what-a-description-says)):
   the problem, the key pieces of the solution, the testing summary
   ([Testing summary](#testing-summary)) and the base.
4. List the patches, one ` NNNN <full subject>` line each:

   ```
   git log --reverse --format='%s' BASE..HEAD | awk '{printf " %04d %s\n", NR, $0}'
   ```

5. Say how the patches are organized, as bullets keyed by index ranges.
6. Give the public git branch the patchset is available in, if there is
   one.
7. End with the diffstat from `git diff --stat BASE..HEAD`.

### Updating it for a new version
1. Start from the previous version's cover letter. Keep its body
   unchanged.
2. Write the changelog for this version above the earlier version links,
   and add the link to the previous version at the top of the links.
3. Update the patch list and the patch count, the branch, and the
   diffstat. Nothing else changes.

## Reference
### Layout
```
vN (<summary>):
- <material change> (#2).
- <material change> (#4, #5).

v<N-1>: https://lore.kernel.org/r/<message-id>
v1: https://lore.kernel.org/r/<message-id>

<the problem>

<the key pieces of the solution>

<testing summary>

Based on <tree>/<branch> (<sha12>).

<patch list, one " NNNN <subject>" line per patch>

<how the patches are organized, as bullets keyed by index ranges>

<git URL> <branch>

<git diff --stat output>
```

A first posting has no changelog and no version links. The cover that
`git format-patch --cover-letter` generates already has a shortlog and a
diffstat, which take the place of the patch list and the diffstat above.

### Changelog
- Material changes only: code, logic and documentation, not review tags.
- High level, with patch numbers and attribution to whoever asked for the
  change: "Fix the exit race in #14 (Jane).", not a detailed enumeration.
- The per-patch version notes in the patches themselves follow
  [submit/patchset-posting](../submit/patchset-posting.md#vn-notes).

### Testing summary
- A few sentences at the level of conclusions: the kinds of tests that
  ran, that race windows were widened with a debug patch, and the
  bottom-line claim.
- No environment variable values, sleep durations, knob names, per-test
  walkthroughs or counter values, unless the magnitude is the claim.
- State coverage in terms the reader can use, such as "the selftest
  exercises every transition across the boundary". When the testing has
  nothing to say at that level, drop the summary.
- Model checking gets one clause at an outside reader's level, such as "the
  enable and disable handshake was model checked for small CPU counts",
  when it helps a reviewer
  ([review/formal-model-check](../review/formal-model-check.md)).
- The local record of what ran where belongs in the plan notes under
  $PLAN_DIR and in the $SENT_LOG entry.

## Pitfalls
- A cover letter once carried the local test record: machine names, the
  names of local suites and selftests that were held back. An outside
  reader can use none of it. The record belongs in the plan notes and the
  sent log.

## Customization
- An overlay sets the user's greeting, signature, fixed phrases and git
  branch URL.
