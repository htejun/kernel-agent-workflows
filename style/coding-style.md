# Coding style preset

- **Load when:** writing, changing or reviewing C code or code comments, or
  deciding how to split a code change into patches.
- **Parameters:** none
- **Depends on:** [code/kernel-idioms](../code/kernel-idioms.md),
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md),
  [patches/line-widths](../patches/line-widths.md),
  [style/voice](../style/voice.md)

One developer's C style: larger functions, static-key checks inside the
function, scoped guards, declarations in a fixed order, names derived from
what they touch, and short comments. It adds to
[code/kernel-idioms](../code/kernel-idioms.md#reference) and
[patches/comments-kerneldoc](../patches/comments-kerneldoc.md#reference),
and wins where they differ.

## Always-on rules
- Keep logic in larger functions. A new helper needs two or more callers, a
  contract of its own or a config-stub boundary, because a chain of
  single-caller helpers makes the reader jump around to follow one story
  ([details](#functions)).
- Settle where a mechanism lives, which static key switches its hot path
  and what its names are before the first draft, because each of them
  moved during review costs another round
  ([details](#home-static-key-and-names)).

## Procedure
### Before showing a code change
1. Fold every new helper that has one caller and no contract of its own
   into its caller ([Functions](#functions)).
2. Check the added lines for loop variables declared in the `for`
   statement:

   ```
   git diff | grep -E '^\+.*for \((int|u32|s32|long|unsigned) [a-z_]+ = '
   ```

3. Compare the order of the declarations added to each header with the
   order of their definitions in the .c file ([Declarations](#declarations)).
4. List the comment lines the change removes. Each removal must be forced
   by a change in meaning
   ([patches/comments-kerneldoc](../patches/comments-kerneldoc.md#comments-across-a-patchset)):

   ```
   git diff | grep -E '^-.* \* '
   ```

5. Go over every comment the change adds against [Comments](#comments).
   Shorten each one of four lines or more, or justify its length.

## Reference
### Functions
- Larger functions: one function that tells the complete story, such as a
  timer callback that accrues, admits, escalates and rearms in one body, is
  easier to follow than a chain of five-line helpers. Before adding a
  helper, count its callers. One caller and no concrete prospect of reuse
  means inline it. A helper earns its place with two or more callers, a
  contract that stands apart, or a config-stub boundary.
- An out-of-line function that runs only on a cheap condition, such as a
  static key or a flag, checks it inside the function in the .c file.
  Never a header inline wrapper that tests the condition plus a renamed
  `__foo()` body, and never `if (cond) foo();` at the call sites, which
  stay identical to the unconditional form. In order of preference: an
  early return at the top of the function, restructuring the declarations
  if initializers have to move below the test, or the condition folded
  into an existing early return.
- Fold an early-return check into the condition when it exists only to
  guard the single statement after it:
  `if (cpu >= 0 && handle_lockup(cpu))` instead of `if (cpu < 0) return;`
  followed by `if (handle_lockup(cpu))`. Keep the early return when several
  statements depend on the check or code after them does. Order the tests
  cheap and likely-false first.
- A single guarded statement takes the positive test: `if (cond) do_x();`,
  not `if (!cond) continue;` followed by `do_x();`. An early `continue` or
  `return` is for a body of several statements.
- An if/else whose branches are single statements never gets braces.
- Use guard() and scoped_guard() instead of manual lock and unlock pairs
  wherever the critical section allows, such as `guard(rq_lock_irqsave)(rq)`,
  `guard(preempt)()` and `scoped_guard (sched_change, p, flags)` in the
  scheduler. They remove flag locals, goto-unlock labels and the unlocks
  repeated before early returns. Use guard() when an existing block scope
  matches the lock's coverage and scoped_guard() when guard() would hold
  the lock past where it is needed. Manual pairs stay only where the unlock
  point isn't scope-shaped, such as lock dances and conditional ownership
  transfer. What a guard implies for RCU is in
  [code/kernel-idioms](../code/kernel-idioms.md#scoped-guards-and-rcu).
- No bare `{ }` block to scope variables or to end a guard early. Use
  scoped_guard() for a scoped lock, declare the locals at the top of the
  function or of the branch block, or split out a helper if the block is
  that big. Loop-body locals go at the top of the loop body. A declaration
  in the middle of a body is fine when it has to sit inside an `#ifdef`:
  the block made for scope is the problem, not the position. tools/textfill
  checks for such blocks by default.
- Every state of a state machine and every transition earns its place.
  Identify the ownership phases first, then the number of states. Don't
  fold two states because a window is zero on one path if that forces an
  overloaded state or an inverse rollback transition, such as INIT_BEGIN
  back to NONE, which shows that the state means too much. No uniform
  terminal transitions that are no-op writes. Ask before folding states.
- Use the accessor when one exists: `scx_get_task_state(p)`, not
  `p->scx.flags & SCX_TASK_STATE_MASK`. If the accessor is defined below
  the call site, a separate preparation patch moves it.
- `unlikely()` on an error from a kernel API that is expected to succeed,
  such as an init helper failing, not a bare `if (ret)`.
- No casts that C's integer promotion already covers, and none added "for
  32-bit safety" in code that runs only on 64-bit, such as BPF arena code
  (the BPF arena exists only on 64-bit kernels). Work out the operand and
  result types first. When taking someone else's fix, strip the casts and
  qualifiers that don't carry weight, test the minimal version, and add a
  cast back only if the compiler or the verifier rejects the code without
  it.

### Splitting work into patches
- A pure move is its own patch. It holds the move and any mechanical
  enabler, such as an include the new location needs, stated in the
  description. Restructuring, helper extraction, new static-key checks and
  renames beyond the move come in a separate patch after it. Verify the
  move by reading its diff as a deletion plus an addition of identical
  text.
- A fix goes into the patch that owns the code. A self-contained mechanism
  whose scope doesn't fit that patch, such as a new runqueue field with its
  plumbing and an authorization change, becomes its own patch at a
  sensible position: after its prerequisites, next to related patches, and
  with the least churn to patches already reviewed. A localized fix, such
  as a reorder or a one-line check, stays in its owner.
- Every patch stands on its own. It leaves the tree correct and coherent,
  and its prerequisites, such as the header change that lets moved code
  build
  ([code/kernel-idioms](../code/kernel-idioms.md#compilation-units-and-headers)),
  land in or before it, never after. Its rationale is complete in itself,
  never "to set up a later patch".

### Home, static key and names
Each of these, moved during review, costs a round. Settle them before the
first draft:
- Home: a mechanism that exists only under a config option lives in that
  option's file or `#ifdef` block. Its prototypes sit under the `#ifdef` in
  its header with one-line stubs in the `#else` block
  ([code/kernel-idioms](../code/kernel-idioms.md#compilation-units-and-headers)),
  its per-CPU or per-runqueue fields in the struct's `#ifdef` group, and
  the kfuncs that read its state in the file's kfunc section with stubs in
  the `#else` section. The internal header holds shared types and the main
  file's prototypes.
- Static key: a hot path that only some users need, such as an operation
  only some implementations provide, sits behind a static key from the
  first draft, counted per enabled user as
  [code/kernel-idioms](../code/kernel-idioms.md#static-keys-counted-per-user)
  describes, so that the other users pay one branch.
- Names come from the operation, the field or the key and counter patterns
  the file already has. `_enable()` and `_disable()` name the bracketing
  calls, and arguments follow the order of the helper they feed. No coined
  verbs or role nouns, such as track, deliver, record or receiver.
  Operation parameters are open-coded like those of the neighboring
  operations, and a value the callee can derive isn't passed.
- The cases of a value keep one order everywhere, the order of their first
  definition. The kerneldoc of a walk says who is notified and why the walk
  stops where it does, with a diagram, never the loops restated group by
  group.

### Names and placement
- A helper for one object names the object and the field it writes:
  `sync_pcpu_weight()`, not `sync_weight_node()`. A `_node` suffix reads as
  a NUMA node, so it never names a list node.
- A helper whose job is calling an operation names the operation verbatim:
  `call_foo_updated()` for `ops.foo_updated()`, not `deliver_foo()`.
- The inverse of a verb-noun name keeps its word order: `unlink_foo_work()`
  next to `queue_foo_work()`, not `unlink_queued_foo()`.
- When the pattern is unclear, suggest two or three names and discuss them.
- A new function declared in a shared header and defined in another .c
  file of the subsystem takes the subsystem prefix, such as `scx_` in
  kernel/sched/ext/, where older ones such as drain_descendants() lack it.
  Rename when moving a static out of its file or adding an extern helper,
  the kerneldoc name line included. Don't rename existing names in bulk
  unless asked.
- A helper is defined directly above its first user, never grouped with
  similar helpers far above it. A later patch that adds a second user puts
  the new user below the definition instead of moving the definition up. A
  helper that runs as the first step of another function, called back to
  back with it, sits directly before that function even when its own caller
  is further down.

### Declarations
- Headers group declarations by kind: types together, prototypes together,
  static inlines together. Externs and `DECLARE_*()` lines go with the
  declarations, not among inline bodies. A new inline joins or starts an
  inline cluster after the declarations, never in the middle of a
  prototype block for topical adjacency. Within a block, new declarations
  follow the order of their definitions in the .c file, interleaved with
  the existing ones.
- Loop variables are declared with the other locals, `s32 i;` and then
  `for (i = 0; ...)`, never `for (u32 i = 0; ...)`. They are `s32` by
  default and `u32` only when bounds or array semantics need it.
- Related compile-time constants go in a named `enum`, not a run of
  `#define`s. `#define` stays for one-offs, values the preprocessor
  evaluates, and headers whose users can't take enum types.
- No whitespace alignment across lines in new code: designated
  initializers, prototypes, enum values, `#define` blocks, declarations and
  pointer columns take single spaces, and only leading indentation is
  padded. Alignment is a diff trap, because a longer name added later
  rewrites every neighbor. Blocks that are already aligned follow
  [patches/line-widths](../patches/line-widths.md#scope).
- Out-parameters come after the inputs, and implicit arguments last, so
  that the reader sees what a function acts on before what it produces:
  `foo_query(u64 id, u64 mask, struct foo_mask *out, const struct
  bpf_prog_aux *aux)`.
- `dst` and `src` name the operands of an operation that modifies one of
  them, such as `foo_copy(dst, src)` and `foo_or(dst, src)`, never `dest`,
  `operand` or `source`. Predicates keep names that say their roles:
  `subset(sub, super)`, `intersects(a, b)`. Fix `dest` and `operand` on
  lines the patch already changes.
- `BIT_U64()` and `GENMASK_U64()` for masks that live in u64 fields, never
  `BIT_ULL()` or `GENMASK_ULL()`. u64 is `unsigned long long` on every
  architecture, so the generated code is the same, but the mask's type
  names the field's type, and BIT_U64() fails the build for a constant bit
  number of 64 or more (include/linux/bits.h).
- The kmalloc_obj() family for allocations sized by a variable's type:
  `kmalloc_obj(*p, gfp)` for `kmalloc(sizeof(*p), gfp)`, kzalloc_obj(),
  `kmalloc_objs(*p, n, gfp)` for kmalloc_array(), kzalloc_objs() for
  kcalloc(), the kvmalloc_obj() and kvzalloc_obj() variants, and
  `kmalloc_flex(*p, fam, count, gfp)` for a struct that ends in a flexible
  array. The size follows the type through typeof(), and the pointer comes
  back typed. Don't convert existing call sites unless the patch already
  changes them.

### Comments
These add to
[patches/comments-kerneldoc](../patches/comments-kerneldoc.md#reference).
- A single-line comment starts lowercase and has no trailing period:
  `/* serialize against unlinking */`. A multi-line comment is full
  sentences with capitals and periods.
- A comment of four or more lines is a smell. Shorten it, or justify the
  length.
- A reader who has to ask what a sentence means has found a sentence to
  delete.
- At a check or a write, a comment takes the shape of
  [style/voice](../style/voice.md#fact-rule-instruction), which also rules
  out the derivation and the adversarial scenario.
- Leave out, in addition to what
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md#what-stays-out)
  lists:
  - prose that justifies the mechanism, such as "no indirect calls, no
    retpoline cost"
  - redundant qualifiers: "the live header" becomes "the header"
  - a narration of the mechanism behind a safety claim. State the claim and
    stop.
  - a trailing imperative that restates the code, such as "Migrate
    instead."
  - status-label prefixes, such as `SLEEPABLE:`, `LOCKED:` or `RCU:`
  - parentheticals with more detail than the reader needs
- A trimmed comment says what is checked and why the step matters, not the
  steps:

  ```
  Before: Bound-check @buf against the arena, snapshot @off and @len via
          READ_ONCE(), validate the snapshot, then re-check that the whole
          range fits in the arena.
  After:  Validate @buf against the arena and snapshot @off and @len. The
          snapshot pins the values used for sizing, because BPF can change
          the header at any time.
  ```

- A barrier or race comment follows
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md#where-to-say-it).
  An example from kernel/sched/ext/sub.c:

  ```
  /*
   * An in-flight scx_process_sync_ecaps() batch may still hold the node
   * privately across dispatch-induced rq unlocks, reading as on-list.
   *
   * Because a bypassing sched gets no op call, init_llist_node() and all
   * @pcpu accesses share one contiguous lock hold, off-list under the rq
   * lock means @pcpu won't be accessed again.
   */
  ```

- A lockless permission check is justified in two parts: false positives
  are acceptable because the harm is bounded, and false negatives can't
  happen, said in plain terms of what each side sees, such as "if the
  caller has seen its ownership, the test sees it too". Never "racy,
  best-effort anyway" and never memory-ordering jargon. When such a check
  may be lockless at all is in
  [code/kernel-idioms](../code/kernel-idioms.md#concurrency-patterns).
- A function comment flows from the action to the mechanism with its
  reason, then to the scope, each sentence doing one job. From
  kernel/sched/ext/arena.c:

  ```
  Tear down the pool. Outstanding gen_pool allocations are freed via
  scx_arena_clear_chunk() so gen_pool_destroy() doesn't BUG. The underlying
  arena pages are released when the arena map itself is torn down.
  ```

- A file header holds the SPDX line, the purpose of the code and the
  copyright in one block. A header file with simple declarations gets the
  SPDX line and the copyright only, and the prose lives in the .c file.
- The kerneldoc of a function whose implementation has several parts,
  driven by several constraints, lists the numbered constraints and the
  numbered implementation parts, both at the contract level. Refcount and
  percpu_ref mechanics belong at the implementation site.
- `@arg: description` takes a single space and a single line, with no tab
  padding to align descriptions. A description that doesn't fit on one line
  does too much. An unavoidable wrap aligns under the first word of the
  description.

## Pitfalls
- A patch that added a notification for an operation took several review
  rounds, each moving something that could have been right from the start:
  its file, its static key, its names and its comment wording. Settle them
  before the first draft.

## Customization
- An overlay names the trees the preset applies to, the subsystem's key,
  counter and helper name patterns, the files that hold a mechanism that
  exists only under a config option, and examples from the user's own trees.
