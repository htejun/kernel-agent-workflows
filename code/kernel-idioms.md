# Kernel code idioms

- **Load when:** writing, changing or reviewing kernel C code, including
  restructuring branches, changing a function signature, adding a WARN or a
  defensive check, sharing data between CPUs without a lock, counting the
  users of a static key, using scoped guards with RCU, iterating the bits
  of a u64, embedding a struct that ends in a flexible array, organizing
  headers, or relying on CPU hotplug state around suspend and hibernation.
- **Parameters:** none
- **Depends on:** [core/git-safety](../core/git-safety.md),
  [review/patchset-review](../review/patchset-review.md),
  [review/formal-model-check](../review/formal-model-check.md),
  [review/kernel-build](../review/kernel-build.md)

Facts about kernel APIs and code patterns that are easy to get wrong, and
the checks a code change gets before it is committed.

## Always-on rules
- Work out the diff a code change should produce before editing, and before
  committing check that every hunk serves the change: no moved or reordered
  code, no brace changes on lines the change doesn't otherwise touch, no
  dropped debug messages and no collateral edits, because reviewers have to
  account for every hunk and an unrelated one hides in the history
  ([details](#changing-code)).
- Add a defensive check only for a concrete failure scenario, and flag
  every deviation from a diff being ported, because a check for a case the
  call path already rules out hides where validation happens, and a silent
  deviation never gets reviewed ([details](#defensive-code-and-warns)).
- A WARN that a legitimate path can reach is a bug, however rare the path.
  Fix it in the change that makes it reachable, because a WARN asserts an
  invariant, and one that fires on a valid path reports a bug that isn't
  there ([details](#defensive-code-and-warns)).
- [GATE] Before adding a lockless fast path: the shared state is one word
  unless the user approved the scheme as a design question, the writer
  releases unless the reader is explicitly advisory, the reader acquires
  or runs on the writer's CPU, and the access is marked for KCSAN and can't
  tear, because READ_ONCE() and WRITE_ONCE() constrain only the compiler,
  and an ordering argument across words that holds on x86 fails on weakly
  ordered CPUs.
- [GATE] Before committing a signature change: every call site and every
  declaration was updated, in this commit and in each later commit of the
  patchset, because a stale caller breaks the build of that commit and
  bisection with it.

## [GATE] Checklist
### Before adding a lockless fast path
- [ ] The shared state is one word, such as a flag, a counter or a pointer
      published under RCU. State that another CPU reads as more than one
      word goes under a lock, unless the user approved a lockless scheme
      for it as a design question ([one word](#lockless-is-for-one-word)).
- [ ] The writer releases, with spin_unlock(), smp_store_release() or an
      equivalent, unless the reader is explicitly advisory.
- [ ] The reader has a matching acquire or runs in program order on the
      writer's CPU.
- [ ] KCSAN sees the access as intended. test_bit() and the atomic bitops
      are already marked. An open-coded read needs READ_ONCE() or
      data_race() ([lockless reads](#lockless-reads)).
- [ ] The read can't tear. A u64 read tears on 32-bit, but a single-bit
      test reads one half and doesn't. Confirm that the read doesn't need
      several bits to be consistent.

### Before committing a signature change
- [ ] `git grep -nw <name>` finds every call site, and each one matches the
      new signature.
- [ ] The forward declarations near the top of the file and the prototypes
      in headers match the new signature.
- [ ] When the change goes into an earlier commit of a patchset, every
      later commit that calls the function was updated and still applies
      ([rebasing](../core/git-safety.md#rebasing-the-agents-own-patches)).

## Procedure
### Changing code
1. Before editing, work out the diff the change should produce: whether it
   moves code, whether it deletes anything it doesn't intend to, and why
   each line changes.
2. Before adding a write, trace every later write to the same data, so
   that the new write isn't overwritten or dead. Before removing a write,
   check that nothing downstream reads the value.
3. When restructuring branches, enumerate every combination of the
   conditions and trace the old and the new code for each: the same masks
   and data, the same safety properties and the same fallback paths. Hand a
   combination space that isn't trivial to the adversarial reviewers of
   [review/patchset-review](../review/patchset-review.md#running-a-milestone-review).
4. Before committing, read the full diff. Every hunk is justified by the
   change. Code is moved or reordered only when the user asked for it, and
   braces change only where the change alters the number of statements.

## Reference
### Scope of a change
- When the user asks for a minimal change, it gets no restructuring, no
  unrelated line removals and no drive-by refactors.
- When finishing the user's incomplete edit, such as a rename made halfway,
  take the newest declarations and comments as the intended end state and
  the side with most uses as the unfinished one. Propagate toward the
  declaration, never back. Ask when it isn't clear which side is intended.
- When porting someone's diff, such as a contributor's prototype, diff the
  original text against the result. Flag each deviation with its concrete
  reason, because a silent "improvement" is a change nobody reviewed.

### Defensive code and WARNs
- No defensive check without a concrete failure scenario. "Airtight", "belt
  and suspenders", "just in case" and "should still work if" mark a made-up
  risk. Before proposing a safer variant, write out the preconditions, the
  sequence and the observable wrong state in which the primary path fails.
  If you can't, drop the variant.
- No checks at the wrong layer: no bound, NULL or membership check for a
  case that the call path's preconditions rule out, such as an element
  looked up through a table built from it, which contains it by
  construction. Such checks hide where validation happens, cost cycles and
  send readers chasing impossible cases. Checks are for inputs that can be
  invalid at this layer, such as state that races.
- WARN_ON() is for broken invariants, not for valid inputs with a defined
  answer. If a future caller passing the input would be a bug, WARN and
  return a safe default. If the input is a question with an answer, return
  the answer. Obviously bad input from a BPF program gets an error return,
  such as -EINVAL, and the subsystem's error report.
- A WARN asserts an invariant, so a path that trips it without breaking the
  invariant makes the WARN wrong. In review, a change that lets an existing
  WARN fire on a legitimate path is a blocker to fix in the change, or the
  WARN moves. It is never a question of whether guarding it is worth it,
  and never weighed against how often the path runs.

### Lockless reads
Choose the marking of a read without a lock by what the read is for
(tools/memory-model/Documentation/access-marking.txt):
- The bitops, such as test_bit(), for a bit: KCSAN already sees them as
  marked, a test is one load, and the tested bit can't tear.
- READ_ONCE() for a value the concurrency design depends on. It is read in
  one access, and the compiler can't merge, reload or move it.
- data_race() for a read used only for diagnostics, statistics or an
  error-tolerant heuristic. It doesn't change code generation, and it lets
  KCSAN keep checking the accesses the design depends on. Such a read that
  must also be one access is `data_race(READ_ONCE(x))`.

```c
/* a capability bit checked on a fast path */
return test_bit(cpu, caps->bits);

/* a statistics snapshot that may race with updates */
for (i = 0; i < nr_words; i++)
        dst->bits[i] = data_race(src->bits[i]);
```

On 32-bit, a u64 read or read-modify-write can tear, but a single-bit test
depends only on the 32-bit half that holds the bit, and that half loads
atomically.

### Lockless is for one word
- Lockless access is for a single word that one side writes and the other
  reads, such as a flag, a counter or a pointer published under RCU. State
  that another CPU reads as more than one word, such as a value plus a
  timestamp, a flag plus a counter or a record plus its owner, goes under a
  lock that the writers and the readers that need the words consistent
  take. An uncontended queued spinlock acquire is one cmpxchg.
- READ_ONCE() and WRITE_ONCE() constrain the compiler, not the CPU, as does
  C's volatile. A store order chosen so that a reader reading in reverse
  sees only safe combinations holds under x86's total store order and
  breaks on arm64.
- A lockless scheme beyond one word, an established primitive such as
  seqcount or u64_stats_sync included, is a design question for the user
  before any code. A scheme of its own also needs the release and acquire
  barriers written out on both sides and an explicit case table in its
  comment, and the table gets checked by a model of the interleavings and
  litmus tests for the ordering
  ([review/formal-model-check](../review/formal-model-check.md#when-a-patchset-needs-a-model)).

### Concurrency patterns
- Library code marks concurrent accesses symmetrically. In a set of generic
  helpers, if the helpers that modify store with WRITE_ONCE(), the
  predicates read with READ_ONCE(), whatever the current callers do. This
  is correctness, not a defensive check, so the
  [rule against defensive checks](#defensive-code-and-warns) doesn't apply.
- Annotate instead of locking. When an exclusion that lockdep or sparse
  can't see protects an access, use rcu_dereference_check() or
  rcu_dereference_protected() with the true condition at that call site.
  An RCU section or a lock that protects nothing makes readers believe it
  is load-bearing. Keep the conditions of shared helpers strict and
  annotate at the call site that differs.
- Match the primitive to the actual concurrency. Trace who touches the data
  concurrently before reaching for atomics: CPU-local state produced with
  IRQs disabled and drained by that CPU's hardirq needs a plain list_head,
  not an llist. Say "this is CPU-local, so no atomics" when presenting it.
- If proving a design correct needs a paragraph of interleaving reasoning,
  the shape is wrong. Restructure until correctness is obvious: one drain
  point, a single writer of the shared state, or a pattern the file already
  uses.
- A permission check runs locklessly only where deferral forces it to. If
  the operation completes synchronously under the lock that also updates
  the permission state, the check is authoritative under that lock, because
  a false positive there is a real hole. Ask first whether the check can
  run under the authoritative lock.
- No `volatile` on kernel fields or variables
  (Documentation/process/volatile-considered-harmful.rst). BPF programs
  have their own rules in [subsys/bpf](../subsys/bpf.md#arenas) when it is
  enabled.

### Static keys counted per user
A static key that turns on a hot path only while some user needs it counts
the enabled users. The count pairs the enable and the disable on state that
the disable path can still see and that is set right before the increment,
never on whether the user's callback exists, because an enable that fails
before the increment still runs the disable. The selftest covers an enable
failure followed by a phase that must still produce output.

### Scoped guards and RCU
Anything that disables preemption, bottom halves or IRQs marks an RCU
read-side critical section, because the RCU flavors are consolidated
(Documentation/RCU/Design/Requirements/Requirements.rst, the Bottom-Half
and Sched flavor sections). That covers preempt_disable(),
local_bh_disable(), local_irq_save() and raw_spin_lock_irqsave(), and the
guards built on them, so guard(rcu)() next to them is redundant:

```c
/* wrong: guard(rcu)() adds nothing */
guard(rcu)();
guard(irqsave)();

/* right */
guard(irqsave)();
/* RCU read-side critical section already active here */
```

Inside such a section without rcu_read_lock(), dereference with
rcu_dereference_all(). With CONFIG_PROVE_RCU, rcu_dereference() checks for
rcu_read_lock() only and warns, and rcu_dereference_sched() warns under
local_bh_disable() on PREEMPT_RT.

### Bits and masks
- `for_each_set_bit(bit, (unsigned long *)&val, 64)` on a u64 is broken on
  32-bit big-endian kernels. unsigned long is 4 bytes there, the cast
  points at the high half, and for_each_set_bit() numbers bits word by word
  in memory order. Convert first:

  ```c
  DECLARE_BITMAP(bits, 64);

  bitmap_from_u64(bits, val);
  for_each_set_bit(bit, bits, 64) {
          ...
  }
  ```

  bitmap_from_u64() puts the low 32 bits in the first word on 32-bit.
  bitmap_to_arr64() converts the other way.
- No open-coded bit loops such as `__ffs64()` plus `word &= word - 1`. Use
  for_each_set_bit(), for_each_cpu() or the container's own iterator, and
  if the container has none, add one in the same patch.

### Struct layout
- Compilers accept a struct that ends in a flexible array as the last
  member of another struct, but nothing reserves storage for its array.
  TRAILING_OVERLAP() from include/linux/stddef.h embeds it with storage: it
  expands to an anonymous union of the struct and named trailing members,
  laid out from the flexible array's offset. Pair it with a static_assert()
  on the offsets to catch layout drift, as crypto/asymmetric_keys/restrict.c
  does:

  ```c
  static struct {
          /* Must be last as it ends in a flexible-array member. */
          TRAILING_OVERLAP(struct asymmetric_key_id, id, data,
                  unsigned char data[10];
          );
  } cakey;
  static_assert(offsetof(typeof(cakey), id.data) == offsetof(typeof(cakey), data));
  ```

  `cakey.id` is a full struct asymmetric_key_id whose flexible array can be
  used up to the 10 bytes `data` reserves. fs/ocfs2/xattr.c is another
  user.
- A union of the struct and a backing array, as in the holder below, only
  reserves storage behind the struct. TRAILING_OVERLAP() also puts named,
  typed members at the flexible array's offset, which the static_assert()
  checks. A struct with a flexible array can't sit in the middle of another
  struct, because the flexible array must come last, and no sizeof() trick
  portably makes the compiler treat a flexible array as fixed size.
- static_assert() over BUILD_BUG_ON() for struct layout. static_assert()
  works at file scope, so it sits next to the struct definition and the
  reviewer sees the invariant with the layout. BUILD_BUG_ON() works only
  inside a function. static_assert() fails when its expression is false,
  BUILD_BUG_ON() when its condition is true.
- Never re-declare another type's layout to cast to it. Embed the real type
  in a union with a backing array, accessed as `&holder.x` and guarded by a
  size assert, put the union under `#ifdef __BPF__` when the real type
  isn't visible on both sides, or hold a pointer to a separately allocated
  object:

  ```c
  struct foo_holder {
          union {
                  struct foo x;
                  u64 words[FOO_NR_WORDS];
          };
  };
  static_assert(sizeof(struct foo_holder) == struct_size_t(struct foo, ents, FOO_NR_ENTS));
  ```

### Compilation units and headers
- kernel/sched/build_policy.c and build_utility.c include several .c files
  into one compilation unit. That is a build detail: organize every file as
  if it were compiled alone. Symbols used across files are declared in a
  shared header and defined non-static, and nothing relies on a sibling
  file's definitions being visible through the include order. clangd and
  other per-file tools compile each file alone and fail where the order
  was relied on.
- When code moves to another file, list every symbol the moved code uses
  and promote the file-local ones to a shared header in the same patch.
- The headers of such a unit come in a fixed order before the .c files, so
  a header sees only what was included before it. A struct defined in a
  later header can only be forward-declared in an earlier one and used
  through pointers. To embed it, include its header from the earlier one or
  move the definition.
- A mechanism that exists only under a config option gets `static inline`
  stubs in the header's `#else` block. A plain `static` stub warns as
  unused in files that don't call it.
- To verify that files are self-contained, run `clangd --check=<file>` on
  every touched file, with a compile_commands.json for the build from
  scripts/clang-tools/gen_compile_commands.py, and an include-cleaner pass:
  `clang-tidy -checks='-*,misc-include-cleaner' <file> -- <flags>`, or
  clangd's UnusedIncludes and MissingIncludes diagnostics set to Strict in
  an editor. `clangd --check` alone reports neither an unused include nor a
  symbol that comes only through an indirect include. Build with the
  configuration options whose use sites the files hold
  ([review/kernel-build](../review/kernel-build.md#required-options)).

### Signatures
The kernel builds with -Wno-unused-parameter except at W=3
(scripts/Makefile.warn), so an unused parameter compiles without a warning.
A patchset can introduce a function's final signature in its first patch,
and later patches change only the body:

```c
static struct foo *foo_alloc(struct foo_set *set, int idx, int node)
{
        return kzalloc_node(sizeof(struct foo), GFP_KERNEL, node);
}
```

### CPU hotplug, suspend and hibernation
With CONFIG_PM_SLEEP_SMP, regular CPU hotplug and a suspend or hibernation
cycle exclude each other (kernel/cpu.c, kernel/power/suspend.c):
- cpu_hotplug_pm_callback() calls cpu_hotplug_disable() on
  PM_SUSPEND_PREPARE and PM_HIBERNATION_PREPARE, and cpu_hotplug_enable()
  on PM_POST_SUSPEND and PM_POST_HIBERNATION.
- cpu_hotplug_disable() takes cpu_add_remove_lock, so it waits for a
  hotplug operation in flight. While hotplug is disabled, cpu_down() and
  cpu_up() return -EBUSY. No regular hotplug operation runs during a cycle,
  and no cycle starts in the middle of one.
- The only CPU downs and ups inside a cycle are the cycle's own.
  freeze_secondary_cpus() and thaw_secondary_cpus() call _cpu_down() and
  _cpu_up() directly with tasks_frozen set, which callbacks see as
  cpuhp_tasks_frozen.
- They hold cpu_add_remove_lock across the whole loop but take
  cpus_write_lock() for each CPU, so cpus_read_lock() doesn't exclude a
  cycle. A non-freezable kthread on a CPU not yet down can take it between
  two downs and see some CPUs offline.
- A failed thaw ("Error taking CPU%d up") leaves the CPU offline after the
  cycle ends.
- The exclusion has gaps. cpuhp_smt_enable() calls _cpu_up() without
  checking whether hotplug is disabled. Its callers include the SMT control
  file in sysfs, x86 hibernation code and x86 idle driver initialization.
  cpu_hotplug_pm_callback() re-enables hotplug in its own POST
  notification, which runs before any priority-0 PM notifier registered
  later, so a regular up can land between the two. PM_RESTORE_PREPARE
  doesn't disable hotplug at all.
- Suspend-to-idle takes no CPU down: suspend_enter() runs s2idle_loop()
  before the point where the secondary CPUs would go down.

## Pitfalls
- A port of a contributor's change translated one of two changes on a call
  and went out without a flag. Diff the original text against the port and
  flag every deviation.
- A patch dropped one condition from the test in front of a WARN, which let
  the WARN fire on a valid path, and the draft review reply asked whether
  guarding it was worth it. A WARN reachable from a legitimate path is a
  blocker, fixed in the change.
- A statistics handler wrote a timestamp, a slot and a counter as
  WRITE_ONCE() stores, and a userspace sampler read them in reverse order
  so that a sample straddling an update would under-count rather than
  over-count. The argument held only under x86's store order. On arm64 both
  excluded cases came back: the interval counted twice, and the interval
  credited to the wrong slot. The fix was a lock around the pair and the
  charge, and a step that closes open intervals before each sample so that
  the sampler reads cumulative counters only.
- The first build of a static-key count undercounted on every rejected
  attach, and no test caught it until a whole chain of notifications went
  dark. Pair the count on state the disable path can see, and test an
  enable failure ([static keys](#static-keys-counted-per-user)).
- A file split built only because the combined unit included the original
  file first, and a reviewer found the new file failing in clangd. List
  the symbols moved code uses, as
  [compilation units](#compilation-units-and-headers) says.

## Customization
- An overlay can add a subsystem's idioms, its error report for bad BPF
  input, the files its tree combines into one unit with their header order,
  and examples from the user's own trees.
