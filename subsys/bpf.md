# BPF programs

- **Load when:** writing, reviewing or debugging BPF programs or their
  userspace loaders, such as struct_ops programs, SEC("syscall") programs,
  kptrs, arenas and BPF locks, or when a program fails to load, exceeds the
  verifier's instruction or stack limit, or needs veristat.
- **Parameters:** `$SCRATCH`
- **Depends on:** [review/kernel-build](../review/kernel-build.md),
  [test/remote-testing](../test/remote-testing.md),
  [test/vm-testing](../test/vm-testing.md)

The verifier facts here hold for mainline v7.3's kernel/bpf, libbpf and
Documentation/bpf, and a fact that changed in a recent release names that
release. When a load behaves differently, check the tree the program runs
on.

## Always-on rules
- Judge a change to a BPF program's verifier cost, its instruction count or
  its stack size, only by loading the real object on the target kernel
  with the loader's settings, never from the source or a standalone
  reproduction, because state pruning and load-time constants decide the
  cost, and those proxies have mispredicted it repeatedly.
- [GATE] Before deploying BPF changes to a test machine: the BPF tools and
  programs were rebuilt clean after the kernel, the build and console gates
  passed, and the verifier log of any failed load was read, because stale
  generated headers and skeletons fail in code the change never touched.

