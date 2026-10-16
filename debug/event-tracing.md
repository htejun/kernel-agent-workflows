# Event tracing

- **Load when:** adding kprobes, bpftrace scripts, debug probe stubs or an
  in-kernel trace ring to see what the kernel does, including hunting a rare
  event in a high-rate stream where the history before the event matters.
- **Parameters:** `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md)

This module covers instrumenting the kernel with bpftrace or an in-kernel
trace ring, and analyzing the dumps round by round until one operation
explains the symptom. For a rare event, each CPU keeps its recent events in a
ring, and a check at the moment of the event dumps every CPU's ring.

## Always-on rules
- When kernel behavior needs a causal answer, add probes and read the data
  instead of reasoning from incomplete data, because a probe round takes
  minutes.
- Keep debug probes, trace rings and triggers in a separate debug commit
  that is never posted or merged, because they are temporary.
- [GATE] Before trusting probe data: confirm that each probe exists in the
  running kernel and sees its arguments, that no output waits for program
  exit, and that ring writers can't collide, because each failure loses or
  corrupts data without an error.

## [GATE] Checklist
### Before trusting probe data
- [ ] Every probe target is in the running kernel's `/proc/kallsyms`. On a
      VM, check inside the guest, because the guest may run a different
      build.
- [ ] For a new probe stub, `nm vmlinux` shows the symbol and the
      disassembly shows the arguments loaded before each call
      ([probe stubs](#probe-stubs)).
- [ ] A probe with zero hits was checked with the ftrace function tracer,
      which tells a broken probe from a function that never runs.
- [ ] bpftrace prints the data that matters from `interval` probes, not only
      from `END` ([bpftrace notes](#bpftrace-notes)).
- [ ] bpftrace ring cursors are keyed by the current CPU and written only
      from kprobe or tracepoint probes, and kernel ring cursors follow
      [ring cursors](#ring-cursors).

## Procedure
### Set up a trigger ring
1. Check that a trigger ring fits ([choosing a tool](#choosing-a-tool)). Use
   a simpler tool otherwise.
2. Define the trigger in one sentence: "dump when X happens". The check runs
   on every event, so make it one comparison. Examples:
   - a cpumask check fails on a dispatch path
   - a task stays queued for more than 1ms without exiting
   - a CPU runs its idle task for more than 5s while it has runnable tasks
3. List every candidate explanation in one sentence with its decisive
   observable: the event, or its absence, that a dump would show. "The
   reschedule IPI entry is absent on the target CPU within 100us of the
   matching `smp_send_reschedule()`" is a hypothesis. "Something in the idle
   path is broken" is not, so refine or drop it. Without this list, one
   candidate gets confirmed while the others are never ruled out.
4. Add one event for every state transition that any hypothesis depends on.
   Over-instrument from the start, because each added probe costs a rebuild.
   A scheduler latency investigation needed about 20 events, covering queue
   changes, context switches, IRQs, softirqs, IPIs and cpuidle.
5. Make each record readable on its own: a timestamp, an event code listed
   in an enum at the top of the debug commit, the pid for filtering, and two
   event-specific arguments. Record the queue length, the state flags and
   every flag or mask that a hypothesis depends on. "Was the queue empty at
   insert?" has no answer if the record left out the length.
6. Add the probes as [probe stubs](#probe-stubs). Starting from an earlier
   investigation's debug commit and renaming its events is faster than
   writing one.
7. Add the dump: on trigger, print every online CPU's ring with
   `trace_printk()` ([dump path and timebase](#dump-path-and-timebase)). Add
   a stop-after-first guard if the trigger is noisy. Save the trace off the
   test machine, so that it survives a crash.
8. Run a workload that reliably reproduces the problem. The trigger should
   fire within minutes. If it doesn't, fix the trigger or the workload
   before adding instrumentation.
9. Analyze the dump ([analyze a dump](#analyze-a-dump)) and write the
   [round report](#round-report).
10. Rebuild with the next probes and repeat from step 8. Each round should
    at least halve the gap. Stop when
    [declare a root cause](#declare-a-root-cause) holds.
11. If the user keeps an investigation log, add one entry per round: the
    dump excerpt, the dead hypotheses, the next probe and why. End the log
    with the debug commits inlined verbatim, so that they survive branch
    rewrites.

### Analyze a dump
1. Filter out the expected cases. The first dump includes events that
   behave as designed, such as exiting tasks that take a shortcut path.
   Write one-line filters for them and work on the rest.
2. Extract the observed facts first, one per line, each quoting the dump
   line it rests on (`cpu=N ts=T ev=E pid=P a1=...`). Quote events instead
   of paraphrasing them.
3. Cluster by timestamp. Outliers on several CPUs that resolve at the same
   moment share one system-wide cause, such as an SMI, a console write or a
   stalled lock holder. Outliers spread evenly in time are per-task.
4. Identify the gap: the start event, the end event and the duration. Count
   the events of each type inside it. Zero poll-exit events during an
   alleged idle-poll stall is a strong lead. A gap whose event mix matches
   the rest of the trace is not the problem.
5. Update the hypothesis ledger. Mark each entry ALIVE, DEAD or MERGED into
   a finer entry, citing the dump line that decided it. An entry leaves the
   ledger only as DEAD or MERGED, never silently.
6. Add the hypotheses the data suggests: from gaps, from surprising or
   missing events, and from splitting a coarse entry into entries tied to
   specific call sites. Each new entry names its decisive observable. When
   every entry is DEAD, the set is incomplete and proves nothing about the
   last entry standing. Seed new entries from the surprising events, the
   absent events and the paths that were not probed.
7. Check every finding against a run that lacks the suspected cause: with
   no instrumentation on the suspect path, with the workload parameter that
   the hypothesis depends on changed, or on another machine or boot
   configuration to catch hardware and firmware effects.
8. Check for drift. Each of these means the investigation is not
   converging:
   - The ledger only shrinks, or holds only seed entries: the candidates
     were pruned, not investigated. Redo step 6 from the dump.
   - One entry stays ALIVE for three or more rounds with no probe aimed at
     its decisive observable. Add that probe or downgrade the entry.
   - A new entry has no decisive observable. Reject it until it has one.
9. Choose the next probes only after steps 1 to 8. A probe either narrows a
   gap, with probes at the entry, the exit and every sub-function of the
   function that holds it, or tells live hypotheses apart. Never add a
   probe for general coverage.

When the trigger fires many times, give each dump to a sub-agent with a
short protocol, such as "classify as A, B or C and report the timestamps and
fields that decided it", and aggregate the results yourself.

### Declare a root cause
1. Declare a root cause only when all of these hold:
   - The current round's ledger has exactly one ALIVE entry.
   - That entry names one primitive operation as the cause, such as one
     slow console write ([pitfalls](#pitfalls)).
   - A dump shows that operation's behavior directly. Being the last entry
     left from the seed list, or "nothing else fits", doesn't count.
2. An A/B run that flips the symptom, or timing that matches the
   operation's physical parameters, corroborates the operation that the
   dump shows. It never replaces the dump
   ([evidence](../core/evidence-discipline.md#what-counts-as-evidence)).
3. Otherwise report the data and the cause as unknown.

## Reference
### Choosing a tool
A trigger ring fits when all four hold:
- The event rate is too high to log every event (millions per second).
- The interesting case is a rare outlier, failure or unexpected path.
- Code in the kernel or in BPF can detect the deviation when it happens.
- The history leading up to the deviation matters, not only the deviation.

Otherwise a simpler tool fits:
- Low event rate: tracepoints or kprobes that print each event.
- No detectable trigger: histograms or `perf record`.
- Only the event itself matters: the ftrace function tracer, filtered to the
  function.

### bpftrace or in-kernel
Start with bpftrace. It needs no rebuild, so a round takes minutes. It
attaches to any function that is not inlined or on the kprobe blacklist, and
to probe stubs added in a quick debug commit.

Switch to an in-kernel ring in a debug commit when:
- bpftrace perturbs the measurement: the outlier depends on latency or
  disappears when bpftrace stops ([pitfalls](#pitfalls)).
- The code paths are out of BPF's reach, such as parts of the idle loop and
  of cpuidle, IRQ-disabled regions without a probeable function, or early
  boot.
- The data must survive a crash of the test machine. With kdump, drgn reads
  the ring from the vmcore. With `ftrace_dump_on_oops`, an oops prints the
  trace buffer to the console.
- The dump must happen atomically with the triggering event, in the same
  context under the runqueue lock or with IRQs off.

### Round report
The round report separates data from reasoning, so that a reader can tell
at a glance which sentences are which. OBSERVED holds facts, each citing a
dump line. INFERRED holds reasoning from those facts. Nothing moves from
INFERRED to OBSERVED without new data that shows it directly. Words such as
"likely", "probably", "appears to", "consistent with", "seems" and "points
to" mark inference and stay in INFERRED. A gap in the data is a probe
target, never a place for "it must be X". The general rules are in
[core/evidence-discipline](../core/evidence-discipline.md#what-counts-as-evidence).

```
Round N: <trigger in one sentence>, dump <file>
OBSERVED
- cpu=3 ts=... ev=... pid=... a1=... a2=...: <what the line shows>
INFERRED
- <reasoning, naming the OBSERVED lines it uses>
LEDGER
- H1 <one sentence>. Decisive: <event or absence>. ALIVE | DEAD | MERGED into Hn (<dump line>)
GAP
- <start event> -> <end event>, <duration>
NEXT PROBES
- <probe>: narrows <gap> | separates H1 from H2
```

### Probe stubs
A probe stub is an empty `__weak noinline` function that bpftrace or a
kprobe attaches to. An empty `static noinline` function loses its calls,
because the compiler sees no side effects. A `__weak` body can be replaced
at link time, so the compiler keeps the calls. `bpf_rstat_flush()` in
kernel/cgroup/rstat.c is an example.

```c
/* the prototype silences -Wmissing-prototypes */
void foo_dbg_my_event(struct task_struct *p, int cpu, u64 val);

