# Patch descriptions

- **Load when:** writing, revising or reviewing a commit message or patch
  description, including its subject line.
- **Parameters:** none
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [patches/commit-tags](../patches/commit-tags.md)

## Always-on rules
- A description says why, not what: who needs the change and why, what it
  guarantees, and what a reviewer needs to check it, because the diff
  already shows what changed.
- A description stands alone in the git history: no names of sibling
  patches and no "this series", because the patchset around it is gone
  once it is merged.
- [GATE] Before committing a patch: the subject follows the tree's
  convention, the description doesn't narrate the diff, and neither the
  subject nor the body uses kerneldoc `%` markup, because a maintainer has
  to fix each by hand or send the patch back.

## [GATE] Checklist
### Before committing a patch
- [ ] The subject has the prefix form and the capitalization after the
      prefix that recent commits touching the same files use
      (`git log --oneline -20 -- <files>`). A nested topic prefix, such as
      a tool name, keeps its own case.
- [ ] The description doesn't narrate the diff
      ([What stays out](#what-stays-out)).
- [ ] The subject and the body contain no kerneldoc `%` markup such as
      `%NULL`. Kerneldoc comments in the code keep it.

## Procedure
### Writing a description
1. Write the description last, once the code is final, from what a
   reviewer of the diff needs, not from what went into writing the code.
   Read a few recent commits in the same tree and the patchset's earlier
   patches first, and match their subject form, voice, style and length.
2. Open with the rationale: who needs the change and why, concretely, then
   what goes wrong without it
   ([What a description says](#what-a-description-says)).
3. Describe the change as properties a reviewer can check. Then name the
   subtle points and say why each is right.
4. Leave out everything in [What stays out](#what-stays-out).
5. Match the length to the kind of patch
   ([By kind of patch](#by-kind-of-patch)).
6. Check the draft with the steps below before showing it.

### Checking a draft
1. The first one or two sentences say who wants the change and what problem
   it solves. "Add X that does Y" describes code, not a problem.
2. Every sentence is the rationale, the problem, the change as properties,
   or a correctness argument the reviewer needs, and tells the reviewer
   something the diff and its kerneldoc don't. The subtle points are named
   as such.
3. Every identifier was checked against the source
   ([core/evidence-discipline](../core/evidence-discipline.md#always-on-rules)),
   and no sibling patch is named.
4. The length matches the kind of patch.

### Revising on request
1. When the user asks for a shorter description, cut from
   [What stays out](#what-stays-out), never the substance. If the draft
   already holds only content, ask which point the user thinks is
   unnecessary.

## Reference
### What a description says
The reader is a reviewer looking at the diff. A description gives the
problem, the change at the level of what it guarantees, and the correctness
argument the reviewer needs.

- One story, each paragraph one stage: who wants the change and why, what
  stands in the way, how the patch resolves it, and the limitations it
  leaves. Incidental extras go last, in their own paragraph.
- Rationale first, concretely: name the consumer of the change and what it
  wants, then what goes wrong without the change. A defect statement is not
  a rationale, and "should be able to" is not a want.
- The problem is the missing capability, not one consumer's symptom: "a
  parent has no way of telling how much each child is using", not "a
  parent sees a busy child as idle". The symptoms follow as consequences,
  if at all.
- Generic infrastructure leads with the general capability and the class
  of users, cites the existing precedent it follows, and brings in the
  driving consumer as an example.
- An addition is stated as the addition, not as the absence it fills:
  "Add a per-CPU count of dropped events to debugfs", not "Nothing records
  how many events are dropped, so add a per-CPU count". Keep the problem
  sentence only when the reader wouldn't otherwise see the need.
- A patch that replaces a mechanism first says what is wrong with it as a
  design, such as an unusual dependency that is brittle and gets in the way
  of future changes, and only then the change it would break. The breaking
  change alone reads as one consumer's symptom.
- The mechanism as properties, not procedure: what it costs on the hot
  path, who gets notified, what gates it, the invariant a teardown checks.
  Never the sequence of calls that produces them, which is the code in
  English and goes stale with the code. Field names, call sites,
  thresholds, half-lives and tuning math belong in comments and kerneldoc.
- A non-trivial refactor gives the insight that lets a reviewer verify the
  equivalence: a non-obvious mapping, transposed checks, a replaced
  representation. "Equivalent conversion" alone is not enough. Call out any
  tightening or loosening of a validation rule.
- Explain the design tradeoffs, why the simpler version fails, and any
  non-obvious invariant, naming, layout or API shape.
- Every sentence earns its place. Of two descriptions that say the same,
  the shorter one is the one to send.
- Point the reader at what is subtle instead of listing the change. Name
  the non-obvious points as such and say why each is right. Leave every
  behavior the code shows to the code and its comments. A list of
  everything the patch does, in prose or as bullets, is too long to read
  and hides what matters. Bullets are for items of one kind that a reader
  has to find, never for walking through the diff.

### What stays out
- Diff narration: API walkthroughs ("X_init() does foo, X_next() does
  bar"), algorithm narration, code walks that name each function and branch
  an event passes through, quoted macro call forms, a paraphrase of
  kerneldoc the diff adds, lists of renamed identifiers, enumerated hunks
  and field-by-field translations. "An invalid request is caught and
  discarded later in the submission path" carries the content. Name a
  function only when the name is the point.
- Reassurance and defense: what still works after the fix, what is
  unchanged unless the reader would otherwise wonder, leak audits for
  removed code when ownership is evident, and correctness arguments and
  locking or lifetime proofs that belong at the code site. "The existing
  race in Z is unchanged and needs a separate fix" stays. "All other
  callers are unchanged" goes.
- Paths not taken. Never say what the change does not do, unless an
  elaborate design choice needs that rationale.
- A failure mode nobody stated. When the user says the problem is X, write
  X.
- Severity editorializing.
- Reproducer details. They belong in the cover letter.
- The patch's position in a patchset.
- Reviewer credit in the body. The tags give it.
- Precedent from other subsystems named in passing.
- Asides that teach the subsystem.
- How a pointer was obtained. Every way a race can happen, when one phrase
  covers them all.
- Sibling patch names, "this series" and "this patchset". Say "a later
  patch" or "an earlier patch". A loop pass is named by its work, such as
  "the init pass", never "iteration 2".
- Model checking. A description never mentions the model. A cover letter
  may ([patches/cover-letters](../patches/cover-letters.md#testing-summary)).
- A mechanism detail a reviewer flags as inaccurate. Drop it rather than
  make it more accurate, unless the reader needs it to understand the fix:
  the flag shows that the sentence wasn't load-bearing.
- Equivalence details after "No functional change". The statement covers
  them.

### By kind of patch
- Trivial fix, such as a typo, a wrong size or a single flag: the bug, the
  cause and the fix.
- Fix for a reported bug: the bug, the fix and any collateral change that
  isn't obvious, such as a reworked condition or a renamed parameter. The
  length follows the complexity of the bug. Leave out how the subsystem
  works, because the reader knows its locking and its macros. The bug
  exists, so no "could potentially".
- Preparation or mechanical patch, such as a rename, a move, a helper
  factored out, a wrapper inlined or a reorganization: about four lines
  that open with the change itself, in the form "Factor X out of Y so that
  Z. No functional change." Name the new functions and what they came out
  of, and give one clause on what they are for. The motivation collapses
  to a phrase such as "to prepare for X". Never a step-by-step of what the
  next patch does with them, because that patch's description says so. No
  placement detail such as "next to X", and no quoted prose phrases. The
  shape:

  ```
  foo: Factor out foo_sync_cpu()

  Factor foo_sync_cpu() out of foo_sync_all() so that the CPU hotplug
  callbacks can sync and report one CPU without walking every CPU. No
  functional change.
  ```

- "No functional change", spelled out and never abbreviated, is only for
  renames, moves and pure refactors whose machine behavior is identical. A
  change to a code path, its ordering, its locking or its timing doesn't
  qualify.
- Real logic change: explain fully, but only the non-obvious parts.
- Fix at the end of a chain of earlier commits: a one-paragraph history
  with the commits listed up front as `[N] <sha12> ("full subject")` and
  referred to as `[N]`, dead ends stated with one sentence on why ("[3]'s
  wait turned out to be a dead end"), then the new approach, concretely.
  Its "Verified by" may be a paragraph, when it has three items or fewer.
  An invasive backport ends with a paragraph on confidence, stable
  applicability and the revert fallback.

### Format
- Imperative voice, "Add X", never "This adds X", matching recent commits
  in the same tree.
- The subject is a headline by impact: "Fix soft lockup in the IPI relay
  loop", not "Migrate task in IPI handler". A fix's subject leads with the
  symptom.
- Cite commits in the form
  [patches/commit-tags](../patches/commit-tags.md#commit-references) gives.
- Kerneldoc's `%CONST` markup stays in kerneldoc comments and never appears
  in a subject or body. `@arg` may name a function argument in either, as
  in "taking @rq explicitly".
- "Verified by" in a description is one sentence, such as "Verified with a
  synthetic reproducer.", without iteration counts or machine details.
  Only the chain-of-commits fix above may use a paragraph. The cover letter
  carries more.
- Tags, including attribution, Fixes: and Link:, follow
  [patches/commit-tags](../patches/commit-tags.md). Merge commit messages
  follow [maintain/merges](../maintain/merges.md#merge-messages).

## Pitfalls
- Describing the next patch. A preparation patch took three rounds to
  shrink to four lines, because each draft explained what the following
  patch would do with the new helpers.
- Opening with the mechanism. A description of a new event-driven reader
  opened with the new kernel interface and its poll loop. The accepted
  version opened with what the user gains: no periodic polling, and an
  error report that aborts the run the moment it lands.
- Describing a symptom and the code. A patch adding a callback took several
  review rounds to arrive at the missing capability instead of one
  consumer's symptom, and at properties instead of procedure, with nothing
  the code already shows.

## Customization
- An overlay records a tree's subject conventions when the user wants them
  enforced beyond what the tree's history shows.
- An overlay can add examples from the user's own trees.
