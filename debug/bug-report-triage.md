# Bug report triage

- **Load when:** a task investigates a reported kernel issue, such as a
  mailing-list bug report, a CI or bot finding, a user complaint or a batch
  of AI-generated findings, even when nobody calls it triage.
- **Parameters:** `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [core/git-safety](../core/git-safety.md),
  [patches/patch-descriptions](../patches/patch-descriptions.md),
  [patches/commit-tags](../patches/commit-tags.md),
  [review/patchset-review](../review/patchset-review.md),
  [core/publishing](../core/publishing.md),
  [submit/patchset-posting](../submit/patchset-posting.md),
  [test/vm-testing](../test/vm-testing.md),
  [debug/event-tracing](../debug/event-tracing.md)

This module turns one bug report or a batch of findings into verdicts, and
turns the confirmed, reachable bugs into a patchset with a reproducer for
every patch.

## Always-on rules
- Code inspection alone never confirms a reported bug. Every fix needs a
  runtime signal that fires before it and is gone after it, because a report
  can quote the right code and still rest on one misread line.
- Reproduce a race by widening its window, never by forcing a state the real
  code never produces, because a forced state proves only the forcing.
- Triage a batch one finding at a time, with the user confirming each
  verdict, because a reproducer and fix built on an unconfirmed verdict are
  wasted when the verdict is wrong.
- [GATE] Before posting a fix patchset from a triage: record each patch's
  reproducer tier in the verification table, because reviewers rely on it.

## [GATE] Checklist
### Before posting a fix patchset from a triage
- [ ] Each patch has a Tier A or Tier B reproducer, or is flagged as Tier C
      in the cover letter ([Reproducer tiers](#reproducer-tiers)).
- [ ] The verification table lists each patch's tier, its signal before the
      fix and its silence after the fix.
- [ ] The general posting gates of
      [submit/patchset-posting](../submit/patchset-posting.md#before-posting-a-patch-or-patchset)
      and [review/patchset-review](../review/patchset-review.md) pass as well.

## Procedure
A batch starts at step 1. A single report starts at step 5.

### Intake
1. Read the whole batch once and build a flat table: number, subject,
   claimed severity, area and proposed fix. Don't validate anything yet.
2. Note recurring themes, such as five claims about one global pointer.
   Claims in one theme often share one root cause, or one misreading.
3. Reuse a finding's own reproducer before writing one. A log in which that
   reproducer doesn't trigger the bug is a strong hint that the claim is
   suspect.
4. Fan out first-pass checks to parallel sub-agents, four to six claims
   each, with the verbatim finding text and the instruction to verify
   against the current source, not the report's excerpts. Their one-line
   verdicts only pick the claims that need your own close reading.

### Verifying
5. Before reading the code closely, run the cheap checks:
   - Do the cited line numbers match the named function? Invented claims
     drift on line numbers.
   - Does the claimed race need an object freed and reallocated between two
     adjacent statements? Check whether the object can be freed within that
     window. It can't while the path holds a lock that the free path also
     takes. An RCU-deferred free can't complete while the path is in an RCU
     read-side section or runs with preemption or IRQs disabled.
   - For an init-order claim, check when each path can run. Catches: a
     function claimed broken at init that can't run until much later in
     boot, and init code claimed to race with code that runs only after SMP
     bring-up, BPF program attach or full RCU setup.
6. Label claims that hedge on reachability as unreachable today at once,
   and bucket them in real-defer (step 17): "latent", "could become a bug
   if", "depends on link order", "may be exploitable in a future config".
   Build no reproducer and ship no defensive-only fix for
   them, because a fix for a path no caller reaches fixes no bug. Catches:
   an unwind path that leaves a dangling pointer no caller dereferences.
7. For each claim still open, check in this order:
   1. Read the function and about 50 lines around it in the current source,
      found by a fresh search, not through the report's excerpt.
   2. Expand every macro on the path. Catches: a wrapper macro whose
      expansion drops the argument the report calls unguarded, the most
      common false positive in one large batch.
   3. List every caller, for example with `git grep -nw FUNC`, and check
      whether each one already establishes the missing precondition: a held
      lock, a NULL check, an RCU read-side section. Catches: an unlocked
      read whose callers all hold the lock.
   4. Check that the invariant the report cites exists, for example as a
      `lockdep_assert_held()`, and that some call site breaks it. Reports
      invent invariants.
   5. Search the subsystem's validation, registration and attach code for
      the flags the claim needs. Catches: a flag combination that a
      validation check rejects with -EINVAL.
   6. Check that the path exists in a configuration that can be built
      ([core/evidence-discipline](../core/evidence-discipline.md#stating-a-cause)).
      Catches: a claim that needs `CONFIG_Y=n` while `CONFIG_Y` is a
      `def_bool y` that is on whenever `CONFIG_X` is set.
   7. For a race on a flag, check whether the flag only ever flips one way.
      Catches: a racy `WARN_ON_ONCE()` read of such a flag, which is benign
      and needs a `data_race()` annotation at most, not a fix.
8. Label the claim with what the reading found. No label confirms a bug,
   because only a reproducer does:
   - reachable: it survives every check, and a caller reaches it today. It
     goes on to a reproducer.
   - unreachable today: it survives every check, but no caller reaches it,
     or only a config that is hard to build does. It needs no reproducer.
   - suspect: the checks turned up problems but settle nothing. Plan a
     deeper read or an experiment.
   - disproved: record in one sentence the validation check, guard or init
     path the report missed.
9. Take the findings one at a time, even when the user asked for a fix
   patchset over the whole batch. Present each with its label and wait for
   the user's confirmation. For a reachable claim, build the reproducer,
   show the result before and after the fix, and refine the fix. Then
   bucket it (step 17) and move on.

### Reproducing and fixing
10. Build a reproducer that produces a kernel-visible signal on the base
    kernel and none with the fix, using only the injections and signals in
    [Reproducers](#reproducers).
11. Trace the claimed path with a probe of its exact events, such as the
    remote operation that empties a queue or the wakeup that never comes
    ([debug/event-tracing](../debug/event-tracing.md#choosing-a-tool)).
    What makes a reproducer count is in
    [core/evidence-discipline](../core/evidence-discipline.md#what-counts-as-evidence).
12. Run the reproducer in a VM with the debug configs the bug shape needs
    ([test/vm-testing](../test/vm-testing.md)).
13. When a reproducer keeps failing, suspect the premise, not the technique.
    Stop varying the test and reread the source for the validation check,
    the precondition or the macro that drops the argument. Widen the window
    only if the premise survives and the window is merely narrow. Otherwise
    move the finding to suspect-investigate.
14. Record the tier of each reproducer ([Reproducer tiers](#reproducer-tiers)).
15. Match the fix to the shape of the bug: no structural rewrite for a
    missing guard, and no `READ_ONCE()` for missing locking
    ([Bug patterns](#bug-patterns)). Put one bug in one patch, even when two
    fixes touch the same file.
16. Write no kernel patch for a problem outside the kernel's responsibility,
    such as kfunc locking checks against BPF programs that misuse them, or a
    caller that the contract rules out. The kernel only has to stay
    internally consistent.

### Recording and posting
17. After the reading and the reproduction, put each finding in one bucket:
    - real-fix-now: survives every check and is reachable today. It is
      confirmed only when its reproducer fires on the base, and it goes
      into the patchset once the fix silences that reproducer.
    - real-defer: survives every check, but is unreachable today, triggered
      only by a config that is hard to build, or trivial to fix when the
      code is next touched. It is confirmed only when a reproducer fires on
      the base, and it stays out of the patchset.
    - suspect-investigate: not settled, or its reproducer never fired. Plan
      a deeper read, an experiment or another reproducer angle, and ship no
      fix for it.
    - fp-drop: disproved. Record in one sentence the validation check, guard
      or init path the report missed.
18. For a batch, keep a table of every original finding, its bucket and
    its one-line reason in `$SCRATCH/<batch>/outcomes.md`, current as the
    work goes, so the user can audit it and ask for the status at any time.
    A single report needs no table.
19. Write each description as
    [patches/patch-descriptions](../patches/patch-descriptions.md#by-kind-of-patch)
    gives for a fix for a reported bug, and credit the reporter with
    Reported-by: as
    [patches/commit-tags](../patches/commit-tags.md#attribution-tags) gives.
20. Before rebasing the patchset, keep the old branch as
    [core/git-safety](../core/git-safety.md#before-rewriting-a-branch)
    describes.
21. Verify the [GATE] checklist, then post the patchset per
    [submit/patchset-posting](../submit/patchset-posting.md#posting-a-patchset)
    under the gate in
    [core/publishing](../core/publishing.md#before-publishing).

## Reference
### Report formats
AI-generated findings come as one markdown file with a section per finding,
or as a directory per finding that can also hold a reproducer and its
output. A finding cites a file, a line and code excerpts, and proposes a
fix. A batch of dozens ranges from real high-severity bugs to confident
fabrications.

### Reproducers
Injections may change when a path runs, never what it does. Allowed:
- `udelay()` or `usleep_range()` to widen a race window, gated by a
  `module_param` knob.
- A `pr_warn()` that fires only when the bad state is detected.
- BPF kfuncs that observe the suspect state or count reads of it.
- Relaxed thresholds that make the racing path run more often, such as a
  lower load-balancing threshold or a shorter timer interval.

Never allowed: reassigning tasks or fields, skipping validation checks,
flipping a branch, or setting a field the way the real code never sets it. A
reproducer that needs one is invalid, and the claim is suspect. When
widening the window can't trigger a race, say that the fix is preventive
instead of fabricating a trigger.

Signals that count: an oops, a lockdep splat, a `WARN_ON_ONCE()`, a KCSAN
or KASAN report, a refcount asymmetry counter such as 0 of 3 release calls
before the fix and 3 of 3 after, or an injected detection `pr_warn()` with N
hits before the fix and none after.

Pick the debug configs by the bug shape instead of turning everything on:
- `CONFIG_KCSAN` for data races. Exclude unrelated noisy functions with the
  KCSAN report filter in debugfs. Virtio queues are common offenders.
- `CONFIG_PROVE_LOCKING` for lock-order and lock-held claims.
- `CONFIG_KASAN` and `CONFIG_DEBUG_OBJECTS` for memory-safety and
  object-lifetime claims.
- A preemptible kernel, which brings preemptible RCU. Some races show up
  only there.
- The subsystem options the path needs.

### Reproducer tiers
- Tier A: an observable kernel symptom fires before the fix and is gone
  after it, on the same VM with the same workload.
- Tier B: the unsafe path demonstrably runs before the fix, shown by a
  counter or a printk hit, and is demonstrably rejected after it, shown by
  the guard firing. The race or the corruption itself isn't observed.
- Tier C: the structural change is demonstrated, but no symptom fired, such
  as two distinct pointers showing that an object is now per-instance.

State the tier of each patch. Never blur the tiers: if most patches show the
symptom gone and a few only show the guard firing, say so.

### Bug patterns
- Refcount asymmetry: an object created by a call that takes a reference,
  such as a register function, is released on the wrong object or not at
  all. Count the put and unregister calls against the acquisition sites.
  Fix: release the acquired object on every path that drops it.
- State clobbered by a nested call: an inner call overwrites state that the
  outer caller still needs. Fix: save the state before the nested call and
  restore it after, instead of re-fetching it at every read.
- A lock invariant broken from a timer or softirq: a function that must
  hold a lock while it modifies some state is called from a timer or softirq
  holding a different instance of that lock. Check periodic callbacks that
  modify cross-CPU state. Fix: take the lock instance the contract
  requires before modifying the state.
- A load before the lock: `p = global; ...; lock(); use(p);` uses a value
  that can be stale by the time the lock is taken. Fix: move the load under
  the lock instead of patching the use.
- Use before init: A is used before B is set. Fix: reorder init instead of
  adding NULL checks downstream, wherever the order can change.
- Parallel instances overwriting one global: the global is often left over
  from when only one instance existed. Fix: make the data per-instance, and
  clean up its `#ifdef`s along with it.
- An out-of-bounds table index: an array is indexed by an enum or an offset
  with more values than the array has entries. Look for tables sized by
  their initializers. Fix: size the table with the last enum value and add
  a `BUILD_BUG_ON()`.
- Mismatched config guards: setup under `CONFIG_X` and teardown under
  `CONFIG_Y`. Fix: enumerate all four combinations of the two symbols and
  decide for each helper whether it needs `#ifdef X`, `#ifdef Y`,
  `#if X || Y` or no guard. Fix all the related guards in the same patch.
- A benign data race: a stale value is allowed. Fix: `READ_ONCE()`,
  `WRITE_ONCE()` or `data_race()`, which only silence the tooling. They
  never fix missing locking around an invariant that spans several fields.

## Customization
- `$SCRATCH` sets where the outcomes table goes. An overlay can move it.
- An overlay can name the reproducer harness, the build command, a
  subsystem's debug configs and its own bug patterns.