__weak noinline void foo_dbg_my_event(struct task_struct *p, int cpu, u64 val)
{
}

/* at the point of interest */
foo_dbg_my_event(p, cpu, val);
```

- Name stubs `<subsys>_dbg_<event>`, so that one grep of `/proc/kallsyms`
  finds all of them.
- Pass all relevant state, such as the task, the CPU, flags and counters,
  which bpftrace reads without further kernel changes. Pack booleans into
  one u32 reason bitfield documented above the stub.
- Place stubs at every decision point, lock boundary and state transition
  on the path. More stubs give a finer timeline.
- Stubs only observe. They have no side effects and no conditionals that
  change control flow.

Verify after every rebuild. For a probe with zero hits, trace the function
with ftrace: hits there mean the kprobe is broken, and no hits mean the
function never runs.

```sh
nm vmlinux | grep foo_dbg_my_event                          # the symbol exists
objdump -d vmlinux | grep -B 10 'call.*foo_dbg_my_event'    # rdi, rsi, rdx... loaded on x86-64
grep foo_dbg_ /proc/kallsyms                                # on the running kernel
echo foo_dbg_my_event > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
```

### Ring cursors
Two writers that share a cursor must not interleave, or their slots collide
and the ring becomes unreadable.

- bpftrace `++` on a map is not atomic. Key the cursor by the current CPU
  and use kprobe or tracepoint probes, which don't nest on one CPU
  (`bpf_prog_active`, kernel/trace/bpf_trace.c). Raw tracepoint probes can
  nest from IRQ context and collide, and a nested kprobe is dropped
  silently.
- Kernel: `this_cpu_inc_return()` is safe against preemption by itself,
  but the task can migrate between it and the `this_cpu_ptr()` lookup of
  the slot. Hot paths under a runqueue lock or with IRQs off are fine.
  Elsewhere, wrap the pair in `preempt_disable()` and `preempt_enable()`.
- Writing another CPU's ring needs an atomic cursor such as
  `atomic_inc_return()` on a per-CPU variable.
- The cursor points at the next slot to write. The oldest entry is at
  `(ri + 0) & MASK` and the newest at `(ri - 1) & MASK`. Print from the
  cursor forward for chronological order.

### bpftrace notes
- Maps and `$` variables need one type across every probe. Cast
  everything, with `(uint64)` the safest. Mixing a kprobe's u64 `arg0` with
  a tracepoint's s32 `args->pid` fails to load with a confusing error.
- Literals default to int64, so `@map % literal` is int64. Write
  `(uint64)(@ri[$cpu] & 31)` instead of `@ri[$cpu] % 32`.
- SIGKILL skips `END`, so `END` is best effort. Print histograms and rings
  from `interval:s:N` probes. In long runs, print progress there too, so
  that the output so far survives a kill or a crash.
- To stop bpftrace from a script, send SIGINT to its PID, wait 3 to 5
  seconds for `END`, then send SIGKILL, and set a flag so that the cleanup
  trap doesn't signal it again. Use the PID, because `pkill bpftrace` hits
  every bpftrace on the machine.

### Dump path and timebase
- Dump with `trace_printk()`, which writes straight into the ftrace ring
  buffer and is safe under the runqueue lock. `printk()` is not: releasing
  the console lock wakes a waiter, and the wakeup takes the runqueue lock.
  The scheduler uses `printk_deferred()` under the runqueue lock for that
  reason (kernel/sched/core.c).
- `bpf_ktime_get_ns()` and `ktime_get_ns()` read the same monotonic clock,
  and `bpf_printk()` and `trace_printk()` write to the same ftrace buffer.
  Records from BPF and from kernel probes therefore merge into one timeline
  when sorted on their timestamp field. A BPF-side record looks like
  `bpf_printk("EVENT pid=%d val=%d ts=%llu", p->pid, val, bpf_ktime_get_ns())`.
- The default ftrace buffer is 1408 KB per CPU and can truncate a large
  dump. Raise it with the `trace_buf_size=` boot parameter or by writing
  `buffer_size_kb` in tracefs.

### In-kernel ring template
Add this to the target source file in a debug commit. Size the ring to the
memory budget: 32 entries is tight, 64 comfortable and 128 generous.

```c
#define DBG_RING_SIZE   64      /* a power of 2 */
#define DBG_RING_MASK   (DBG_RING_SIZE - 1)

