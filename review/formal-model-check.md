# Formal model checks

- **Load when:** `$FORMAL_MODEL` is `on` and a patchset adds or changes a
  protocol in which two or more concurrent contexts coordinate through
  shared state, or the user asks for a TLA+ model, a TLC run or a herd7
  litmus test.
- **Parameters:** `$FORMAL_MODEL`, `$MODEL_CHECK_HOST`, `$PLAN_DIR`,
  `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [core/validation-scope](../core/validation-scope.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md)

The agent translates the protocol's C into a PlusCal model, and TLC checks
every interleaving. Memory-ordering claims go to herd7 litmus tests instead,
because TLC assumes sequential consistency.

## Always-on rules
- With `$FORMAL_MODEL` on, every multi-actor shared-state protocol a patchset
  adds or changes gets a model check, because reading misses rare interleavings.
- [GATE] Before trusting a model result: it failed a falsification run and its
  traces replay on the C, because a weak model passes and a wrong one fails.

## [GATE] Checklist
### Before trusting a model result
- [ ] The model was written from the C code paths, not from the patch
      description. A list next to it names every abstraction: what was
      dropped, what was merged into one step, and which constants bound it.
- [ ] A falsification run removed the protective mechanism (the lock, the
      check, the order of two steps) and TLC reported the violation.
- [ ] The exhaustive run is recorded with its constants, distinct states,
      depth and wall time. A `-simulate` run is labeled as sampling.
- [ ] Every counterexample maps step by step onto C lines. A trace that
      needs an abstracted step is a model defect, not a finding.
- [ ] The report states the sequential-consistency assumption. Ordering
      claims went to herd7 or are marked UNKNOWN.

## Procedure
1. Decide whether the patchset needs a model by the
   [trigger](#when-a-patchset-needs-a-model). Reviewers name the candidate
   protocols. After the review round, the main agent or one dedicated
   sub-agent writes the model.
2. Pick the actors and the shared words. Each concurrent context is one
   process, and the shared state is the set of words they read or write in
   the protocol. Everything else is process-local or dropped. A variable
   earns its place only when leaving it out hides an interleaving, because
   each variable multiplies the state space.
3. Abstract the data.
   - Identities are model values (`CPUS = {p1, p2}`, `TASKS = {t1, t2,
     t3}`), never integers, so that `SYMMETRY` can collapse permutations.
   - Counters are bounded, with a `MAXVAL` and a wrapping increment, or
     with a `CONSTRAINT` that stops exploring past the bound.
   - A multi-field word is a record, such as `[locked |-> FALSE, pending
     |-> FALSE, tail |-> NoCPU]`. Read or write it as one value where the C
     does so atomically, and as separate fields where it doesn't.
   - A per-CPU array is a function `[c \in CPUS |-> ...]`, a queue is a
     set or a sequence depending on whether order matters, and a
     pointer-linked structure is a function over a small set of addresses.
4. Translate the code paths. Follow the C control flow, not the patch
   description.
   - Write each atomic, such as cmpxchg(), xchg(), atomic_inc() or
     fetch_or(), as a macro. A macro can't contain labels, so it stays one
     step.
   - A spinlock is `await ~l; l := TRUE;` and `l := FALSE;`.
   - Interrupt enable state is a per-CPU flag. An interrupt is either a
     separate process that `await`s the flag, or an `either { call irq(); }
     or { skip; }` branch at every point where one can land.
   - Nondeterministic choices, such as which task runs next, whether a
     trylock succeeds or whether a timer fires, are `either`/`or` or `with
     (x \in S)`.
   - Put a label at every access to a shared word and nowhere else. This
     is the coarsest grain with one shared access per step. It keeps the
     state space small, and every interleaving of the shared accesses still
     exists in the model.
   - Exception: a lock acquire or release may share a step with the first
     or last access of its critical section, because no other process can
     touch the protected words in between. This doesn't hold when some
     process reads the lock word without taking the lock, as
     spin_is_locked() does.
5. State the properties.
   - Always include `TypeInv`, which says every variable is in its value
     set. It catches translation slips.
   - State the patch's correctness claim as a state predicate: mutual
     exclusion over `pc` (`~(pc[a] = "cs" /\ pc[b] = "cs")`), "no free
     while referenced" (`count = 0 =>` no task points at it), "at most one
     owner".
   - An `assert` inside a step checks what the code assumes at that point.
   - A history variable checks a concurrent structure: keep an abstract
     copy, such as a plain `queue`, update it in the same step as the real
     linearization point, and assert that the concrete result matches.
   - Progress claims, such as "every waiter eventually proceeds", are
     `PROPERTIES` with `~>` on `fair` processes. Check them separately and
     last.
6. Translate and parse: `pcal -nocfg`, delete the `.old` backup, then run
   SANY ([commands](#running-tlc)). The translator and SANY messages are
   exact, so iterate until both are clean. Then read the generated TLA+
   for the steps that matter: which variables each action changes, where
   each `await` landed, and what `UNCHANGED` covers.
7. Falsify before trusting. Remove the mechanism the patch relies on
   (comment out the lock, drop the check, swap two steps) and run TLC. It
   must report a violation with a trace that reads like the bug the
   mechanism prevents. Then restore the mechanism. A knob constant such as
   `LOCKED` makes the two runs one cfg apart. Run the falsifications before
   the exhaustive runs: they take seconds, and an exhaustive pass counts
   only once its falsification has failed.
8. Run exhaustively, small first: two actors, then three, with symmetry on
   the identity sets and constraints on the counters
   ([running](#running-tlc)). Review models stay at two or three actors and
   finish in minutes, within the review's effort budget
   ([core/validation-scope](../core/validation-scope.md)). A larger model
   is a separate task with its own budget. Add liveness only when the claim
   is about progress, and expect it to take orders of magnitude longer. If
   the size that matters is out of reach, `-simulate -depth N` samples
   random behaviors. Sampling can find bugs but can't show their absence,
   so the report says "sampled".
9. Clean up each run as soon as its result is recorded
   ([cleanup](#cleanup)).
10. Map every trace to the C. `-difftrace` prints only what changed in each
    step. For each step, name the C line and the context executing it. If
    a step has no C counterpart, or two words change together in the
    model but separately in the C (or the reverse), the model is wrong:
    fix it and rerun. Only a trace that replays on the code is a finding.
    Report it with the interleaving written out, labeled "model-derived,
    replayed on the code", never as an observed failure
    ([core/evidence-discipline](../core/evidence-discipline.md)).
11. Record and report. Store the model as `$PLAN_DIR/tla/<topic>.tla` with
    its cfgs. Add a paragraph to the plan notes: what was modeled, the
    abstractions, the constants, the falsification run, the distinct
    states and the result. The review report has one line per model, or a
    "model: no" line with the reason when the patchset needs none, so that
    a skipped check is visible:

    ```
    model: yes <protocol>: <actors and constants>, <N> distinct states, depth <D>, <time>;
      falsified by removing <mechanism>; sequential consistency assumed
    model: no <reason>
    ```

    Patch descriptions and cover letters mention the model only as
    [patches/patch-descriptions](../patches/patch-descriptions.md#what-stays-out)
    and [patches/cover-letters](../patches/cover-letters.md#testing-summary)
    allow.

## Reference

### What PlusCal and TLC check

PlusCal is a pseudocode front end to TLA+ with C-like syntax. The algorithm
sits in a comment of the `.tla` file, and the translator writes the TLA+
into the same file between `\* BEGIN TRANSLATION` and `\* END TRANSLATION`.
Processes are the actors, and procedures and macros structure the code.
Labels mark the grain of atomicity: everything from one label to the next
is one indivisible step, and other processes interleave only at labels. The
labels are the model's central claim, so a reviewer checks them first.

For fixed small constants, TLC enumerates every reachable state and checks
the invariants on each. A violation comes back as the shortest trace that
reaches it. A clean run means that the model, as written, has no reachable
bad state for these constants. It says nothing about the C, the compiler or
anything the model leaves out. A violation means that one interleaving of
the modeled steps breaks the invariant. It is a candidate bug until the
trace replays on the real code.

### When a patchset needs a model

A patchset needs a model when any of these holds:

- It adds or changes a protocol in which two or more actors coordinate
  through shared state, and the correctness argument spans more than one
  step of more than one actor. Actors are CPUs, tasks, hardirq or softirq
  contexts, a BPF program against the kernel, or a userspace sampler
  against the kernel. Typical shapes are enable or disable counting paired
  against the state the disable path can see, refcount handoff, ownership
  or claim-flag queues, state machines with transitions from more than one
  context, lockless single-word schemes justified by step order ("A can't
  observe X because B checks Y first"), and generation or rollover schemes.
- A reviewer reports an interleaving it can't settle by reading, or two
  reviewers disagree about one.
- The user asks for a model.

A patchset doesn't need a model for:

- pure refactors and code moves
- state touched only inside one lock's critical sections, because the lock
  makes the protocol single-actor at the model level
- data structure or arithmetic changes with no concurrent transitions
- memory-ordering questions, such as release and acquire pairing, barrier
  placement or whether a READ_ONCE() is enough. Those are
  [herd7 litmus tests](#memory-ordering-with-herd7).
- anything whose state space is a whole subsystem. The model is the
  protocol slice and nothing else. Modeling a whole scheduler path is the
  known anti-pattern.

### PlusCal crib

Copy this model as the starting point for a new one. It has per-actor
loops, lock macros, a knob constant for the falsification run, a type
invariant, a counter constraint and a symmetry set. With `LOCKED = FALSE`,
TLC reports `SnapInv` violated: the sampler reads the new value and the old
counter between the writer's two stores.

```
---- MODULE pairsnap ----
\* Per-CPU writers advance a pair of words (val, seq) together, and a sampler
\* snapshots one CPU's pair. LOCKED = FALSE must violate SnapInv.
EXTENDS Naturals, Sequences, FiniteSets, TLC

