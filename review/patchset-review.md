# Patchset review

- **Load when:** a coherent batch of patches is ready for broader testing
  or for posting, the user asks for a review of a patch, a commit or a
  patchset, or review findings need verifying and fixing.
- **Parameters:** `$REVIEW_PROMPTS`, `$FORMAL_MODEL`, `$SCRATCH`,
  `$PLAN_DIR`
- **Depends on:** [core/validation-scope](../core/validation-scope.md),
  [core/evidence-discipline](../core/evidence-discipline.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md),
  [patches/patch-descriptions](../patches/patch-descriptions.md),
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md),
  [patches/cover-letters](../patches/cover-letters.md),
  [review/formal-model-check](../review/formal-model-check.md),
  [code/kernel-idioms](../code/kernel-idioms.md)

Parallel sub-agents review a batch adversarially, with review-prompts
(https://github.com/masoncl/review-prompts) as the engine for each commit
and one reviewer also checking prose and style.

## Always-on rules
- Review at milestones: one batch review when a coherent batch is ready
  for broader testing or for posting, never one per patch, edit or commit
  update, because a review per edit slows the work and a correction gets a
  focused check instead.
- Skip the milestone review only when the user says "skip review", because
  it catches what building and testing miss before the batch moves on.
- Judge a finding about patch N against the tree with patches 1..N
  applied, never the final tree, and report one that a later patch
  resolves as "valid at N, resolved by N+k", because every patch has to
  build and work on its own for bisection.
- Sub-agents inherit the parent's model. Never set a different one,
  because the user chose the model for the work, and a smaller one misses
  what the parent would find.

## Procedure
### Running a milestone review
1. Start a round at a milestone: a coherent implementation or correction batch
   ready for broader testing, or a finished patchset ready for posting. Builds
   and focused development tests may come first
   ([review/kernel-build](../review/kernel-build.md#procedure)). Keep each patch
   reviewable while the batch grows.
2. Put every commit since the head the last round recorded in the batch, as
   one range `<base>..<head>`, including tooling, scripts, documentation
   and trivial changes. A batch without code is reviewed as in
   [batches without code](#batches-without-code).
3. Update the engine with `git -C $REVIEW_PROMPTS pull --ff-only` when the
   user's overlay or profile allows it. Otherwise use the clone as it is.
4. Start 2-4 parallel sub-agents and divide the batch among them, by
   commits or by the behavior each part changes. Scope each review to the
   changed behavior. Give each reviewer its own detached worktree
   ([tree state](#tree-state-for-each-reviewer)) and the
   [reviewer prompt](#reviewer-prompt). One of them also reviews
   [prose and style](#prose-and-style) for every patch in the batch.
5. With `$FORMAL_MODEL` on, every reviewer names each protocol in the batch
   that meets the model check's
   [trigger](../review/formal-model-check.md#when-a-patchset-needs-a-model).
6. Wait for the reviewers as
   [core/waits-and-long-ops](../core/waits-and-long-ops.md) describes.
7. Re-check each finding against the tree of the patch it names before
   accepting it ([core/evidence-discipline](../core/evidence-discipline.md)).
   The reviewers already ran review-prompts' false-positive guide as part
   of its protocol. `/kverify` in the main session checks only that
   session's own analysis, so it doesn't verify a sub-agent's findings.
8. With `$FORMAL_MODEL` on, the main agent or one dedicated sub-agent
   models the named protocols after the round, as
   [review/formal-model-check](../review/formal-model-check.md#procedure)
   describes.
9. Report the material findings and their resolution, in proportion to the
   change: for each finding, its severity, location and trigger, and the
   patch it is valid at. Style findings keep their own form. With
   `$FORMAL_MODEL` on, the report carries the model lines either way.
   Record the reviewed head in the report or under `$PLAN_DIR`, so that the
   next round knows where it starts.
10. Remove the reviewers' worktrees once their reports are moved out.
11. Fix findings within the scope the user already authorized, without
    asking again. Ask only for a scope or design decision the task didn't
    authorize, or for an action that needs its own approval.
12. Check corrections against the round's results instead of starting a new
    round, with the review prompts loaded once and reused. Review each
    correction with the callers and invariants it affects. A minor
    correction gets a focused check and joins the next milestone, unless
    it invalidates a material safety or correctness conclusion. Reopen
    unchanged analysis only when new evidence or a substantial
    restructuring invalidates it.

### Reviewing one patch on request
1. Check its header: From, Subject and Signed-off-by.
2. Review it as a batch of one, as above.
3. When asked to review a patch and write its description, present the
   review first and wait for the user's confirmation before writing.

## Reference
### Driving review-prompts
review-prompts' kernel/review-core.md runs a deep regression analysis of
one commit. It loads technical-patterns.md and the matching subsystem
guides itself, verifies findings with false-positive-guide.md, and writes
its report in the shape of inline-template.md. This module doesn't repeat
its checklists.

- Reviewer sub-agents always use the direct form: read
  `$REVIEW_PROMPTS/kernel/review-core.md` and run "a deep dive regression
  analysis" of the commit. review-prompts asks for that wording because a
  request for a review makes the agent follow its own idea of a review
  instead of the protocol.
- When review-prompts' commands are installed for the agent harness, they
  appear in its command list, and the main session can use them as a
  shortcut: `/kreview` for one commit, and `/kseries <base>..<head>` when
  one reviewer takes a whole range commit by commit, with
  `--cover-letter <message-id>` for a posted patchset.
  `$REVIEW_PROMPTS/setup.sh <agent> kernel` installs them into the
  harness's global configuration, so run it only when the user asks.
- Given the range, review-core looks forward through it for a later fix
  and still reports a regression that a later patch fixes. That matches
  "valid at N, resolved by N+k".
- review-core ignores problems in test programs unless they crash the
  system. Reviewers check selftest and tool commits themselves.
- `/korcreview` and the agents in kernel/agent/ split the review of one
  commit over several agents, each with a model fixed in its definition.
  Use the direct form, which keeps the parent's model.

### Tree state for each reviewer
Each reviewer works in a detached worktree of its own, as review-prompts'
kernel/scripts/review_one.sh does. Checked out at commit N, the worktree
holds patches 1..N and nothing later, which is the tree a finding about
patch N is judged against. review-core writes review-inline.txt and
review-metadata.json into the current directory, so the worktree also
keeps each reviewer's reports apart from the others and out of the kernel
tree.

```
git -C <tree> worktree add -d $SCRATCH/<topic>/review/<reviewer> <first sha>
git checkout --detach <sha>        # in the worktree, before each commit
git -C <tree> worktree remove $SCRATCH/<topic>/review/<reviewer>
```

After each commit, the reviewer moves its two report files to
`$SCRATCH/<topic>/review/reports/<sha12>/`. The worktree is removed after
the round.

### Reviewer prompt
Fill in and adapt:

```
Your commits are patches <N1>..<N2> of the patchset <base>..<head>:
<sha12> ("<subject>"), one per line.
Work in the detached worktree <worktree>, with it as the current directory. For each of
your commits, in order, run `git checkout --detach <sha>` there, so that the tree holds
patches 1..<N> and nothing later.
Read $REVIEW_PROMPTS/kernel/review-core.md and run a deep dive regression analysis of the
commit, with <base>..<head> as the git range for finding later fixes. Judge every finding
against the checked-out tree. Report a problem that a later patch resolves as
"valid at <N>, resolved by <N+k>". For a selftest or tool commit, check the test itself.
After each commit, move review-inline.txt and review-metadata.json to
<reports>/<sha12>/.
For each finding, give its severity, its location and the trigger.
Name every protocol in which two or more concurrent contexts coordinate through shared
state, with its actors and shared words.          (only with $FORMAL_MODEL on)
```

### Prose and style
One reviewer checks every patch in the batch against
[patches/patch-descriptions](../patches/patch-descriptions.md),
[patches/comments-kerneldoc](../patches/comments-kerneldoc.md),
[patches/cover-letters](../patches/cover-letters.md),
[code/kernel-idioms](../code/kernel-idioms.md#changing-code) and the enabled
style presets:

- the content and length of the description, and word choice
- the call-out of a tightened or loosened validation rule that
  [patches/patch-descriptions](../patches/patch-descriptions.md#what-a-description-says)
  requires
- every comment the patch adds: its shape, clarity, spelling and grammar,
  in the first round, not as an afterthought
- every hunk justified by the purpose the description states. An
  unjustified hunk suggests a split.

Column widths are not a reviewer's task, because the mechanical audit of
[patches/line-widths](../patches/line-widths.md#auditing-a-patchset) checks
them. Each finding takes the form `<patch> (<rule>) <quoted line> -> <fix>`, or
the reviewer reports "clean".

### Batches without code
A batch of workflow documents, plans or other prose gets no kernel path
tracing and no review-prompts run. Reviewers check its scope, its
consistency and whether a plan can be carried out, against the writing
modules and the domain's own guidance.

### Effort
Review counts against the task's effort budget
([core/validation-scope](../core/validation-scope.md)). Don't add review
rounds for test harnesses, manifests or release readiness, because the
milestone round already covers them.

## Customization
- `$FORMAL_MODEL` adds the model trigger and the model lines to every
  round.
- An overlay can add review focus areas for a subsystem or a project, say
  how the review-prompts commands are installed, and allow pulling
  `$REVIEW_PROMPTS` without asking.
