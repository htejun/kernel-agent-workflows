# VM testing

- **Load when:** launching, restarting or stopping a QEMU or virtme-ng (vng)
  VM to test a kernel, building a kernel for vng, getting fresh binaries
  into a VM, capturing a large console dump from a VM, or testing suspend
  and resume in a VM.
- **Parameters:** `$BUILD`
- **Depends on:** [core/waits-and-long-ops](../core/waits-and-long-ops.md),
  [review/kernel-build](../review/kernel-build.md),
  [test/remote-testing](../test/remote-testing.md),
  [test/machine-access](../test/machine-access.md),
  [debug/crash-debugging](../debug/crash-debugging.md)

This module covers what is specific to VMs: starting QEMU and vng so that
they survive the agent harness, and getting a new kernel and fresh binaries
into them. Running tests on a VM as on any test machine is in
[test/remote-testing](../test/remote-testing.md).

## Procedure
### Launching a QEMU VM
1. Copy anything worth keeping from the previous run's console log
   ([test/remote-testing](../test/remote-testing.md#console-monitoring)).
   Then remove the console log and the monitor socket, so that nothing from
   the previous run is mistaken for the new one.
2. Launch QEMU in a shell call of its own and print its PID. The kernel
   comes from the build tree with `-kernel`, the console goes to ttyS0, and
   the monitor listens on a unix socket:

   ```
   nohup qemu-system-x86_64 -nographic -kernel <tree>/arch/x86/boot/bzImage \
           -append "console=ttyS0 <boot options>" \
           -monitor unix:<socket>,server=on,wait=off <machine options> \
           </dev/null > <console log> 2>&1 &
   echo $!
   ```

   Launch it detached like this, with nohup, stdin from /dev/null and the
   output in a file, because the harness can kill a shell's background
   children when the call ends, and the session manager can kill a plain
   background job minutes later. Never pipe QEMU through tee and never run
   it as the harness's own background task. A private tmux server also
   outlives the shell call:

   ```
   tmux -L <name> new-session -d \
           "bash -c 'exec qemu-system-x86_64 ... </dev/null > <console log> 2>&1'"
   ```

3. In a separate shell call, about five seconds later, check that the exact
   PID printed at launch is alive (`ps -p <pid>`). Never look it up with
   pgrep, which can match unrelated processes. If the process is gone, read
   the console log for the error before launching again.
4. Poll ssh to the VM every three seconds until it answers, within the
   deadline of [core/waits-and-long-ops](../core/waits-and-long-ops.md).
5. Check that the VM runs the kernel just built, as the test gate of
   [test/remote-testing](../test/remote-testing.md#before-starting-a-test-on-a-test-machine)
   describes.

### Booting a new kernel
1. QEMU reads `-kernel` only when it starts, and a reboot inside the guest
   boots the old kernel again. Stop QEMU and launch it again for every new
   build. The test kernel is never installed into the VM's disk image
   ([test/remote-testing](../test/remote-testing.md#deploying-a-test-kernel)).
2. If the VM hung or crashed, dump its memory first
   ([debug/crash-debugging](../debug/crash-debugging.md#qemu-guest-memory-dumps)),
   because stopping it drops its state.
3. Stop it with `quit` on the monitor socket, or with `kill -9 <pid>` in a
   shell call of its own. When other QEMU or vng instances may be running,
   stop it by its exact PID or through its monitor, never with a pattern
   kill such as `pkill -f qemu-system`, which also matches the VMs of other
   sessions ([one command per shell call](#one-command-per-shell-call)).
   Check with `ps -p <pid>` that it is gone, and wait two seconds.
4. Launch it as above.

A VM that is already running may belong to a concurrent session, which the
agent can't tell from one left by an earlier session. Restart without
asking only a VM the overlay names as the user's alone
([test/remote-testing](../test/remote-testing.md#test-machines)). For any
other VM, check with the user before stopping it.

### Getting fresh binaries into the VM
A VM whose disk image holds a copy of the trees, with no share of the
workstation's trees, runs the fresh kernel from `-kernel` but stale tools
and test scripts from its disk. Before every run, sync the fresh tools,
binaries and test scripts into the VM with rsync, in the foreground
([test/remote-testing](../test/remote-testing.md#remote-shells)). This is
mechanics: do it without asking.

### Testing suspend and resume
1. Start QEMU with the monitor socket.
2. Suspend from inside the guest by writing `mem` to /sys/power/state.
   QEMU has no monitor command that suspends a guest.
3. Wake the guest from the host with `system_wakeup` on the monitor socket,
   sent every five seconds while the test cycles through suspends:

   ```
   echo system_wakeup | socat - UNIX-CONNECT:<socket>
   ```

### Running vng
vng boots a kernel on a copy-on-write view of the host's root filesystem.
When the overlay names a QEMU VM as the default for kernel test runs, use
vng only when the task calls for it, such as a BPF verifier run, a quick
boot probe or a reproducer for
[debug/bug-report-triage](../debug/bug-report-triage.md).

1. Generate the configuration in the kernel tree with
   `vng --build --config <fragment>`. Then force the virtualization options
   that vng's fragment handling leaves as modules or off
   ([vng kernel configuration](#vng-kernel-configuration)) and rebuild
   ([review/kernel-build](../review/kernel-build.md#procedure)).
2. Run the built kernel with
   `vng --run <tree>/arch/x86/boot/bzImage -- <command>`. Always name the
   kernel: `vng --run` without one boots the host's kernel.
3. Give vng a terminal for its output and send that to a file
   ([vng output](#vng-output)).
4. Run one vng instance at a time. Run a long one in a foreground shell
   call with a timeout that covers it, or check that a background one
   survives its first minute.

## Reference
### One command per shell call
- Kill a VM in one shell call and launch it in another. A `pkill` that
  matches nothing exits nonzero and aborts the rest of an `&&` chain, and
  `pkill -f qemu-system-x86_64` matches the shell's own command line and
  kills that shell before the launch runs.
- `pkill -x qemu-system-x86_64` matches nothing, because the kernel cuts a
  process name to 15 characters.
- Never chain QEMU steps with `; sleep N;`. Each step gets its own shell
  call.

### Monitor socket
- Start QEMU with `-monitor unix:<socket>,server=on,wait=off` and send a
  command with `echo '<command>' | socat - UNIX-CONNECT:<socket>`.
- Useful commands: `info status`, `system_wakeup` to resume a suspended
  guest, `quit`, and `dump-guest-memory` for a hung guest, which on a
  KASAN kernel needs `-device vmcoreinfo` at launch
  ([debug/crash-debugging](../debug/crash-debugging.md#qemu-guest-memory-dumps)).
  SysRq keys and resets through the monitor are in
  [test/machine-access](../test/machine-access.md#sending-sysrq).

### Large console dumps
QEMU's emulated ttyS0 is orders of magnitude slower than a virtio console.
A run that can print a large dump, such as
`ftrace_dump_on_oops` with a large `trace_buf_size` or SysRq-t on many
CPUs, needs a virtio console next to ttyS0. Otherwise the dump takes tens of
minutes and a competing watchdog panic cuts it short.

- Add a virtio console that writes to a file:

  ```
  -device virtio-serial-pci,id=vser0 -chardev file,id=hvclog,path=<hvc log> \
  -device virtconsole,chardev=hvclog,bus=vser0.0
  ```

- Add `console=hvc0` to the boot options and keep `console=ttyS0`. Both
  receive the full log.
- Raise `watchdog_thresh` for the dump window, so that a soft-lockup panic
  can't cut the dump short.

### vng output
- Run from a shell without a terminal, as an agent's background call is,
  vng prints `WARNING: stdin/stdout/stderr not accessible via
  /proc/self/fd` and the guest command's output never appears. Give it a
  terminal with script and strip the `\r` from the log when reading it:

  ```
  script -qec 'vng --run <tree>/arch/x86/boot/bzImage -- bash /abs/repro.sh' /abs/out.log
  ```

- Never pipe vng through tail, which buffers until vng exits and hides
  whether it hangs or progresses. Redirect to a file and poll the file.
- An exit with status 255 after about two seconds with an empty log is a
  transient failure to start the VM. Run it again.

### vng boot hangs
When vng prints nothing, its log stays empty and QEMU sleeps after burning
some CPU, the microvm machine type is usually stuck calibrating the TSC
(`tsc: Unable to calibrate against PIT`, `tsc: No reference (HPET/PMTIMER)
available`). Run again with `--disable-microvm`, which uses the q35 machine
type. `--verbose` shows the boot, and "Marking TSC unstable" followed by
silence is the fingerprint. When q35 stalls too, check the kernel
configuration ([vng kernel configuration](#vng-kernel-configuration)).

### vng kernel configuration
`vng --build --config <fragment>` doesn't honor `=y` for the virtio options
in the fragment. It leaves `VIRTIO_*` as modules and HYPERVISOR_GUEST off.
vng's virtio runs over MMIO, and stale modules in `.virtme_mods` fail BTF
validation against a rebuilt vmlinux. After every configuration
generation, force the options, run `$BUILD olddefconfig` and rebuild:

```
scripts/config -e HYPERVISOR_GUEST -e PARAVIRT -e KVM_GUEST -e VIRTIO -e VIRTIO_PCI \
        -e VIRTIO_MMIO -e VIRTIO_CONSOLE -e VIRTIO_BLK -e VIRTIO_NET -e VIRTIO_FS -e FUSE_FS \
        -e DAX -e FUSE_DAX -e OVERLAY_FS -e NET_9P -e NET_9P_VIRTIO -e 9P_FS -e DEVTMPFS_MOUNT
```

A stall at "Marking TSC unstable" means KVM_GUEST is missing, which leaves
the kernel without kvm-clock. An init panic after "mounting hostfs", such as
virtiofs reporting `tag <ROOTFS> not found`, means VIRTIO_MMIO or OVERLAY_FS
is not built in.

### vng filesystem
- `~` inside the guest is not the host home: root's home is remapped under
  /run/tmp. Use absolute paths.
- The shared host home is effectively read-only in the guest. `--rwdir`
  gives a writable view, but its writes were seen gone after vng exited. A
  harness that writes results into the host tree runs with `--rw`.
- The guest command starts in the host's current directory. Invoke scripts
  by absolute path, with the executable bit set.
- The guest's /tmp may be read-only. Write to a tmpfs on /run, or log
  through /dev/kmsg.

### vng VMs that die
When a vng VM dies with vhost noise, such as `vhost_set_vring_call failed
22`, `Failed to set msg fds` or `set virtio-user-fs status to 7 failed`,
read the top of the log first. `terminating on signal 15 from pid ...`
naming the agent harness means the harness reaped a background task. A
second vng instance running at the same time is the other suspect: one
coincided with the first instance's virtiofs backend dying.

## Pitfalls
- A QEMU started as a plain background job without nohup was killed by the
  user's session manager minutes later. Use the launch pattern in step 2.

## Customization
- The overlay holds the launch command: the wrapper script if there is one,
  the disk image, the CPU topology, the boot options, the ssh target and
  port, and the paths of the console log and the monitor socket.
- The overlay records the sync commands for each tree's tools and test
  scripts, and which VM is the default for kernel test runs.
- `$BUILD` rebuilds a kernel after its configuration changes.