CONSTANTS CPUS,         \* set of model values, such as {p1, p2}
          RD,           \* model value naming the sampler process
          MAXCNT,       \* bound on the counters, keeps the state space finite
          LOCKED        \* TRUE: writer and sampler take the per-CPU lock

ASSUME /\ MAXCNT \in Nat \ {0}
       /\ LOCKED \in BOOLEAN

(* --algorithm pairsnap {
variables
        val = [c \in CPUS |-> 0];
        seq = [c \in CPUS |-> 0];
        lock = [c \in CPUS |-> FALSE];
        snap = [cpu |-> 0, val |-> 0, seq |-> 0];

define {
        TypeInv == /\ val \in [CPUS -> 0..MAXCNT]
                   /\ seq \in [CPUS -> 0..MAXCNT]
                   /\ lock \in [CPUS -> BOOLEAN]
        \* the pair advances together, so a consistent snapshot has val = seq
        SnapInv == pc[RD] = "rd_check" => snap.val = snap.seq
        Constr == \A c \in CPUS : seq[c] < MAXCNT
        Perms == Permutations(CPUS)
}

macro spin_lock(l) { await ~l; l := TRUE; }
macro spin_unlock(l) { l := FALSE; }

fair process (writer \in CPUS)
{
w_loop: while (TRUE) {
                if (LOCKED) spin_lock(lock[self]);
w_val:          val[self] := val[self] + 1;
w_seq:          seq[self] := seq[self] + 1;
                if (LOCKED) spin_unlock(lock[self]);
        }
}

fair process (sampler = RD)
{
rd_loop: while (TRUE) {
                with (c \in CPUS) snap.cpu := c;
                if (LOCKED) spin_lock(lock[snap.cpu]);
rd_val:         snap.val := val[snap.cpu];
rd_seq:         snap.seq := seq[snap.cpu];
                if (LOCKED) spin_unlock(lock[snap.cpu]);
rd_check:       skip;
        }
}
} *)
\* BEGIN TRANSLATION
\* END TRANSLATION
====
```

The next snippet makes cmpxchg() a macro and retries it in a procedure. It
assumes a global `owner` and a model value `NoCPU`. The statement after a
`call` needs a label, here `cs`, where the caller's critical section
starts.

```
macro cmpxchg(v, old, new, ret) { ret := v; if (v = old) v := new; }

