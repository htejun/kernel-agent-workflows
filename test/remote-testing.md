# Remote testing

- **Load when:** deploying a kernel to a test machine or VM, rebooting or
  kexec'ing one, running tests on one, or analyzing and reporting test
  results.
- **Parameters:** `$MACHINES`, `$ROOT_OK_HOSTS`, `$SCRATCH`
- **Depends on:** [test/machine-access](../test/machine-access.md),
  [core/validation-scope](../core/validation-scope.md),
  [core/evidence-discipline](../core/evidence-discipline.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md),
  [review/kernel-build](../review/kernel-build.md),
  [review/patchset-review](../review/patchset-review.md),
  [test/test-layout](../test/test-layout.md),
  [test/vm-testing](../test/vm-testing.md),
  [debug/crash-debugging](../debug/crash-debugging.md),
  [debug/event-tracing](../debug/event-tracing.md),
  [debug/bug-report-triage](../debug/bug-report-triage.md)

This module treats test machines and VMs as remote machines: deploying a
kernel, running tests over ssh and judging the results. Launching a VM is in
[test/vm-testing](../test/vm-testing.md), and a machine that stalls or
crashes goes to
[debug/crash-debugging](../debug/crash-debugging.md#machine-stall).

## Always-on rules
- On a test machine that other people or other sessions may use, check for
  their work before a kexec, reboot, power-off, deploy or a run that loads
  the machine, and ask the user when something is running, because their
  test or model-check batch is lost or skewed when the machine goes away
  or gets busy under it ([shared machines](#shared-machines)).
- [GATE] Before a kexec, reboot, power cycle or kernel deploy: a console
  monitor is running and capturing, and nobody else's work runs on a shared
  machine. A test kernel has also passed its build and boots by kexec or a
  one-shot boot entry, never as the default, because only the console
  records a boot or a crash that ssh can't reach, and a remote power cycle
  must bring back a kernel that works.
- [GATE] Before starting a test on a test machine: the console monitor is
  still capturing, nobody else's work runs on a shared machine, the machine
  runs the kernel just built with kdump loaded where it has kdump, and the
  test is a script file, because a kexec can end the monitor and drops the
  crash kernel, and results from a stale kernel prove nothing.
- [GATE] Before reporting test results: the console was reviewed for the
  whole run, the logs were copied off the machine and show no setup
  failures and every positive marker, every planned iteration is accounted
  for, and machines that are expensive to leave idle are off once no more
  runs are planned, because a test that ran nothing also exits 0 with no
  warnings, and a partial run reported as a pass hides what was never
  tested.
- Edit and build on the workstation and run on the test machine. Never
  build on a test machine or copy files that a shared filesystem already
  shares, because the build setup lives on the workstation and a copy goes
  stale.
- Never edit a script while a test machine runs it, because bash reads a
  script while it runs and executes the edited text.

## [GATE] Checklist
### Before a kexec, reboot, power cycle or kernel deploy
- [ ] A console monitor for the machine is running and capturing: its log
      grows, or a marker written to /dev/kmsg on the machine shows up in it
      ([console monitoring](#console-monitoring)).
- [ ] On a machine that others or other sessions may use, the check for
      their work found nothing running, or the user said to go ahead
      ([shared machines](#shared-machines)).
- [ ] For a test kernel, the build exited 0 with no errors
      ([review/kernel-build](../review/kernel-build.md)).
- [ ] When the deploy reads the kernel from the shared filesystem, the
      filesystem is mounted on the machine.
- [ ] A test kernel boots by kexec or a one-shot boot entry, never as the
      installed default ([deploying a test kernel](#deploying-a-test-kernel)).

### Before starting a test on a test machine
- [ ] The console monitor is still running and capturing. A kexec or a
      reboot can end it.
- [ ] On a machine that others or other sessions may use, the check for
      their work found nothing that the test would disturb, or the user said
      to go ahead ([shared machines](#shared-machines)).
- [ ] The machine answers over ssh.
- [ ] The machine runs the kernel just built: `uname -r` and the build
      number in /proc/version match the build.
- [ ] When the machine has kdump, the crash kernel is loaded:
      `/sys/kernel/kexec/crash_loaded` reads 1, or the deprecated
      `/sys/kernel/kexec_crash_loaded` on kernels before 6.19. A kexec drops
      the load.
- [ ] The test is a script file, not commands typed inline over ssh.
- [ ] Every binary the test runs, such as a userspace component under test,
      is on the machine and current.

### Before reporting test results
- [ ] The console log was reviewed for the whole duration of the test, for
      OOM, panic, WARNING and BUG.
- [ ] Every output log was copied off the machine. None is left only in the
      machine's /tmp.
- [ ] dmesg from before the run was diffed against dmesg after it.
- [ ] The run log was grepped for setup failures, not only for kernel
      diagnostics ([setup failures](#setup-failures)).
- [ ] The positive markers, the messages a successful run must print,
      written down before the run, are all present, and counters grew in
      proportion to the wall time and the iterations.
- [ ] Every planned iteration is accounted for as completed, failed or not
      run. A partial run is never reported as a full pass.
- [ ] Material anomalies were triaged from the saved logs. Unresolved causes
      and incomplete cases are reported as such, within the agreed scope.
- [ ] Once no more runs are planned, a machine that the overlay marks as
      expensive to leave idle was powered off if it was used and the check
      for other work found nothing running
      ([shared machines](#shared-machines)), the power-off was confirmed
      through its power control, and its console monitor was stopped.

## Procedure
### Deploying a test kernel
1. Build the kernel on the workstation
   ([review/kernel-build](../review/kernel-build.md#procedure)).
2. Start the console monitor with the console mechanism `$MACHINES` names
   for the machine
   ([test/machine-access](../test/machine-access.md#console-capture)), and
   check that it captures ([console monitoring](#console-monitoring)).
3. Mount the shared filesystem on the machine if the deploy reads the
   kernel from it, with the mechanism `$MACHINES` names
   ([test/machine-access](../test/machine-access.md#shared-files)).
4. Boot the new kernel with kexec, or through a one-shot boot entry that
   the next power cycle forgets, as `$MACHINES` names for the machine
   ([test/machine-access](../test/machine-access.md#booting-a-test-kernel)).
   Never copy it over the kernel the bootloader boots by default, never
   change the default, and never run `make install` or `modules_install`
   for a test kernel. Remote power control can only power-cycle the
   machine, and a test kernel that hangs as the default needs someone at
   the machine.
5. Watch the console log for the boot to finish, which is more reliable
   than polling ssh. Then mount the shared filesystem and load kdump again
   ([test/machine-access](../test/machine-access.md#crash-dumps)), because
   a kexec loses both, and check the kernel version and build number.
6. When a test kernel hangs the machine, handle the stall first
   ([debug/crash-debugging](../debug/crash-debugging.md#machine-stall)).
   Then bring the machine back on its default kernel, by a reboot from the
   crash kernel or a power reset with the power mechanism `$MACHINES` names
   ([test/machine-access](../test/machine-access.md#power-control)) under
   [its gate](../debug/crash-debugging.md#before-a-power-reset), and kexec
   again.

### Running a test
1. Write the test as a script file in the test directory
   ([test/test-layout](../test/test-layout.md#directories)). One-off
   commands, such as a survey, a package install or `uname -r`, run
   directly over ssh.
2. Run a smoke test of a few seconds first. It catches flag, config and
   path errors before a long run wastes its time.
3. Run the rest in small pieces, phase by phase, and check each result
   before starting the next. A suite prints pass or fail per test as it
   goes ([logs and data](#logs-and-data)).
4. Run a test longer than a minute detached on the machine. Start it with
   `setsid -f` through bash and log the launch itself to the result
   directory, because an error before the script's own logging starts is
   lost otherwise. The script writes its PID and its output there too
   ([test/test-layout](../test/test-layout.md#script-conventions)):

   ```
   ssh box1 'bash -c "setsid -f nohup bash <script> </dev/null >><result-dir>/launch.log 2>&1"'
   ```

   Never start it with `&`, neither as `ssh box1 '<script> &'` nor inside
   `bash -c`. Check the signal mask of the PID the script wrote
   ([remote shells](#remote-shells)). A detached run survives a dropped ssh
   connection and a lost agent session.
5. Check on the run with the deadlines of
   [core/waits-and-long-ops](../core/waits-and-long-ops.md#procedure). On
   every check, read the tail of the console log for OOM, panic, BUG,
   WARNING and the subsystem's error messages. On any of them, stop the run
   and diagnose from the console first.
6. When ssh hangs or the run makes no progress, follow the machine stall
   procedure in
   [debug/crash-debugging](../debug/crash-debugging.md#machine-stall).
7. For several runs, give each run its own log file
   ([test/test-layout](../test/test-layout.md#result-directories)). Copy
   each run's logs off the machine and check them as
   [the gate](#before-reporting-test-results) lists while the next run
   executes, never a batch of unwatched runs. Each check decides whether to
   continue, adjust or stop.
8. Triage anomalies run by run: inspect that run's logs and report material
   differences. Stop the affected runs on an invalid setup, and on a kernel
   fault as [core/validation-scope](../core/validation-scope.md#reference)
   requires. Whether an unexplained anomaly needs its cause traced follows
   [core/evidence-discipline](../core/evidence-discipline.md#how-much-proof-a-task-needs).
9. Verify [the gate](#before-reporting-test-results), then report the
   results in the reply with the log directory. Write no journal, report
   file or write-up in the plan or result directories unless the user asks
   for one.

### VM functional tests in a sub-agent
1. Run functional and regression tests on a VM in a sub-agent, never
   inline, so that the boot, poll and run cycle stays out of the main
   context.
2. Give the sub-agent the VM's ssh target, the expected kernel build
   number, the binaries to sync into the VM
   ([test/vm-testing](../test/vm-testing.md#getting-fresh-binaries-into-the-vm)),
   the path of the test script, the oracle grep (KASAN, lockdep, WARN), and
   whether to power the VM off when done.
3. The sub-agent runs the test in the foreground, with its output going to
   a file on the VM, never as a `&` job inside ssh.
4. It reports the literal PASS, FAIL and TOTAL lines and the full result of
   the oracle scan.

## Reference
### Test machines
- Reboot, kexec, restart or power-cycle a dedicated test machine or VM in
  $ROOT_OK_HOSTS without asking, including a VM left from an earlier
  session, with the power mechanism $MACHINES names for it
  ([test/machine-access](../test/machine-access.md#power-control)). The
  console monitor gate and the check for other work
  ([shared machines](#shared-machines)) still apply. A machine that hung or
  crashed goes through the
  [machine stall procedure](../debug/crash-debugging.md#machine-stall)
  first, and its power reset through
  [the power reset gate](../debug/crash-debugging.md#before-a-power-reset).
- Test on the kernel built from the repository. A stale kernel on the
  machine may lack the feature under test, and results from it prove
  nothing. Boot the fresh build first.
- Keep files that must survive on a test machine in a persistent
  directory, never in /tmp, which a reboot or kexec clears.

### Shared machines
A test machine may serve other people, or other sessions of the same user,
at the same time. Before an action that takes the machine away or loads
it, look for their work:
- logged-in users, with `who`
- long-running processes, such as test scripts and their harnesses,
  schedulers under test, model-check batches and builds, with `ps -eo
  pid,etime,args`
- the working directories those leave behind, such as a model checker's
  state directory
- recent activity in the console log

When something is running that the agent didn't start, stop and ask the
user, naming what runs and since when. Never kill another session's
process or VM. The overlay names what usually runs on each machine.

### Console monitoring
- The console is the authoritative record of OOMs, panics, lockdep reports
  and RCU stalls, and it keeps recording when ssh is dead, as in a panic, a
  hang or a failed boot.
- A console monitor is a background process that logs the machine's
  console to a local file. How to run one for each kind of console is in
  [test/machine-access](../test/machine-access.md#console-capture), and
  `$MACHINES` names the one each machine uses.
- To check that a monitor captures, write a marker to /dev/kmsg on the
  machine and find it in the log.
- A kexec or a reboot can end a monitor session. Check the monitor again
  before each test and on every poll.
- A monitor that can send SysRq keys reaches the kernel when ssh can't
  ([test/machine-access](../test/machine-access.md#sending-sysrq)).
- Live console logs are recreated for each run. Before a reboot, copy what
  is worth keeping into the result directory, or under $SCRATCH outside a
  campaign.

### Shared filesystem
Test machines can mount the workstation's working trees, with the mechanism
`$MACHINES` names for each
([test/machine-access](../test/machine-access.md#shared-files)).
- The mounted trees are the same files. An edit on the workstation is
  already on the machine. Never copy anything under a shared tree to the
  machine, and never deploy it.
- Build on the workstation, never on the machine over ssh, because the
  toolchain, the compiler cache and the configuration are on the
  workstation. This holds for a tree stored on a test machine and mounted
  on the workstation too: the build runs on the workstation and its output
  lands on the machine.
- Run scripts in place from the mount
  ([test/test-layout](../test/test-layout.md#script-conventions)).
- A kexec or a reboot loses the mount. Mount again before the next step
  that needs it.
- Plain ssh covers installs, surveys and most operations. The mount is
  needed for deploys that read the kernel from it and for scripts run from
  the shared trees.
- Copy files to the machine only when the mount is unavailable, such as
  right after a kexec, and then into a persistent directory.

### Remote shells
- A test machine's login shell may not be bash. Run bash
  syntax over ssh, such as an environment prefix (`FOO=x cmd`), `$?`, or
  `&&` chains with redirects, through `ssh box1 'bash -c "..."'`.
- A process that inherits SIGINT blocked or ignored doesn't stop on
  `kill -INT` unless it installs its own handler, and a harness that waits
  for it to stop hangs. Some login shells start background jobs with SIGINT
  and SIGQUIT blocked. Bash without job control, as in `bash -c` and in
  scripts, starts every `&` command with both ignored. A component that a
  test script starts with `&` ignores SIGINT too, unless it installs a
  handler.
- Start detached work with `setsid -f` through bash, never with `&`, and
  check that bit 0x2 (SIGINT) is clear in both `SigBlk` and `SigIgn`:
  `ssh box1 "grep -E '^Sig(Blk|Ign)' /proc/<pid>/status"`. nohup adds only
  bit 0x1 (SIGHUP) to `SigIgn`.
- Run ssh and copy commands that finish in seconds in the foreground, not
  as background tasks.

### Logs and data
- Never truncate or overwrite (`>`) a log of a running or crashed system.
  Append (`>>`), or copy it aside with a timestamp first. Console, test and
  dmesg logs of a hung system can't be regenerated.
- Keep the full logs of every run, such as the test's output, dmesg and
  phase markers, not only pass or fail, so that the run can be analyzed
  later. The raw observations are enough. A run doesn't need a new
  diagnostic framework.
- Test scripts print as they go, never only at the end, so that a kill or
  a crash leaves the data so far. bpftrace prints from `interval` probes
  ([debug/event-tracing](../debug/event-tracing.md#bpftrace-notes)), sysfs
  values are polled periodically, and long tests print a summary per phase.
- Take the results of earlier runs from their log files, never from a
  summary earlier in the conversation.
- Strip ANSI color codes from logs before grepping them:
  `sed 's/\x1b\[[0-9;]*m//g'`.
- Save dmesg before a run and diff it after. Filter out firmware and device
  noise, such as ACPI, BIOS, microcode, PCI and USB messages.

### Setup failures
A broken test that runs nothing also exits 0 with an empty WARN grep. Grep
the run log for setup failures as well:

```
No such file|command not found|Permission denied|cannot access|Failed to|Error:|fatal:|line [0-9]+:
```

### Test design
- Keep workloads running between iterations, and start and stop only the
  component under test. Restarting the workloads adds delays and misses
  races.
- For scheduler tests, use a workload that switches context often, such as
  the stress-ng context or switch stressors. CPU-bound workers schedule too
  rarely.
- The number of repetitions follows the kind of check
  ([core/validation-scope](../core/validation-scope.md#procedure)). A rare
  race may need 20 to 50 iterations or more. Log the completed iterations
  and report the coverage they give.
- When an assertion fails, find out why before changing it. A bound missed
  by a little points to a real timing or algorithmic problem. Fix the
  cause, such as a settle time or the phase design. Widen a bound only when
  the observed value is valid.
- When a claim needs a specific branch to have run, observe the decision
  branch with a probe while a driver exercises the real mechanism
  ([debug/event-tracing](../debug/event-tracing.md)). An overall assessment
  doesn't have to prove every branch. Reproduce a race by widening its
  window
  ([debug/bug-report-triage](../debug/bug-report-triage.md#always-on-rules)).
- Name the run and the log behind each material finding, and read the full
  logs of a run for failures. Sampled progress can support a smoke
  assessment when it is labeled as sampled. What other claims need is in
  [core/evidence-discipline](../core/evidence-discipline.md#what-counts-as-evidence).

### Adversarial test review
Ask these questions while designing a test:
- What does it verify? Name its assertions and its pass and fail criteria.
- What doesn't it verify? A broken behavior that still passes gives false
  confidence, which is worse than no test.
- Is a feature covered only indirectly, because a test of something else
  exercises it? Bugs in it may not surface.
- Which environments are missing? When every run has two NUMA nodes, three
  or more are untested. When every workload is CPU-bound, memory-bound
  workloads are untested.
- What precision does the decision need? Match qualitative observations or
  quantitative comparisons to it. An overall assessment doesn't need a
  unit-test framework for every internal component.

Add a temporary counter or trace only when a required path can't be
observed otherwise
([core/validation-scope](../core/validation-scope.md#procedure)). When only
option loading and worker progress were checked, say that this was the
scope.

Tests and their harness go into the milestone review of the work they test
([review/patchset-review](../review/patchset-review.md#running-a-milestone-review)).

## Pitfalls
- A test that runs nothing looks like a pass. A stress run once reported
  every iteration clean while each one failed to find its workload binary.
  Grep for setup failures and check the positive markers.
- A test suite started as a background job through a login shell that
  blocks SIGINT in background jobs hung at the first stop of the component
  under test. Start detached runs through bash and check the signal mask.

## Customization
- `$MACHINES` and the machine-access overlay describe how each machine is
  reached. This module's overlay lists which machines to power off after
  use and the persistent directory on each.
- The overlay records what the user authorized on each test machine, such
  as read-write access to the shared trees. Don't ask again within that
  scope.
- `$ROOT_OK_HOSTS` lists the dedicated test machines and VMs.