## [GATE] Checklist
### Before deploying BPF changes to a test machine
- [ ] The kernel and every tool the change touches passed the gate in
      [review/kernel-build](../review/kernel-build.md#before-testing-posting-or-applying-a-code-change).
- [ ] The BPF tools and programs were rebuilt clean after the kernel. This
      adds a case to the clean builds of
      [review/kernel-build](../review/kernel-build.md#procedure) step 8,
      because of
      [stale BPF tool builds](../review/kernel-build.md#stale-bpf-tool-builds).
- [ ] The console monitor is capturing, as
      [test/remote-testing](../test/remote-testing.md#before-a-kexec-reboot-power-cycle-or-kernel-deploy)
      requires.
- [ ] When a load failed, its verifier log was read before anything else
      changed ([debugging a failed load](#debugging-a-failed-load)).

## Procedure
### Debugging a failed load
1. Read the verifier log before anything else. The verifier rejects a
   program with EACCES or EINVAL. Its EPERMs come from capability checks.
2. Get the whole log. A loader's printed log can be cut short, and level 1
   shows only the path that failed. Give the failing program its own
   buffer with `bpf_program__set_log_buf()` at level 2 and write the buffer
   to a file under `$SCRATCH`. Keep the buffer under 1 GiB, because the
   kernel rejects a larger one with EINVAL. When the log overflows the
   buffer, the load fails with ENOSPC, even when verification passed. The
   kernel keeps the tail of the log by default and the head with
   BPF_LOG_FIXED (veristat's `--log-fixed`).
3. Match the error:
   - EACCES with "Possibly NULL pointer passed to trusted ...": a pointer
     that may be NULL, such as a kptr read from a map, reached a kfunc
     argument that must not be NULL ([kptrs](#kptrs)).
   - "variable ctx access var_off=... disallowed" or "dereference of
     modified ctx ptr ... disallowed": a context access at a variable
     offset, or through a context pointer moved by a constant
     ([context pointers](#context-pointers)).
   - "addr_space_cast insn can only be used in a program that has an
     associated arena": [arenas](#arenas).
   - "combined stack size of N calls is X. Too large":
     [fixing a stack size error](#fixing-a-stack-size-error).
   - "BPF program is too large. Processed N insn": [the instruction
     budget](#finding-where-the-instruction-budget-goes).
   - EINVAL when a SEC("syscall") program runs, not when it loads: the
     context buffer is too small or too large
     ([syscall programs](#syscall-programs)).

### Finding where the instruction budget goes
1. Load the object on the target kernel, in a VM or on a test machine
   ([test/vm-testing](../test/vm-testing.md)). veristat needs a kernel
   whose BTF has every kfunc the object calls, so it doesn't run on a
   workstation with another kernel ([veristat](#veristat)).
2. Load with the read-only data the loader sets at open time. The source
   defaults of `const volatile` globals are often zero, and the verifier
   folds every path they disable away, so the count comes out far too low.
   Run the program in a configuration that loads, find its `.rodata` map
   with `bpftool map list`, dump it with `bpftool -j map dump name
   <name>`, and load the object with that blob as the `.rodata` initial
   value (`bpf_map__set_initial_value()` before the load). Mainline
   veristat has no option for that, so it takes a small patch to veristat
   or a loader of your own. veristat's `-G` then overrides single globals.
3. Read the per-subprogram stats. Since v7.3 (c2e6c7de8830 ("bpf: Show
   more useful info in stack depth stats")), level 4 ends the log with one
   line per subprogram, `subprog N (name) main|static|global insns_self X
   insns_total Y stack Z`, and sorting on insns_total names the costly
   static helper. veristat with `-v` always requests them: `-v -l 1
   --log-size <bytes> --log-fixed`. A loader requests them with
   `bpf_program__set_log_level()` at 4.
4. For a static helper that is verified many times, take a level 2 log
   (`-l 2`). For a large program it runs to hundreds of megabytes, so give
   the VM the memory. The log shows every call with the caller's state and
   a `N: safe` line for each pruned arrival. Count the visits per call
   target and the distinct caller states.
5. The caller dump leaves out r1-r5 and dead stack slots. When visits
   repeat with identical-looking states, a temporary verifier patch that
   prints both states at the first failed equivalence check names the
   register. veristat `-t` checkpoints at every prune point, which tells
   "no state was recorded" from "the states differ".
6. For cost per source line, use the `; <source> @ file.c:NNN` annotations
   of the level 2 log, or veristat `-S N` for the N most frequent lines.
7. Apply the fixes in [verifier complexity](#verifier-complexity) one at a
   time and measure each.

### Fixing a stack size error
check_max_stack_depth() sums the deepest stack access of each subprogram
along every call chain, each rounded up to 16 bytes with a JIT and 32
without, against 512 bytes. The may_goto slot is added after the check and
doesn't count.

Programs on a private stack are checked per frame instead: each subprogram
frame of 64 bytes or more gets its own stack and fails alone, with "stack
size of subprog N is X. Too large". Kprobe, tracepoint, perf event and raw
tracepoint programs get one where the JIT supports it, and so do struct_ops
programs that request one. sched_ext requests it for `ops.dispatch()` and
`ops.sub_caps_updated()` when sub-schedulers are configured.

1. Model the chains from the object before using a VM: run
   `llvm-objdump -dr --no-show-raw-insn`, take each function's deepest
   `r10 - 0x..` offset, resolve calls within a section relatively (target =
   instruction index + 1 + imm) and calls across sections by their
   `R_BPF_64_32` relocation, ignore calls to undefined symbols, which are
   kfuncs, and walk the call graph from each program. Such a model has
   reproduced the kernel's numbers exactly.
2. Move scratch-heavy helpers out of line, so that their frames return
   before the deeper calls: a wrapper around a large inline scan that runs
   before a deeper call, `__noinline` on mask preparation. Inlining a
   helper into a frame that stays live under deeper calls only adds to the
   total.
3. Measure each candidate. Frame sizes predicted by hand mislead: one
   `__noinline` grew its caller's frame.

## Reference
### Syscall programs
- A SEC("syscall") program takes a plain function signature, because its
  context is the caller's `context_in` buffer itself:

  ```c
  SEC("syscall")
  int my_prog(struct my_arg *arg)              /* arg points at context_in */
  ```

  `BPF_PROG(my_prog, struct my_arg *arg)` passes the buffer's first 8 bytes
  as the pointer, as for a tracing context, and the verifier rejects the
  dereference. BPF_PROG works only for programs that take no argument.
- `bpf_prog_test_run_syscall()` copies the buffer in and runs the program
  on it. It rejects a buffer shorter than the end of the farthest context
  access the verifier saw, or longer than U16_MAX, with EINVAL when the
  program runs.
- The skeleton finds a program by its ELF symbol name, whichever form the
  program uses. In the scx repository, `arena_init()` in lib/arena.bpf.c
  takes an argument, and scx_layered's `refresh_layer_cpumasks` is a
  zero-argument BPF_PROG.
- From Rust with libbpf-rs:

  ```rust
  let input = ProgramInput {
      context_in: Some(unsafe { plain::as_mut_bytes(&mut arg) }),
      ..Default::default()
  };
  let output = skel.progs.my_prog.test_run(input)?;
  ```

### Context pointers
- Program types whose context accesses the verifier rewrites at load time,
  through a per-field offset table, need constant offsets from an
  unmodified context pointer. A variable offset fails with "variable ctx
  access var_off=... disallowed", and a pointer moved by a constant fails
  with "dereference of modified ctx ptr ... disallowed".
- Since v7.1, SEC("syscall") programs may access their context at variable
  offsets up to U16_MAX (ae5ef001aa98 ("bpf: Support variable offsets for
  syscall PTR_TO_CTX")), and program types without the rewrite accept a
  pointer moved by a constant (de6c7d99f898 ("bpf: Relax fixed offset
  check for PTR_TO_CTX")). Before v7.1, both errors applied to
  SEC("syscall") too: field accesses at constant offsets worked, and an
  array index in a loop didn't.
- For kernels before v7.1, copy an array out of the context with a
  constant-size `__builtin_memcpy()`. Clang emits one constant-offset load
  per element, and each is checked on its own. Indexing every element with
  a constant works too, as `arena_topology_init()` in scx's lib/arena.bpf.c
  does.

### kptrs
- kptrs can't live in a map that userspace mmaps, such as .bss: the kernel
  refuses to mmap a map whose value has kptrs or other special fields. Put
  them in a `SEC(".maps")` array map.
- Every kptr read from a map may be NULL: an RCU pointer inside an RCU
  read section, an untrusted one outside. Observed: a NULL check didn't
  carry across `bpf_for()` iterations. Read and check again in each
  iteration.
- An update of a whole map value, from userspace or with
  `bpf_map_update_elem()`, loses the value's kptrs. A hash map update
  without BPF_F_LOCK puts in a new element. An array map update released
  the kptrs before v7.2, and since a3a81d247651 ("bpf: Cancel special
  fields on map value recycle") it leaves them in place. Initialize the
  plain fields before BPF programs allocate the kptrs, and change plain
  fields of a value with kptrs afterwards only from BPF.
- To change those fields from userspace, write the new values into .bss
  staging arrays, then run a SEC("syscall") program with test_run that
  copies them into the map values.

### Arenas
Examples: tools/sched_ext/scx_sdt.bpf.c and
tools/testing/selftests/bpf/progs/arena_*.c.

- An arena is a `BPF_MAP_TYPE_ARENA` map with `BPF_F_MMAPABLE`. The
  optional `map_extra` fixes its userspace address: `0x1ull << 44` on
  x86-64 and `0x1ull << 32` on arm64 in the examples. `max_entries` bounds
  the page count. Pages come from `bpf_arena_alloc_pages()` or from a
  userspace fault on the arena.
- Fixed-size data is a global declared `struct foo __arena bar;`. libbpf
  places it in the `.addr_space.1` section and initializes it, and
  userspace reaches it as `skel->arena->bar`. It needs no
  `bpf_arena_alloc_pages()`.
- Data sized at run time comes from `bpf_arena_alloc_pages(&arena, NULL,
  nr_pages, NUMA_NO_NODE, 0)`, with the pointer kept in an `__arena`
  field.
- With clang's `__BPF_FEATURE_ADDR_SPACE_CAST`, `__arena` pointers convert
  between address spaces by themselves and `cast_kern()` and `cast_user()`
  are no-ops. Exception, measured: clang 19 to 21 compile
  `__builtin_memset()` on the address of an `__arena_global` object into
  stores through the raw address with no `addr_space_cast`, and the
  verifier rejects them with "R1 invalid mem access 'scalar'". clang 22
  emits the cast.
- A program that reaches arena memory only through pointers kept
  elsewhere, such as in task storage, must reference the arena map, or it
  fails with "addr_space_cast insn can only be used in a program that has
  an associated arena". `asm volatile("" :: "r"(&arena));` makes the
  reference. Put it in the shared lookup helper so that every caller gets
  it, as `QMAP_TOUCH_ARENA()` in tools/sched_ext/scx_qmap.bpf.c does.
- Atomic instructions on arena memory need JIT support. x86-64 rejects
  the fetch forms of AND, OR and XOR, such as `__sync_fetch_and_or()` with
  its result used, and arm64 needs LSE atomics. The verifier reports
  "BPF_ATOMIC stores into R<n> arena is not allowed". A word that needs one
  of those operations goes in .bss on x86-64. Other atomics work on arena
  memory, and scx's arena spinlock lives in the arena. A counter with one
  writer can use a plain `++`.
- Never mark shared or arena fields `volatile`. It stops compiler caching
  but gives no atomicity or ordering. Use the compiler's atomic builtins:
  `__sync_fetch_and_add()` and `__atomic_load_n(..., __ATOMIC_RELAXED)`
  for statistics. A flag handshake between contexts sets the flag with
  `__sync_fetch_and_or()` and reads and clears it with
  `__sync_fetch_and_and(&flag, 0)`. Plain READ_ONCE() and WRITE_ONCE() are
  not enough for a handshake. `const volatile` for configuration the loader
  sets is the one exception.

### Locks
- A map value holds at most one `bpf_res_spin_lock`, so each lock in
  global data gets a section of its own, `SEC(".data.<name>")`. Mark it
  `__hidden`, so that libbpf doesn't make the section mmapable: the kernel
  refuses to mmap a map whose value holds a lock.
- For a lock chosen at run time, have a helper return the lock's pointer,
  keep it in a local variable and pass that variable to both the lock and
  the unlock. The verifier can't match two calls of the helper to the same
  lock.
- Under a lock, a program may call only the list, rbtree and refcount
  kfuncs, the number iterators, the arena and stream kfuncs, the lock
  kfuncs themselves, and the helpers `bpf_spin_unlock()` and
  `bpf_kptr_xchg()`. Since v7.3 (ed3b3093b624 ("bpf: Add KF_SPINLOCK_SAFE
  flag for kfuncs under bpf_spin_lock")), the allowed kfuncs are the ones
  flagged `KF_SPINLOCK_SAFE`.
- The verifier ties the IRQ state saved by `bpf_local_irq_save()` to the
  stack slot it was written to, so the restore must name that slot. A
  cleanup guard can't take the state from a constructor that returns by
  value, because the copy moves it to another slot. Make the constructor a
  macro that passes a compound literal of the caller's block,
  `&(struct g){ .lock = l }`, to an always-inline init function that saves
  into it and returns the pointer. The class type is that pointer, so
  `guard()` and `scoped_guard()` work unchanged. scx's
  scheds/include/lib/spinlock.h builds its irqsave guard this way.

### Global functions
- A helper called from many paths is verified again on every path unless
  it is a true global function: non-static, default visibility and
  `__noinline`. libbpf's `__hidden` does the opposite. bpf_helpers.h says
  it makes a subprogram "effectively static" for verification, so
  `__hidden __noinline` changes nothing.
- A global function is verified once per program, with opaque arguments.
  A struct pointer arrives as sized memory that may be NULL unless the
  argument is `__arg_nonnull`. A kernel pointer stored in that struct
  reads back as a scalar, so pass it as its own `__arg_trusted` argument.
  Arena pointers inside survive, and an arena pointer argument takes
  `__arg_arena`.
- The call site checks a memory argument for both read and write, whatever
  its `const`, which scrubs the pointers the caller spilled into that
  struct. Kernel pointers can't live in a context struct passed to a global
  function.
- Measured on a large scheduler: making ten widely called picker helpers
  global cut two programs' verified instructions by more than an order of
  magnitude. Helpers with one call site changed almost nothing.

### Verifier complexity
- An open-coded `bpf_for()` over a large runtime count, around 1024,
  exceeds the 1M instruction limit unless the loop converges, which needs
  the register state at the loop head to repeat. Two things break that:
  - A loop-carried counter that the verifier tracks precisely, such as a
    `rank++` it follows from 0 to N. Keep the counter local, apply
    `barrier_var()` to it right after the increment so that it is
    imprecise at the loop head, and don't narrow it again with a bound
    check. Take the final count from a popcount, not from the counter. A
    rank can also be computed without a counter, as the popcount of the
    bits below the position.
  - A heavy, branchy or nested loop body. Move the body into a global
    function that takes the index as an argument. The loop then holds one
    opaque call, converges, and the body is verified once. Inside the
    function, apply `barrier_var()` to the arguments and clamp them with
    `if (x < BOUND)`, so that one cheap check proves the array indices.
    Make only the per-iteration body global, not the whole computation,
    which would lose the caller's range narrowing.
- Measured not to help: `bpf_loop()` in place of `bpf_for()`,
  `#pragma unroll`, restructuring to carry no state, and `barrier_var()`
  on the induction variable or at the top of the loop.
- Always-inline helpers are verified again at every call site. In one
  scheduler, most of a 1M instruction rejection was mask word loops
  inlined four to five static frames deep, and the fix that mattered was
  one word loop per intersection instead of separate copy, AND and empty
  checks.
- Removing a NULL test after a lookup that still has a NULL-returning
  branch costs more, because the NULL state then runs the rest of the
  function too.
- Liveness counts r1-r5 as read by every subprogram call, whatever the
  callee takes. Helpers and kfuncs get a call summary with their argument
  count, subprograms don't. When the state recorded at a join has a stale
  but initialized r5 and a later arrival has r5 uninitialized, regsafe()
  fails on the type and the arrival is not pruned. The reverse prunes,
  because regsafe() accepts any register against a recorded one that is
  uninitialized.

### veristat
veristat loads an object's programs on the running kernel and reports the
verifier's statistics.

- Build it in tools/testing/selftests/bpf with `make veristat`. To run it
  on a test machine, copy the binary, any shared libbpf it links (check
  with `ldd`) and the object.
- Use the linked object of a multi-file program. The separate `.bpf.o`
  files lack each other's symbols.
- `const volatile` globals take their source defaults. `-G "var = value"`
  overrides one, and the loader's values come from its rodata
  ([budget](#finding-where-the-instruction-budget-goes)).
- Pass `--log-size` with `-v` or `-S`. Without it, veristat probes for a
  log buffer of almost 1 GiB and allocates it, which crashes it in a small
  VM.
- Common forms: `veristat obj` for the table, `-v` for the verifier log,
  `-f <prog> -v -l 2` for one program's full log, `-t` to checkpoint at
  every prune point.
- `-f` on an object with an arena map has reported "Processed 0
  programs". Run such objects unfiltered.
- scx's CI has a veristat job that loads the schedulers' objects on
  several kernels.

### Rust loaders
- Convert map values with the `plain` crate, not raw pointer casts:
  `plain::from_bytes::<T>(v.as_slice())` to read,
  `unsafe { plain::as_bytes(&val) }` to write, and
  `unsafe { plain::as_mut_bytes(&mut val) }` for a test_run `context_in`.
  Build a u32 key with `(id as u32).to_ne_bytes()`.
- `unsafe impl Plain for bpf_intf::T {}` goes in the crate that defines
  the type, usually its lib.rs, because of Rust's orphan rule.
- Access maps through the skeleton, `skel.maps.<name>.update()`, `lookup()`
  and `delete()`, with `MapFlags::ANY`.

### struct_ops callbacks
- BTF argument-name suffixes on struct_ops callback arguments, `__nullable`
  and `__ref`, and since v7.3 `__arena` and `__arena__nullable`, go on both
  the CFI stub's parameters and the ops struct member's parameters, and
  always match. The kernel reads only the stub's suffixes, in
  `prepare_arg_info()`, so the match is a convention: the member's
  prototype feeds `btf_distill_func_proto()` and is the API that vmlinux.h
  shows, and a reader trusts it.

### Kfuncs and iterators
- An open-coded iterator's constructor, `bpf_iter_<type>_new()`, always
  initializes the iterator, empty when its arguments are invalid, and the
  destructor always runs (Documentation/bpf/bpf_iterators.rst). A bug
  report that assumes a failed constructor leaves the iterator
  uninitialized misreads that contract.
- When a handle or an iterator can be NULL and BPF programs can reach a
  kfunc, the kfunc checks for NULL on entry. Its companion kfuncs usually
  check already and show the pattern.

### BPF C notes
- `bpf_map_lookup_elem()` returns a pointer or NULL. Check it before use.
- Paths that run later, such as timer callbacks, check again that the map
  entry still exists.
- `p->cgroups->dfl_cgrp` is never NULL.
- State with several conditions goes in a u64 bitmap.
- Configuration that userspace sets goes in `const volatile` globals.
- Keep the code flat.

## Customization
- The overlay names local verifier probe builds, analysis scripts and the
  VMs with the target kernels.