enum dbg_ev {
        EV_INSERT = 1,          /* a1=queue id, a2=queue length */
        EV_REMOVE,              /* a1=queue id, a2=queue length */
        EV_RESCHED,             /* a1=target cpu, a2=reason */
        /* one per observable state transition */
        EV_TRIGGER,             /* a1=observed delta, a2=pid */
};

struct dbg_event {
        u64 ts;
        u32 ev;
        u32 pid;
        u64 a1;
        u64 a2;
};

struct dbg_ring {
        u32 ri;                 /* next slot to write */
        struct dbg_event ev[DBG_RING_SIZE];
};

static DEFINE_PER_CPU(struct dbg_ring, dbg_ring);
static bool dbg_done;           /* set by the first dump */

/* call with preemption disabled, see "Ring cursors" */
static __always_inline void dbg_record(u32 ev, u32 pid, u64 a1, u64 a2)
{
        struct dbg_event *e;
        u32 idx;

        if (dbg_done)
                return;
        idx = this_cpu_inc_return(dbg_ring.ri) - 1;
        e = &this_cpu_ptr(&dbg_ring)->ev[idx & DBG_RING_MASK];
        e->ts = ktime_get_ns();
        e->ev = ev;
        e->pid = pid;
        e->a1 = a1;
        e->a2 = a2;
}

