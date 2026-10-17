# sched_ext

- **Load when:** changing sched_ext in the kernel (kernel/sched/ext,
  include/linux/sched/ext.h, tools/sched_ext and its selftests) or a BPF
  scheduler in the scx repository, testing, tracing or debugging a
  sched_ext scheduler, or reviewing such a change.
- **Parameters:** none
- **Depends on:** [subsys/bpf](../subsys/bpf.md),
  [core/git-safety](../core/git-safety.md),
  [core/overlays](../core/overlays.md),
  [review/kernel-build](../review/kernel-build.md),
  [review/patchset-review](../review/patchset-review.md),
  [code/kernel-idioms](../code/kernel-idioms.md),
  [debug/event-tracing](../debug/event-tracing.md),
  [debug/crash-debugging](../debug/crash-debugging.md),
  [debug/bug-report-triage](../debug/bug-report-triage.md),
  [test/perf-comparison](../test/perf-comparison.md),
  [submit/github-pr](../submit/github-pr.md)

Conventions of the sched_ext tree and of the scx repository
(https://github.com/sched-ext/scx), where most BPF schedulers live, and
recipes for testing, tracing and reviewing schedulers.

## Always-on rules
- A patch that changes `struct sched_ext_ops` adds its compat handling to
  tools/sched_ext/include/scx in the same patch, because schedulers built
  against the new headers must still load on older kernels
  ([details](#interface-changes)).
- Decode task and enqueue flags, `p->flags`, `p->scx.flags` and
  `enq_flags`, with a decoder script, never by hand, and write one from the
  flag definitions in the tree's headers when none ships, because one
  misread bit sends the analysis after the wrong state.
- [GATE] Before a commit of Rust code: `cargo fmt` ran, `cargo build`
  passed, the branch isn't main, and the user saw `git diff --cached` and
  confirmed it, because CI rejects an unformatted or unbuildable commit and
  a stray staged file goes out with the pull request.

## [GATE] Checklist
### Before a commit of Rust code
- [ ] `cargo fmt` ran and `cargo build` passed. In the scx repository, the
      checks that its AGENTS.md lists before each commit passed too.
- [ ] The commit goes on a topic branch, not main.
- [ ] The working tree holds no change outside the commit, and the files
      were staged by name
      ([core/git-safety](../core/git-safety.md#always-on-rules)).
- [ ] The user saw `git diff --cached` with a description of every change,
      and confirmed it.

## Procedure
### Testing sub-schedulers
1. Boot the test kernel with lockdep. Many sub-scheduler bugs show up as
   lockdep reports, so lockdep is the main oracle.
2. Build a cgroup hierarchy such as A/B/C, with workers at each level and a
   sub-scheduler attached at each level. `scx_qmap -c <cgroup path>`
   attaches as a sub-scheduler, using the directory's inode number as the
   cgroup ID. Several instances nest.
3. Start the worker processes before opening any BPF skeleton. A child
   forked without exec after the load keeps the struct_ops link's file
   descriptor, which is close-on-exec, and the unload waits until that
   child exits. The unregistration completes with the last close.
4. Cover each dimension: load and unload cycles, nested hierarchies, rapid
   attach and detach, the closest scheduler winning, SysRq-D (dump, which
   exercises the debug paths) and SysRq-S (disable all, which exercises the
   cleanup), suspend and resume, error injection, fork storms, and machines
   with many CPUs and NUMA nodes.
5. Inject errors with `scx_qmap -e <count>`, which fails after that many
   enqueues: a sub-scheduler error while the root stays alive.
6. Run a fork storm, such as `stress-ng --fork 8 --timeout 5s`, while
   sub-schedulers attach and detach.
7. Tear down both ways: leaf to root, and the root first while its children
   are loaded.
8. Run on a small VM for the basic bugs and on a large multi-node machine
   for the scaling ones, with 32 or more workers per cgroup to expose
   per-CPU locking races.

## Reference
### Interface changes
- An additive change to `struct sched_ext_ops`, a new field or callback,
  adds a load-time fix-up stanza to `SCX_OPS_OPEN()` in
  tools/sched_ext/include/scx/compat.h, which probes BTF for the field,
  warns and clears it, and a line to the file's `COMPAT:` version list. The
  stanza can only drop what the kernel lacks. An incompatible change, such
  as a new callback signature, needs several variants of the ops struct.
  `SCX_OPS_DEFINE()` in compat.bpf.h defines one variant today, and such a
  change extends it to emit the variants, from which the loader picks the
  one the kernel supports.
- The cid form, `struct sched_ext_ops_cid` and `SCX_OPS_CID_OPEN()`, gets
  no compat handling. That is the current policy, until cid-form
  schedulers target released kernels.
- A wrapper in compat.bpf.h for a kfunc or a flag records only the kernel
  version that added it, `v7.4: Add scx_bpf_foo().`, with no drop point,
  no support-window sentence and no narration of what the wrapper returns.
  Older blocks in the file still carry drop points. Don't ask contributors
  to add one.
- Callback argument suffixes follow
  [subsys/bpf](../subsys/bpf.md#struct_ops-callbacks).
- An op variant qualified by a child takes the child's cgroup ID first,
  followed by the arguments of the unqualified op unchanged:
  `ops.sub_child_ecaps_updated(cgroup_id, cid, before, after)` mirrors
  `ops.sub_ecaps_updated(cid, before, after)`.
- The tools and selftests that
  [review/kernel-build](../review/kernel-build.md#procedure) builds after
  the kernel are tools/sched_ext and tools/testing/selftests/sched_ext.

### Code layout
- kernel/sched/build_policy.c includes ext.c, cid.c, arena.c, idle.c and
  sub.c from kernel/sched/ext into one compilation unit. Each file still
  stands alone
  ([code/kernel-idioms](../code/kernel-idioms.md#compilation-units-and-headers)).
  Each .c file has its own header, cid.h, arena.h, idle.h and sub.h, for
  the declarations other files use. internal.h holds shared types and
  ext.c's prototypes.
- Where a mechanism lives, its static key and its names take the forms
  below in sched_ext. The general forms and their reasons are in
  [style/coding-style](../style/coding-style.md#home-static-key-and-names)
  when that preset is enabled.
- Home: a sub-scheduler mechanism lives in sub.c inside its
  CONFIG_EXT_SUB_SCHED block, with its prototypes under the ifdef in sub.h
  and their stubs in the `#else` block, as for `scx_online_ecaps()` and
  `scx_offline_ecaps()`. Its rq fields go in the ifdef group of
  `struct scx_rq`, and kfuncs that read its state go in sub.c's kfunc
  section with a stub in the `#else` section.
- Static key: the key for an op that only some schedulers implement counts
  the enabled schedulers that implement it, so cpu-form roots and
  uninvolved schedulers pay one branch. The count pairs on the `has_op`
  bits, the state
  [code/kernel-idioms](../code/kernel-idioms.md#static-keys-counted-per-user)
  asks for. Add a selftest whose sub-scheduler attach is rejected, followed
  by a phase that must still produce output.
- Names: the key is `__scx_<x>_enabled`, the count `scx_nr_<x>`, and the
  function that calls the op on the schedulers `scx_<x>_notify_<op>()`, as
  in `__scx_ops_cid_sched_updated_enabled`, `scx_nr_ops_cid_sched_updated`
  and `scx_cid_notify_sched_updated()`. Other existing keys are
  `__scx_has_subs` and `__scx_tid_to_task_enabled`.
- The error report for obviously bad input from BPF
  ([code/kernel-idioms](../code/kernel-idioms.md#defensive-code-and-warns))
  is `scx_error()`.
- Prose calls an instance of `struct scx_sched` a sched.

### Sub-schedulers
- A parent's `ops.sub_attach()` for a cgroup never runs before
  `ops.sub_detach()` of that cgroup's previous sub-scheduler has returned.
  The enable mutex doesn't order them: `scx_sub_disable()` unlinks the
  sub-scheduler and drops `scx_enable_mutex` before it calls
  `ops.sub_detach()`. The sysfs node does. Each sub-scheduler's kobject is
  `sub-<cgroup id>` in its parent's `sub` kset, added before
  `ops.sub_attach()` and removed at the end of `scx_sub_disable()`, after
  `ops.sub_detach()`, `ops.exit()` and the RCU wait. An attach that comes
  too early fails with -EEXIST and a sysfs duplicate-filename warning.
- The enable mutex stays off the callbacks. Disabling is a chain in which
  an ancestor's `drain_descendants()` waits for its children to unlink,
  and a BPF callback of unbounded length under the mutex would stall every
  enable and disable.
- A parent therefore needs no instance identity in its records: the
  cgroup ID identifies a sub-scheduler, and cgroup IDs are never reused.
  Detach callbacks for different cgroups run concurrently with each other
  and with attaches on other cgroups. A record changes owner only after its
  cgroup is gone, so the only late arrival for an old record is its own
  detach, which finds it by cgroup ID with a plain compare and clear.
  Don't write detach code or comments that assume an attach raced it.
- A scheduler may always preempt its own tasks and its sub-schedulers'
  tasks. `SCX_CAP_PREEMPT` is needed only to preempt a task outside its
  subtree, an ancestor's or an unrelated scheduler's. The kernel counts a
  refused preempting kick in `SCX_EV_SUB_PREEMPT_DENIED` and degrades it to
  a plain reschedule. A sub-scheduler that preempts only its own tasks
  never consults the cap, and checking the cap for that preemption is
  wrong.
- `SCX_CAP_PERF` is needed for the CPU performance writes on each cid.
- `SCX_CAP_ENQ_IMMED` alone is the base cap. The kernel still calls the
  sub-scheduler's `ops.dispatch()` on such a cid. An insert there with
  `SCX_ENQ_IMMED` succeeds, and an insert without it bounces for the
  missing `SCX_CAP_ENQ`. A sub-scheduler with no task to insert with
  `SCX_ENQ_IMMED` leaves the cid unused and returns from dispatch at once.

### BPF scheduler code
- Write every new loop in a scheduler's BPF code, in the scx repository
  and in tools/sched_ext and its selftests, with
  `bpf_arena_for(var, start, end)` from common.bpf.h, counted walks and
  bounded retry loops alike, because `bpf_for()` calls
  `bpf_iter_num_next()` on every iteration. A retry loop counts its tries
  against a bound. Keep
  `bpf_for()`, `bpf_repeat()` or `bpf_loop()` only where a conversion was
  shown to exceed the verifier's limits, with a comment saying why. Leave
  existing loops alone unless the patch touches them, and never hand-roll a
  loop on a `const volatile` zero.
- `bpf_arena_for()` is not free for the verifier. Converting some nested
  scans in a large scheduler's select path took it past the 1M instruction
  limit until the helpers the loops called were made global
  ([global functions](../subsys/bpf.md#global-functions)). Test a
  conversion on the real kernel
  ([subsys/bpf](../subsys/bpf.md#always-on-rules)).
- Take time from `scx_bpf_now()`, never `bpf_ktime_get_ns()`. Under the rq
  lock it returns the cached rq clock without reading the clocksource.
  Outside it, such as in a timer or a syscall program, it returns a fresh
  rq clock, and every program type that scx registers its common kfuncs
  for can call it. Using it for every timestamp a scheduler compares keeps
  them on one clock. Use `bpf_ktime_get_ns()` only for a value that must
  match CLOCK_MONOTONIC in userspace, and say so.
- common.bpf.h defines `BPF_NO_KFUNC_PROTOTYPES`, which drops vmlinux.h's
  kfunc prototypes, and declares the generic kfuncs itself. Add a new one
  there, not in a scheduler.
- The enum values that the compat headers declare as `const volatile`
  globals, such as `__SCX_ENQ_*` and `__SCX_DSQ_*`, and scx's
  `scx_lib_features` are read-only data the loader sets at open time.
  Measure verifier cost with the values the loader sets
  ([subsys/bpf](../subsys/bpf.md#finding-where-the-instruction-budget-goes)),
  because their zero defaults fold whole paths away.
- State that another CPU reads as more than one word takes the arena
  spinlock, `scx_spin_lock()`, per object on hot per-CPU paths
  ([code/kernel-idioms](../code/kernel-idioms.md#lockless-is-for-one-word)).
- A lock taken under the rq lock in a struct_ops callback nests inside
  IRQs off. Every other holder, a syscall program included, takes it with
  `scx_spin_lock_irqsave()`. Otherwise an IRQ on the holder's CPU that
  wakes a task deadlocks against the callback until the arena lock's
  timeout aborts the scheduler.
- Index a BSS array at a variable offset through `MEMBER_VPTR()`, such as
  `MEMBER_VPTR(layers, [id].node[src].xnuma[dst])`. The plain expression
  often fails verification.
- A port or a conversion of a scheduler keeps its scheduling behavior.
  Diff the decision logic path by path against the original: pick orders,
  spreading, tie-breaks and vtime handling. Keep the semantics exactly,
  such as a per-CPU rotor that mirrors `bpf_cpumask_any_distribute()`, and
  call out every unavoidable difference in the description. A fallback
  once replaced spreading with a first-set-bit pick, and only a review
  caught it.

### Reviewing schedulers
These focus areas add to the review in
[review/patchset-review](../review/patchset-review.md#running-a-milestone-review).

- Affinity: a mask that isn't intersected with `p->cpus_ptr`, such as a
  per-node mask, serves only tasks allowed on all CPUs, inside the branch
  that tested that. Tasks with restricted affinity use per-task masks
  intersected with `p->cpus_ptr`.
- `scx_bpf_pick_idle_cpu()` picks from the mask it is given and doesn't
  check the task's affinity. A CPU picked from a mask that isn't per-task
  is checked with `bpf_cpumask_test_cpu(cpu, p->cpus_ptr)`. Kernel threads
  bound to one node, such as kcompactd, are the usual victims.
- Accounting: a budget is charged where the event happens, such as in
  `ops.running()` on the destination CPU, once per event, with the right
  source and destination and the actual duration. A check before the event
  only peeks at the budget and never deducts. Its result isn't cached
  across loop iterations, because each destination has its own budget.
- A task that keeps running across a dispatch, picked again with no switch,
  gets neither `ops.stopping()` nor `ops.running()`. Accounting that must
  see all of its runtime also runs from `ops.tick()`.
- Topology: assumptions hold with three or more NUMA nodes, such as sticky
  flags, iteration order and the independence of per-node state.
- Verifier safety: variable-indexed BSS arrays are reached as
  [above](#bpf-scheduler-code), and array accesses in loops are
  bounds-checked.
- Direction: source and destination stay consistent across the push path
  (idle CPU selection), the pull path (consuming from a DSQ) and
  preemption.

### Bug patterns
For [debug/bug-report-triage](../debug/bug-report-triage.md#bug-patterns):

- A nested op call clears per-CPU context. The op-call macros record the
  locked rq in a per-CPU variable for the duration of the call. A wrapper
  that clears the variable on exit, instead of restoring the previous
  value as `SCX_CALL_OP_RET()` does, breaks the outer call when a nested
  call goes through it.
- A kfunc keys off the root scheduler, `scx_root`, where the calling
  scheduler belongs, while sub-schedulers exist. Check every use of
  `scx_root` in kfunc bodies.
- An op called from a function that holds the rq lock passes NULL as the
  locked rq. A callback that takes the rq lock again then deadlocks. Check
  every op call that passes NULL for the locked rq.

### Testing schedulers
- Check that the configuration the scheduler loaded is the one intended,
  before trusting a run. A config parser can ignore a field at the wrong
  nesting level and use its default without an error. Cross-check the
  fields that matter against the stats output.
- Exercise restricted affinity, with taskset or cpusets, as well as tasks
  allowed on every CPU. A load of unrestricted workers never runs the
  restricted-affinity paths.
- For a scheduler that works on cgroups, enable the cpu controller, not
  only cpuset, in `cgroup.subtree_control` along the path. sched_ext's
  cgroup hooks run from the cpu controller, so without it
  `ops.cgroup_init()` never runs for those cgroups. Start a scheduler that
  discovers cgroups as they appear before creating them.
- An x86 QEMU guest with `-smp 16,sockets=2,cores=4,threads=2` has two
  LLCs, CPUs 0-7 and 8-15.
- `SCX_SLICE_DFL` is 20ms. A workload sized for 20us slices is about 1000
  times too heavy for it
  ([test/perf-comparison](../test/perf-comparison.md#ab-latency-comparison)).
- Measure what the mechanism changes. `SCX_ENQ_IMMED` keeps tasks from
  being stranded on local DSQs, so it is measured by the time tasks spend
  on a local DSQ: from the insert, `dispatch_to_local_dsq()` for a
  dispatch and `scx_dispatch_enqueue()` for a direct dispatch, to
  sched_switch or the re-enqueue. Check the functions' signatures and that
  they aren't inlined in the tree under test.
- Test pure logic of a Rust scheduler without BPF: move the computation
  into free functions that don't touch the skeleton, test them in
  `#[cfg(test)]`, and run `cargo test -p <crate> --bin <binary> <filter>`.

### Tracing schedulers
These add to
[debug/event-tracing](../debug/event-tracing.md#set-up-a-trigger-ring).

- Name probe stubs `scx_dbg_<event>`. The debug commit with the rings and
  probes goes against kernel/sched/ext/ext.c or the file that holds the
  path.
- Record `dsq->nr` in every DSQ insert, removal and pick event, and the rq
  state flags. Without the count, whether a DSQ was empty at insert, which
  decides whether re-enqueue triggers fire, has no answer.
- Triggers that worked: `dispatch_to_local_dsq()` failing its cpumask
  check, a task that isn't exiting staying on a local DSQ for more than
  1ms, and `curr=swapper` on a rq with runnable tasks for more than 5s.
- A local DSQ latency investigation needed about 20 events: DSQ insert and
  removal with the count, every re-enqueue trigger, sched_switch, IRQ and
  softirq entry and exit, the reason bits of local DSQ inserts,
  sched_wake_idle_without_ipi, `smp_send_reschedule()`, reschedule IPI
  entry and exit, `cpuidle_select()` entry and exit, and probes inside
  whichever function held the gap.
- Filter exiting tasks first. Unless the scheduler sets
  `SCX_OPS_ENQ_EXITING`, a `PF_EXITING` task skips `ops.enqueue()` and goes
  to the local DSQ directly, so it never gets `SCX_ENQ_IMMED` and may sit on
  a local DSQ by design.
- In a stalled machine that SysRq-S
  ([debug/crash-debugging](../debug/crash-debugging.md#machine-stall))
  didn't recover, ask which DSQs hold the stuck tasks and what the sched_ext
  watchdog reported.

### The scx repository
- The repository's AGENTS.md is authoritative for syncing the shared
  headers with the kernel, updating vmlinux.h, releases, dependency
  updates, patch descriptions and pull request descriptions. Follow it, and
  change it there, not in an [overlay](../core/overlays.md#recording), when
  the procedure changes. Opening and updating pull requests follows
  [submit/github-pr](../submit/github-pr.md#opening-or-updating-a-pull-request).
- The rest of this section and the
  [commit gate](#before-a-commit-of-rust-code) apply to Rust code in any
  repository. When the work is kept with another patch tool, the gate
  applies to the commits that import it onto the pull request branch.
- Build in debug mode while iterating. Build with `--release` only when
  asked, or for final testing and deployment.
- Keep the dependencies in each Cargo.toml in alphabetical order.
- Rust idioms:
  - Reach the value in an `Arc<Mutex<T>>` with `&mut *guard`.
  - Resolve borrow conflicts with explicit `drop()` calls or scoped guards.
  - Take one guard for several accesses under the same lock, before the
    first use, and drop it after the last.
  - Prefix unused bindings with `_`.
  - Use `bail!()`, not `return Err(anyhow!())`, and
    `.with_context(|| format!("..."))` with `?`.
  - Check the existing imports before adding one.
  - In a crossbeam `select!`, create `never()` before the block.

## Customization
- The overlay holds the user's remotes and branch prefix in the scx
  repository, the reviewers to request on each scheduler's pull requests,
  the location of the flag decoder, test machines, and notes on the
  schedulers the user works on.
