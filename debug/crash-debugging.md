# Crash debugging

- **Load when:** a test machine or VM hangs, stalls or crashes, a crash
  dump is to be triggered, saved or analyzed with drgn, a machine is to be
  power-reset, a hung VM's memory is to be dumped, or a bug report is to be
  written about a crash.
- **Parameters:** `$MACHINES`, `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [core/safe-operations](../core/safe-operations.md),
  [test/machine-access](../test/machine-access.md),
  [test/remote-testing](../test/remote-testing.md),
  [test/vm-testing](../test/vm-testing.md)

## Always-on rules
- SysRq-c keeps a hung machine's state through kdump, and a power reset
  destroys it. Never use one in place of the other, because the choice
  decides whether the failure can be analyzed.
- [GATE] Before triggering a crash with SysRq-c: the console monitor is
  capturing and kdump is loaded, because the crash ends the running kernel,
  and the console log and the dump are then the only records of its state.
- [GATE] Before a power reset: the console log was saved and reviewed, a
  crash dump was captured or can't be triggered, and the user accepted
  losing the diagnostic data, because a power reset destroys the state
  that explains the failure ([what counts](#power-resets)).
- [GATE] Before analyzing a crash dump: drgn opens the vmcore, the vmlinux
  is from the same build as the crashed kernel, and dmesg was extracted
  first, because a mismatched vmlinux gives wrong types and stacks without
  an error.

## [GATE] Checklist
### Before triggering a crash with SysRq-c
- [ ] The console monitor is running and capturing
      ([test/remote-testing](../test/remote-testing.md#console-monitoring)).
- [ ] kdump is loaded: `/sys/kernel/kexec/crash_loaded` reads 1, or
      `/sys/kernel/kexec_crash_loaded` on kernels before 6.19. Without
      kdump, the power reset gate applies ([power resets](#power-resets)).
- [ ] When the key goes through the console, kernel.sysrq allows it: 1
      enables all keys
      ([SysRq through the console](#sysrq-through-the-console)).
- [ ] The action is the crash trigger, not a power reset.

### Before a power reset
- [ ] The console log was saved and reviewed.
- [ ] A crash dump was captured, or a crash can't be triggered because the
      machine is fully dead.
- [ ] The user explicitly accepted losing the diagnostic data.

### Before analyzing a crash dump
- [ ] The vmcore is valid and readable: `drgn -c <vmcore>` opens it without
      an error.
- [ ] The vmlinux is from the same build as the crashed kernel.
- [ ] dmesg was extracted from the vmcore as the first step.

## Procedure
### Machine stall
When ssh hangs or a run on a test machine makes no progress:

1. Read the console log first: a panic, an RCU stall, a soft lockup or a
   hung task report. Never power-reset blindly.
2. Only if a BPF scheduler (sched_ext) is running: SysRq-S ejects it. Send
   it with the SysRq mechanism `$MACHINES` names for the machine
   ([test/machine-access](../test/machine-access.md#sending-sysrq)), wait
   about 15 seconds and check the console. If the machine recovers, kill
   the load generators with SIGKILL. Go on only when the machine stays
   unresponsive.
3. Check the gate, then trigger SysRq-c: through /proc/sysrq-trigger when a
   shell still answers, otherwise through the console with the mechanism
   `$MACHINES` names
   ([test/machine-access](../test/machine-access.md#sending-sysrq)). Never a
   power reset.
4. Watch the console log while the crash kernel boots, instead of waiting
   a fixed time. A distribution's kdump service may save the dump and
   reboot by itself. Then copy the dump from the service's dump directory
   and skip to step 6.
5. When the crash kernel stays up, log in over ssh and save the dump to the
   machine's persistent directory
   ([test/remote-testing](../test/remote-testing.md#test-machines)). A
   vmcore can't be replaced.

   ```
   makedumpfile -c -d 31 /proc/vmcore <dir>/vmcore-$(date +%Y%m%d-%H%M%S).dmp
   ```

   `-c` compresses with zlib, and `-d 31` drops zero, cache, user and free
   pages. Without makedumpfile, `cp /proc/vmcore <dir>/vmcore.raw` saves an
   uncompressed copy.
6. Copy the dump to `$SCRATCH` on the workstation, such as with
   `scp box1:<dir>/vmcore-<time>.dmp "$SCRATCH/<topic>/"`.
7. Reboot the machine only after the dump is copied off.
8. Analyze the dump ([analyzing a crash dump](#analyzing-a-crash-dump)).
   The key questions: where the tasks are stuck, which run queues hold
   them, what the stall watchdogs reported, and which locks are held.

kdump must be loaded before the test starts, because a hung machine can't
load it anymore. It is loaded with the mechanism `$MACHINES` names
([test/machine-access](../test/machine-access.md#crash-dumps)), and the test
gates of
[test/remote-testing](../test/remote-testing.md#before-starting-a-test-on-a-test-machine)
check it.

### Analyzing a crash dump
1. Check the gate.
2. Run drgn directly, with no wrapper:
   `drgn -c <vmcore> -s <vmlinux> -e '<python>'`. drgn's Linux helpers,
   such as `per_cpu()` and `print_dmesg()`, are in scope.
3. Extract dmesg first ([drgn notes](#drgn-notes)).
4. Find the CPU that panicked in dmesg.
5. Get each CPU's backtrace through the task running on it
   ([drgn notes](#drgn-notes)).
6. For a lock that a CPU waits on, read its owner CPU.
7. Scan every CPU's backtrace for the paths of interest, and every task's
   for stuck tasks, such as tasks with mutex or rwsem frames.

### QEMU guest memory dumps
For a hung VM, dump the guest's memory through the QEMU monitor and analyze
it like a vmcore.

1. The VM must have been started with a monitor socket. How to send monitor
   commands, and which others help with a hung VM, is in
   [test/vm-testing](../test/vm-testing.md#monitor-socket).
2. On a kernel without KASAN, run `dump-guest-memory -p <file>` with the
   file under `$SCRATCH`. It writes an ELF dump with paging.
3. On a KASAN kernel, `-p` walks the KASAN shadow page tables for minutes
   and leaves a 0-byte file. Run a plain `dump-guest-memory <file>`
   instead. With `-device vmcoreinfo` given when QEMU started, it dumps in
   seconds and drgn reads it. Without vmcoreinfo a plain dump may be
   unusable, which is why `-p` stays the choice on other kernels.
4. Analyze it with drgn and the vmlinux of the build the VM ran
   ([analyzing a crash dump](#analyzing-a-crash-dump)).

## Reference
### Power resets
The power reset gate covers a power or IPMI reset and anything else that
drops the running state of a hung or crashed machine. SysRq-c without kdump
loaded is one of them: it drops the state and saves nothing.

### SysRq through the console
- A SysRq key sent through the console
  ([test/machine-access](../test/machine-access.md#sending-sysrq)) is
  checked against kernel.sysrq. Writing the key to /proc/sysrq-trigger
  bypasses the check.
- kernel.sysrq 1 enables every key. A bitmask enables groups: SysRq-c
  needs the dump bit (8), and SysRq-S needs the nice bit (0x100).
  systemd's default of 16 blocks both.
- Set kernel.sysrq on each test machine during setup, before any hang,
  because a hung machine can't change it.

### drgn notes
- `prog.stack_trace(N)` is the backtrace of the task with PID N, never of
  CPU N. For CPU N, take the task running there:

  ```
  prog.stack_trace(per_cpu(prog["runqueues"], 15).curr)
  ```

- dmesg: `print_dmesg()` prints it. `get_printk_records()` returns records
  whose message is the `.text` attribute, never `r["text"]`, and it holds
  bytes that need decoding:

  ```
  for r in get_printk_records(): print(r.text.decode("utf-8", errors="replace"))
  ```

- Lock owners: with CONFIG_DEBUG_SPINLOCK, a raw spinlock records its
  owner CPU:

  ```
  Object(prog, "raw_spinlock_t", address=0xaddr).owner_cpu
  ```

- Scanning: walk the CPUs or the tasks and check each backtrace's frames:

  ```
  for cpu in for_each_online_cpu(): st = prog.stack_trace(per_cpu(prog["runqueues"], cpu).curr)
  for t in prog.threads(): frames = [str(f) for f in t.stack_trace()]
  ```

- rhashtable: bit 0 of a bucket pointer is a lock bit. Mask it off, and a
  bucket that is then NULL is empty. Follow the chain until a pointer with
  bit 0 set, the nulls marker, and get each object from its `rhash_head`
  with `container_of()`.

### Bug reports
A bug report states facts only. It makes no diagnosis that can't be proven
([core/evidence-discipline](../core/evidence-discipline.md#always-on-rules))
and keeps what happened apart from any analysis. Its sections, in order:
1. Summary: one sentence on what happened, not why.
2. System: exact versions of the kernel, the affected packages and drivers,
   and the hardware involved.
3. Reproduction: the minimal steps.
4. What happened: the observable facts, with exact timestamps.
5. Stack trace: with offsets if there are no symbols.
6. Duplicate search: where the reporter searched and what was found.

For a userspace crash, the systemd tools give the trace and the log, with
symbols from the distribution's debuginfod server set in `DEBUGINFOD_URLS`:

```
coredumpctl gdb <pid> --debugger-arguments="-batch -ex 'bt full'" > bt.txt 2>&1
journalctl -b | grep -i crash
journalctl --since "<YYYY-MM-DD HH:MM:SS>"
```

Use precise words: "the process received SIGABRT", not "it crashed";
"the crash occurred during ...", not "likely due to ...". Give exact
timestamps, never "immediately", and quote messages verbatim. If an AI
assisted with the report, say so in a note.

## Pitfalls
- A vmcore is the state after the panic. CPUs may have moved on from a
  deadlock by the time the dump was taken.

## Customization
- `$MACHINES` names each machine's SysRq, crash dump and power mechanisms,
  and the machine-access overlay holds their details. This module's overlay
  names the persistent directory for dumps.
- The overlay names the distribution's debuginfod URL and its package
  version command for bug reports.
- `$SCRATCH` holds copied dumps and guest memory dumps.