/* one stub per event kind, or call dbg_record() directly if per-site kprobes don't matter */
void foo_dbg_insert(u32 pid, u64 queue_id, u64 queue_len);
__weak noinline void foo_dbg_insert(u32 pid, u64 queue_id, u64 queue_len)
{
        dbg_record(EV_INSERT, pid, queue_id, queue_len);
}

/* dump every online CPU's ring, oldest entry first, see "Pitfalls" for the dbg_done race */
void foo_dbg_trigger_dump(u32 pid, u64 delta);
__weak noinline void foo_dbg_trigger_dump(u32 pid, u64 delta)
{
        int cpu, i;

        if (dbg_done)
                return;
        dbg_done = true;

        trace_printk("TRIGGER pid=%u delta=%llu\n", pid, delta);
        for_each_online_cpu(cpu) {
                struct dbg_ring *r = per_cpu_ptr(&dbg_ring, cpu);
                u32 ri = READ_ONCE(r->ri);

                for (i = 0; i < DBG_RING_SIZE; i++) {
                        struct dbg_event *e = &r->ev[(ri + i) & DBG_RING_MASK];

                        if (!e->ts)
                                continue;
                        trace_printk("cpu=%d ts=%llu ev=%u pid=%u a1=0x%llx a2=0x%llx\n",
                                     cpu, e->ts, e->ev, e->pid, e->a1, e->a2);
                }
        }
}

/* in the hot path, at each transition */
foo_dbg_insert(p->pid, queue_id, queue_len);

/* in the hot path, the trigger evaluated on every relevant event */
delta = ktime_get_ns() - queued_at;
if (unlikely(delta > NSEC_PER_MSEC && !(p->flags & PF_EXITING)))
        foo_dbg_trigger_dump(p->pid, delta);
```

Read the dump back to the agent's machine and merge every CPU's records
into one timeline, then narrow it to the pid that fired the trigger:

```sh
ssh box1 cat /sys/kernel/tracing/trace > "$SCRATCH/<topic>/dump.log"
grep ' ts=' "$SCRATCH/<topic>/dump.log" |
        sed -E 's/.* ts=([0-9]+) .*/\1 &/' | sort -n | cut -d' ' -f2- \
        > "$SCRATCH/<topic>/timeline.log"