procedure claim()
        variable seen = NoCPU;
{
cl_try: cmpxchg(owner, NoCPU, self, seen);
        if (seen # NoCPU) goto cl_try;
cl_ret: return;
}

fair process (cpu \in CPUS)
{
loop:   while (TRUE) {
                call claim();
cs:             skip;
rel:            owner := NoCPU;
        }
}
```

`self` is the process identity, and `pc[p]` is the label process p is at.
A local `v` of a process set is `v[p]` in invariants. Such invariants go
below `\* END TRANSLATION`, where the locals are declared. `f[k] := v` is
the usual PlusCal form of a function update, and `x[1] := a || x[2] := b`
sets two components of one variable in one step. Procedures can't return
values, so pass them through a global indexed by `self`.

Label rules, enforced by the translator:

- The first statement of a process or procedure is labeled.
- Every `while` is labeled.
- A statement after `call`, `return` or `goto` is labeled, and so is a
  statement after an `if` or `either` that contains one of those or a
  label.
- No labels inside a macro or a `with` body.
- A label separates two assignments to the same variable, except within
  one `||` assignment.

For the full statement and expression syntax, see the PlusCal manual (C
syntax) on the TLA+ home page.

### TLC configuration

A cfg file has the spec's basename by default. The crib's pass and
falsification cfgs differ in one line:

```
SPECIFICATION   Spec
CONSTANTS       CPUS = {p1, p2}
                RD = RD
                MAXCNT = 3
                LOCKED = TRUE           \* FALSE in pairsnap-lockless.cfg
INVARIANTS      TypeInv SnapInv
CONSTRAINTS     Constr
SYMMETRY        Perms
```

- A constant of the form `X = X` declares a model value, and `{p1, p2}`
  declares a set of them.
- `PROPERTIES` lists liveness properties, such as `LockAll == \A p \in
  ProcSet : pc[p] # "cs" ~> pc[p] = "cs"` for the cmpxchg() snippet.
  Define it below `\* END TRANSLATION`, where `ProcSet` exists. Drop
  `SYMMETRY` from a cfg that checks properties, because TLC's liveness
  checker isn't sound with it.
- TLC reports a deadlock only when no process can step. A process stuck at
  an `await` while others still step, or a retry loop that never succeeds,
  is not a deadlock to TLC. Only a liveness property catches it. Set
  `CHECK_DEADLOCK FALSE` only when every process legitimately stops at the
  end.
- When a procedure variable has no initial value, the translator emits
  `CONSTANT defaultInitValue`. Then add `CONSTANT defaultInitValue =
  defaultInitValue` to the cfg.

### Running TLC

Translate and parse where the model is edited. Run TLC on
`$MODEL_CHECK_HOST` unless the instance is known to finish in seconds,
because run time is hard to predict from a model's size. On a remote host
that doesn't share the working tree's filesystem, copy the model and its
cfgs over, start each instance detached, and copy its result line back
into a local summary file under `$SCRATCH/<topic>/`. Wait for runs as
[core/waits-and-long-ops](../core/waits-and-long-ops.md) describes. TLC's
periodic Progress lines give the distinct states found so far.

```
java -cp tla2tools.jar pcal.trans -nocfg <topic>.tla && rm -f <topic>.old   # keep the cfg
java -cp tla2tools.jar tla2sany.SANY <topic>.tla                           # parse
java -XX:+UseParallelGC -Xmx8g -cp tla2tools.jar tlc2.TLC -workers auto \
        -metadir "$SCRATCH/<topic>/tlc" -config <topic>.cfg <topic>.tla
java ... tlc2.TLC -difftrace ... <topic>.tla                    # traces show changes only
java ... tlc2.TLC -simulate -depth 200 ... <topic>.tla          # sampling, never a pass
```

Distribution packages wrap these as `pcal`, `tla2sany` and `tlc`, which
take the same arguments. `-deadlock` turns off deadlock checking,
`-continue` keeps going after a violation, and `-checkpoint N` with
`-recover ID` resumes a long run.

Reading the output:

- A pass ends with "Model checking completed. No error has been found."
  and the state counts, depth and time.
- A violation prints "Invariant X is violated." and the trace as `State 1`
  to `State n`, with `pc` showing where every process stands.
- "Temporal properties were violated." names no property, and its trace is
  not minimal.

### Cleanup

A TLC run leaves a metadir with fingerprint partitions, the state-queue
spill and trace pages. On a large instance it grows to tens or hundreds of
gigabytes.

- Point `-metadir` outside the working tree, and delete the translator's
  `.old` backup after each translation.
- Delete each run's metadir as soon as its summary line is recorded: the
  result, distinct states, depth and time, plus the last Progress line for
  an aborted run.
- On a remote host, also delete the run's log once its result line, and
  its counterexample trace if it has one, are copied back. The host's run
  directory is empty when the batch of runs ends.
- Keep only the model, the cfgs, the summary file, counterexample traces
  and the report.
- A run suspended with SIGSTOP keeps its metadir until it is killed or
  resumed.

### Memory ordering with herd7

TLC steps are sequentially consistent, so a TLC pass says nothing about
whether the C's accesses are ordered. A claim such as "the reader can't see
the new value with the old counter", "this release pairs with that acquire"
or "this smp_mb() is needed" is a litmus test.

The kernel ships its memory model, the LKMM, in tools/memory-model, and
herd7 runs it. Worked tests are in tools/memory-model/litmus-tests/, such
as MP+pooncerelease+poacquireonce, SB+fencembonceonces and the lock
variants. tools/memory-model/Documentation/litmus-tests.txt describes the
format. A test is a few lines of C per thread plus an `exists` clause that
describes the bad outcome.

```
cd tools/memory-model           # in a kernel tree
herd7 -conf linux-kernel.cfg "$SCRATCH/<topic>/<topic>.litmus"
```

"Never" under `Observation` means the model forbids the bad outcome.
"Sometimes" or "Always" means the ordering argument is wrong.
tools/memory-model/README names the herd7 version it needs. klitmus7 turns
a test into a kernel module that runs it on hardware.

### Installation

- The TLA+ tools are one jar, tla2tools.jar: the SANY parser, TLC, the
  PlusCal translator, TLATeX and a REPL. It needs Java 11 or newer, and a
  headless runtime is enough. Releases are at
  https://github.com/tlaplus/tlaplus/releases. Use the stable release. The
  rolling pre-release is rebuilt from every commit.
- herd7 and klitmus7 come from herdtools7, https://github.com/herd/herdtools7.
  Install them with opam as its INSTALL.md describes, or from a
  distribution package.

### Kernel precedents

kernel-tla is a public repository on git.kernel.org with TLA+ models of
kernel code such as the spinlocks, context switch and the arm64 ASID
allocator. Its qspinlock
model found the liveness problem fixed by 59fb586b4a07 ("locking/qspinlock:
Remove unbounded cmpxchg() loop from locking slowpath") and 6512276d97b1
("locking/qspinlock: Bound spinning on pending->locked transition in
slowpath"). Start from its ticketlock.tla, the smallest model, before
writing a new one.

### Sources

- The PlusCal manual (C syntax) and "Current Versions of the TLA+ Tools",
  which lists TLC's options, both on the TLA+ home page.
- kernel-tla on git.kernel.org, with its README.
- tools/memory-model/README in the kernel tree, for the LKMM and herd7.

## Pitfalls
- Weakening the spec until TLC passes: a constraint that the race doesn't
  occur, an interleaving declared acceptable, or labels coarsened until the
  race disappears. Model the mechanism the code has instead. If the code
  has none, that is the finding.
- Vacuous properties: an invariant true by construction, the end state as
  an invariant, or a property that restates the implementation. The
  falsification run catches them every time.
- Wrong abstraction level: treating model devices such as `pc` or a history
  variable as implementation state, or modeling timing, such as a sleep, as
  a value instead of a nondeterministic step.
- Translator-added labels: `pcal -label` adds the minimal labeling, which
  hides interleavings. Label a concurrency model by hand.
- Syntax slips: `=` for `==`, `!=` for `#`, TLA+ syntax in the cfg, two
  assignments to one variable in one step, a label inside a macro or a
  `with`. A model with procedures fails SANY without `EXTENDS Sequences`,
  because the translation keeps the call stack as a sequence.
- Model values compare silently: a model value compared with anything but
  a model value is unequal, with no warning. Typed model values, such as
  `p_1` with the prefix `p_`, make TLC report a comparison between
  different kinds.
- A pass in a fraction of a second, or with a few hundred distinct states
  on a concurrency model, is a sign that the model is too small or the
  invariant too weak.

## Customization
- `$FORMAL_MODEL` turns the check on for patchset review.
  `$MODEL_CHECK_HOST` picks where TLC runs go, and `$PLAN_DIR` where models
  are stored.
- An overlay usually holds the model-check host's details and its
  large-run settings (NUMA pinning, fingerprint set, worker count), local
  install paths, and a local starting template.
