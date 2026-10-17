# Performance comparison

- **Load when:** comparing the performance of two kernels, schedulers,
  builds or settings, measuring latency for a claim, running an A/B or a
  before/after comparison, or generating and modulating load with stress
  or stress-ng.
- **Parameters:** `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [core/validation-scope](../core/validation-scope.md),
  [review/kernel-build](../review/kernel-build.md),
  [test/test-layout](../test/test-layout.md),
  [debug/event-tracing](../debug/event-tracing.md)

Design choices in performance work are settled by measurement
([core/evidence-discipline](../core/evidence-discipline.md#performance-work)).
This module covers how to measure so that a comparison holds.

## Always-on rules
- [GATE] Before comparing performance: both variants ran on bare metal, on
  performance kernels without debug instrumentation built from the same
  configuration (one kernel for userspace variants), with the effective
  configuration, the identity and the boot options verified and saved,
  because a VM or a debug kernel measures the host and the instrumentation,
  not the change.
- Measure the metric that the mechanism changes, never a system-wide metric
  that unrelated factors dominate, because a tangential metric hides the
  effect in noise.

## [GATE] Checklist
### Before comparing performance
- [ ] Every measurement and pilot ran on bare metal. VMs are for
      functional, verifier and stress validation, never for performance.
- [ ] Both variants ran on separately built performance kernels, with
      lockdep, KASAN, KCSAN and expensive memory and debug instrumentation
      off. Kernel variants were built from the same configuration.
      Userspace variants ran on the same kernel.
- [ ] The effective .config was audited after configuration resolution.
      The settings a fragment requested are no evidence by themselves.
- [ ] The running kernel's identity and boot options were verified against
      that build, and the configuration is saved with the results.
- [ ] Functional and stress validation ran on a separate debug kernel, and
      no result from it is presented as a performance comparison.

## Procedure
### A/B latency comparison
This procedure is for a targeted latency investigation. For an overall
performance assessment, use the benchmark's own measurements and report
their limits ([core/validation-scope](../core/validation-scope.md#procedure)).

1. Name the exact metric the mechanism changes, and measure that. A
   mechanism that keeps tasks from being stranded on a per-CPU queue is
   measured by the time tasks spend on that queue, not by general runqueue
   latency, which unrelated factors dominate.
2. Check the kernel command line in /proc/cmdline: the preemption mode
   (`preempt=`), the root device and the console settings.
3. Keep the normal C-state and frequency settings. Change idle settings
   only when the scoped question needs a controlled experiment, and save
   the previous values and restore them afterwards
   ([C-states](#c-states)).
4. Look into firmware sources of stalls, such as SMIs, only when observed
   stalls justify it, and record the existing settings. BIOS and TPM
   changes are outside a routine benchmark and need their own scope and
   authorization. An unexplained gap alone doesn't establish its cause.
5. Match the workload's granularity to the slice of the scheduler under
   test. A workload sized for very short slices is far too heavy for long
   ones. Use about twice as many workers as CPUs, which gives realistic
   contention and still leaves ssh usable.
6. Check that every probe target exists before the run
   ([debug/event-tracing](../debug/event-tracing.md#before-trusting-probe-data)).
7. Check that the mechanism is engaged: read `/proc/<pid>/cmdline` of the
   process under test and the mechanism's counters in sysfs, because ssh
   quoting can drop a flag without an error.
8. Collect distributions for tail claims. Keep the benchmark's samples or
   histograms from the first run, and report only the percentiles its
   output supports. If tracing is needed, print periodically, not only at
   exit. An overall assessment doesn't need a new tracing system to get a
   missing percentile.
9. Present the results as described in [latency results](#latency-results).

### Before/after comparison
1. Build the baseline from the base commit without the change
   ([review/kernel-build](../review/kernel-build.md#procedure)), copy the
   binary aside under `$SCRATCH`, then build the changed version.
2. Run an identical sequence, such as the same staircase of load steps,
   with each binary. Write each run's results to its own file and compare
   them side by side.
3. Set the noise tolerance in the measured unit before comparing.
   Differences within it are noise.
4. Keep two kinds of cases apart: cases the change must not affect, which
   must match the baseline exactly, and cases where the change alters
   behavior on purpose, which may differ only in the intended way.

### Modulating load with SIGSTOP and SIGCONT
Modulate load by stopping and continuing workers, not by killing and
restarting them. Killing the workers drops their utilization to zero, and a
scheduler that sizes allocations from utilization shrinks them and then
recovers slowly.

1. Start the maximum number of workers up front and SIGSTOP all of them.
2. Save the worker PIDs to a file. Poll until every worker has spawned
   before going on. stress-ng takes two to three seconds to fork its
   workers.
3. In each phase, SIGCONT and SIGSTOP subsets of the workers.
4. Find workers by the tool's process layout
   ([stress and stress-ng](#stress-and-stress-ng)).
5. Clean up with SIGKILL to the workers first, then to the master, and
   guard each kill under `set -e` as
   [test/test-layout](../test/test-layout.md#script-conventions) describes.
   A SIGTERM to the master hangs while its children are stopped.

## Reference
### C-states
- Disabling every idle state: write 1 to each
  `/sys/devices/system/cpu/cpu*/cpuidle/state*/disable`, then check that
  they all read 1. Deep idle states add tens to hundreds of microseconds of
  wakeup latency.
- Save the previous values before writing and restore them after the
  experiment.

### Latency results
- Use buckets at several resolutions: 10us steps up to 4ms, 1ms steps up
  to 400ms and 100ms steps up to 40s. In bpftrace, keep threshold counters
  at 1, 5, 10, 50, 100 and 500us, at 1, 5, 10, 50, 100 and 500ms, and at
  1s.
- Present the distribution as a CDF, the share of samples at or below each
  threshold. Summarize it with percentiles, the latency at which the CDF
  reaches each share, as `P50=Xus P90=Yms P99=Zms P99.9=Wms`, so that an
  A/B comparison reads at once.
- Print per-interval and cumulative figures in every print. Per-interval
  figures show the steady state, and cumulative ones include the ramp-up.
  Reset the per-interval counters after each print.

### stress and stress-ng
- `stress --cpu N` is purely CPU-bound with no variants. Its workers are
  direct children of the master, all named `stress`, so `pgrep -P <master>`
  finds them.
- stress-ng names each worker `stress-ng-<stressor>`, cut to the kernel's
  15-character process name limit, such as `stress-ng-matri`. Some
  stressors fork further children, so find workers by name
  (`pgrep stress-ng-matri`), never by parent. A pgrep pattern longer than
  15 characters matches nothing.
- Give each load group its own stressor, or stress for one and stress-ng
  for another, so that each group has its own process name to match.
- A prefix match on `stress` also matches every stress-ng worker. Check the
  stress-ng prefixes first.
- Memory-bound stressors such as `--cache` and `--ptr-chase` spend most of
  their time waiting for memory. Dozens of cache workers may produce only a
  few CPUs of utilization, so never assert high CPU counts from them.

| Stressor | Load |
|---|---|
| `--cpu N` | general CPU; `--cpu-method matrixprod` is cache-heavy, `int32` integer-only |
| `--matrix N` | 2D matrix, floating point and cache; `--matrix-size 256` for a larger working set |
| `--intmath N` | pure integer math |
| `--stream N` | STREAM memory bandwidth, NUMA-sensitive |
| `--cache N` | cache thrashing; `--cache-level 3` targets the L3 |
| `--ptr-chase N` | pointer chasing, latency-sensitive |
| `--syncload N` | bursty; `--syncload-msbusy M --syncload-mssleep M` |
| `--yield N` | rapid `sched_yield()` |
| `--switch N` | context switch pressure |
| `--varyload N` | self-modulating, through its own SIGSTOP and SIGCONT |

## Pitfalls
- `pgrep -c` prints 0 and exits 1 when nothing matches, so
  `$(pgrep -c ... || echo 0)` yields two lines. Write
  `count=$(pgrep -cP "$master" 2>/dev/null) || count=0`.

## Customization
- The overlay names the bare-metal machines, the performance kernel's
  configuration and where it is built, the noise tolerance of each
  benchmark on each machine, and where baseline binaries go.
- `$SCRATCH` holds baseline binaries and comparison results outside a test
  campaign's result directory.