pid=$(grep -o 'TRIGGER pid=[0-9]*' "$SCRATCH/<topic>/dump.log" | head -1 | cut -d= -f2)
grep " pid=$pid " "$SCRATCH/<topic>/timeline.log"
```

### bpftrace ring template
A cross-CPU ring with an outlier trigger. It records events from a
hot-path function and, on trigger, prints every CPU's ring and exits. The
ring size is a power of 2, and RMASK is the size minus 1. Add one handler
like the first per probed function, each with its own event code.

```
#!/usr/bin/bpftrace

kprobe:some_hot_function {
        $cpu = (uint64)cpu;
        $idx = (uint64)((++@ri[$cpu] - 1) & 31);        // RMASK = 31
        @r_ts[$cpu, $idx] = nsecs;
        @r_ev[$cpu, $idx] = 1;
        @r_pid[$cpu, $idx] = pid;
        @r_a1[$cpu, $idx] = arg0;
        @r_a2[$cpu, $idx] = arg1;
}

// the trigger: arg0 = start timestamp, arg1 = pid
kprobe:outlier_detector /!@dumped/ {
        $delta = nsecs - arg0;
        if ($delta > 1000000) {
                @dumped = 1;
                printf("OUTLIER pid=%d delta=%ld ns\n", arg1, $delta);
                $cpu = (uint64)0;
                // unroll() needs a constant and silently skips CPUs at or above it
                unroll(8) {
                        $i = (uint64)0;
                        unroll(32) {    // the ring size
                                printf("cpu=%ld i=%ld ts=%ld ev=%ld pid=%ld a1=%lx a2=%lx\n",
                                       $cpu, $i, @r_ts[$cpu, $i], @r_ev[$cpu, $i],
                                       @r_pid[$cpu, $i], @r_a1[$cpu, $i], @r_a2[$cpu, $i]);
                                $i++;
                        }
                        $cpu++;
                }
                exit();
        }
}

// periodic output, since END may not run
interval:s:60 { printf("alive\n"); }
```

Set the outer `unroll()` to the number of CPUs. Run the script and merge
every CPU's records into one timeline:

```sh
ssh box1 bpftrace <path>/ring.bt > "$SCRATCH/<topic>/ring.log" 2>&1
grep '^cpu=' "$SCRATCH/<topic>/ring.log" |
        sed -E 's/.* ts=([0-9]+) .*/\1 &/' | sort -n | cut -d' ' -f2-
```

One-liners for checking probes before building a ring:

```sh
# events per second
bpftrace -e 'kprobe:foo_dbg_my_event { @cnt++; } interval:s:1 { print(@cnt); clear(@cnt); }'
# hits for one command name
bpftrace -e 'kprobe:foo_dbg_my_event { $p = (struct task_struct *)arg0;
        if ($p->comm == "stress-ng") { printf("pid=%d cpu=%d val=%llu\n", $p->pid, arg1, arg2); } }'
```

## Pitfalls
- Instrumentation perturbs what it measures. bpftrace probes on scheduler
  hot paths once made the investigated stall about 25 times more frequent,
  which is why step 7 of [analyze a dump](#analyze-a-dump) reruns without
  them.
- A trigger that fires too often lets the ring overwrite itself before the
  dump runs. Tighten the trigger, enlarge the ring, or stop after the first
  dump to freeze the state of the first hit.
- The trigger fires on one CPU while the bug is on another. Dump every
  CPU's ring, never only the current CPU's.
- Console output stalls the CPU that prints it. A printk, warning or WARN_ON
  to a 115200-baud serial console with IRQs disabled blocks for about 87us
  per byte, so an 85-byte warning takes about 7.4ms. Before chasing
  multi-millisecond stalls on a machine with a serial console, check how
  much the kernel logged during the run, and consider booting with `quiet`
  or with the console on ttynull.
- Rate-limited printks such as `pr_warn_ratelimited()` print a burst and
  then go quiet. A cluster of about ten warnings followed by silence is the
  rate limit, not the condition going away.
- The stop-after-first guard is best effort: a concurrent second trigger
  can pass the plain flag, and the worst case is two interleaved dumps.
  Accept it for a rare trigger with a microsecond window. Use cmpxchg only
  if concurrent triggers turn out to be common.
- Pausing tracing to read a live buffer loses tens of milliseconds of
  events, and a later read shows no overrun for that gap. Read each buffer
  once per run. A second read once cost a full investigation of a test
  failure that wasn't real.

## Customization
- `$SCRATCH` holds dumps and ring logs, one subdirectory per topic.
- An overlay usually adds the subsystem's probe prefix, earlier
  investigations whose debug commits make good starting points, the serial
  console speed of the test machines, and where investigation logs go.
