# Machine access

- **Load when:** setting up a test machine or VM for the agent, adding one
  to `$MACHINES`, or reaching one for a shell, console capture, power
  control, SysRq, crash dumps, shared files or booting a test kernel.
- **Parameters:** `$MACHINES`, `$ROOT_OK_HOSTS`, `$SCRATCH`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [test/remote-testing](../test/remote-testing.md),
  [test/vm-testing](../test/vm-testing.md),
  [debug/crash-debugging](../debug/crash-debugging.md)

A test needs several capabilities from a machine, and each can be provided
in more than one way. This module describes each mechanism: what it needs on
the machine and the workstation, the commands, how to check it and what goes
wrong.

## Always-on rules
- Reach a test machine through the mechanism `$MACHINES` names for it and
  the capability, and ask the user when it names none, because an
  improvised path can act on the wrong machine or lose the state the named
  one preserves.
- Read access credentials from a secret store or a file at the moment of
  use, and never write them into a command, a script or an overlay,
  because transcripts and files get copied. Prefer a tool's file or
  environment option to a password argument, because command lines show up
  in process lists ([pitfalls](#pitfalls)).

## Procedure
### Adding a machine
1. Ask the user for the machine's name, whether it is a VM or bare metal,
   and which mechanism provides each capability: shell, console capture,
   power control, SysRq, crash dumps, shared files and test-kernel boot
   ([Reference](#reference)).
2. Set up the shell first, because most other checks run through it. Set up
   the console capture second, because the SysRq and boot checks are read
   from it.
3. Set up and check each other mechanism as its section describes. Check
   power control by reading the power state, never by changing it. A
   capability whose check fails is recorded as none, and the user is told.
4. Set kernel.sysrq on the machine
   ([debug/crash-debugging](../debug/crash-debugging.md#sysrq-through-the-console)).
   On a machine with kdump, check that a crash kernel loads
   ([crash dumps](#crash-dumps)).
5. Record the machine and its mechanisms in `$MACHINES`. Record the details
   in the overlay: hosts and ports, device paths, BMC addresses, where each
   credential is kept, and the user's wrapper scripts.

### Using a machine
1. Find the machine in `$MACHINES` and use the mechanism it names for the
   capability, with the details from the overlay. A machine that isn't
   listed, or a capability listed as none, means asking the user.
2. A power reset and a crash trigger pass their gates in
   [debug/crash-debugging](../debug/crash-debugging.md#gate-checklist) first.
3. On a machine that others or other sessions may use, check for their
   work before taking the machine away or loading it
   ([test/remote-testing](../test/remote-testing.md#shared-machines)).

## Reference
### Shell
ssh as root to a dedicated machine in `$ROOT_OK_HOSTS`.
- Needs: sshd on the machine, the workstation's public key in root's
  authorized_keys there, and a Host entry on the workstation so that
  commands use a short name:

  ```
  ssh-copy-id root@box1.example.org

  # ~/.ssh/config
  Host box1
      HostName box1.example.org
      User root
  ```

- Check: `ssh -o BatchMode=yes box1 true` exits 0. BatchMode turns password
  and host key prompts into failures, so a check that needs a person fails
  instead of hanging.
- A VM reached through a forwarded port takes `Port` in its Host entry.
- After a machine is reinstalled, its host key changes and ssh refuses it.
  Remove the old key with `ssh-keygen -R <host>` once the user confirms the
  reinstall.
- The login shell may not be bash
  ([test/remote-testing](../test/remote-testing.md#remote-shells)).

### Console capture
A capture runs detached on the workstation, appends the machine's console
to a log under `$SCRATCH`, and is checked on every poll
([test/remote-testing](../test/remote-testing.md#console-monitoring)).
Start it with nohup, stdin from /dev/null and its output in a file, and
note its PID, as for a VM
([test/vm-testing](../test/vm-testing.md#launching-a-qemu-vm)). Every
mechanism is checked the same way: write a marker to /dev/kmsg on the
machine and find it in the log.

#### IPMI serial-over-LAN
- Needs: a BMC with serial-over-LAN enabled and reachable from the
  workstation, a BMC user allowed to use it, the kernel's `console=` on the
  serial port the BMC redirects at the BMC's bit rate, and ipmitool and
  expect on the workstation. ipmitool takes the password from a file with
  `-f` or from `IPMI_PASSWORD` with `-E`.
- ipmitool needs a terminal and reads escape sequences from it, so run it
  under expect, which logs the session and can send keys
  ([sending keys into a capture](#sending-keys-into-a-capture)):

  ```
  #!/usr/bin/expect -f
  # usage: sol-capture.exp <log> <keyfile> ipmitool -I lanplus -H <bmc> -U <user> -f <passfile>
  set log [lindex $argv 0]
  set keyfile [lindex $argv 1]
  set timeout 2
  log_file -a $log
  log_user 0
  file delete -force $keyfile
  spawn {*}[lrange $argv 2 end] sol activate
  expect {
      "SOL Session operational" {}
      "SOL payload already active" { exit 1 }
      timeout { exit 1 }
  }
  while {1} {
      expect -re ".+" {} timeout {} eof { exit 1 }
      if {![file exists $keyfile]} { continue }
      set f [open $keyfile r]; set key [string trim [gets $f]]; close $f
      file delete -force $keyfile
      if {$key eq "quit"} { send "\r"; after 300; send "~."; expect eof; exit 0 }
      send "\r"; after 300; send "~B"; after 500; send $key
  }
  ```

- Start it with `nohup expect sol-capture.exp <log> <keyfile> ipmitool ...`
  as above, and stop it by writing `quit` to the key file.
- "SOL payload already active" means an earlier session still holds the
  BMC's console. Stop the local capture that owns it, run
  `ipmitool ... sol deactivate`, and start again.
- ipmitool recognizes `~.` and `~B` only right after a newline, which is
  why the script sends a carriage return first. Under an ssh session, `~`
  is also ssh's escape character: pick another SOL escape with `-e`.

#### Serial adapter on a console host
- Needs: the machine's serial port wired to a USB serial adapter on a
  console host that the workstation reaches over ssh, such as a
  KVM-over-IP device, and the kernel's `console=` on that port.
- Name the adapter by its stable path under /dev/serial/by-id/. The ttyUSB
  number can change after a replug, and a console host may have several
  adapters.
- Capture, as above:

  ```
  ssh <console host> 'stty -F <device> 115200 raw -echo; exec cat <device>' >> <log> 2>&1
  ```

- The capture only reads. Keys go through a separate write
  ([serial break](#serial-break)).
- Run one reader per device. Two readers split the incoming bytes between
  them, and each log misses part of the console.
- A dropped ssh connection ends the capture. The check on every poll finds
  it, and the capture starts again.

#### netconsole
The kernel sends its messages over UDP to a listener on the workstation
(Documentation/networking/netconsole.rst). It only sends, so it can't carry
SysRq, and it starts only once the network driver is up, so it misses
early boot failures.
- Needs: netconsole built in or as a module on the machine, and an open
  UDP port on the workstation.
- On the machine, in the kernel command line or at run time:

  ```
  netconsole=6665@<machine ip>/<dev>,6666@<workstation ip>/<workstation mac>
  modprobe netconsole netconsole=@/<dev>,6666@<workstation ip>/<workstation mac>
  ```

- On the workstation, as above. `-u` keeps socat listening after its own
  stdin, /dev/null, ends:

  ```
  socat -u udp-recv:6666 - >> <log>
  ```

- Without a target MAC address, netconsole broadcasts, and some switches
  drop broadcasts. Across subnets, give the MAC address of the gateway.
- The console loglevel on the machine limits what is sent. A firewall on
  the workstation drops the packets without an error.

#### QEMU serial console
The console goes to a file as part of the launch, with `-serial file:<log>`
or with `-nographic` and QEMU's output redirected
([test/vm-testing](../test/vm-testing.md#launching-a-qemu-vm)). A run that
can print a large dump needs a virtio console as well
([test/vm-testing](../test/vm-testing.md#large-console-dumps)).

### Power control
Read the power state after every action. A reset or power-off of a machine
that hung or crashed passes
[the power reset gate](../debug/crash-debugging.md#before-a-power-reset)
first.

#### IPMI chassis power
```
ipmitool -I lanplus -H <bmc> -U <user> -f <passfile> chassis power status
ipmitool -I lanplus -H <bmc> -U <user> -f <passfile> chassis power on
```

- `on` powers up. `off` cuts power without shutting the system down. `cycle`
  keeps the power off for at least a second and does nothing to a machine
  that is already off, so read the state first. `reset` is a hard reset.

#### KVM-over-IP ATX control
A KVM-over-IP device wired to the machine's power and reset headers presses
the buttons for it. With a PiKVM, through its HTTPS API:

```
printf 'user = "%s:%s"\n' <user> "$(<secret lookup>)" | curl -sk -K - https://<kvm>/api/atx
printf 'user = "%s:%s"\n' <user> "$(<secret lookup>)" |
        curl -sk -K - -X POST 'https://<kvm>/api/atx/power?action=on'
```

- curl reads the credentials from its standard input (`-K -`). printf is a
  shell builtin, so the password never appears on a command line.
- `GET /api/atx` returns `busy`, `enabled` and the power LED state in
  `leds.power`.
- `POST /api/atx/power?action=` takes `on`, which does nothing when the
  machine is on, `off`, a short press that the operating system handles,
  `off_hard`, a long press, and `reset_hard`, the reset button.
- `POST /api/atx/click?button=` takes `power`, `power_long` and `reset`.
- The device may be reachable only from another host. Then the commands run
  there over ssh, and the overlay says so.
- Other KVM-over-IP devices have their own APIs. The overlay holds their
  commands.

#### Redfish
A BMC with Redfish takes power actions over HTTPS. Find the system, read its
`PowerState` and the reset action, then post the reset:

```
curl -sk -K <auth file> https://<bmc>/redfish/v1/Systems
curl -sk -K <auth file> https://<bmc>/redfish/v1/Systems/<id>
curl -sk -K <auth file> -X POST -H 'Content-Type: application/json' \
        -d '{"ResetType": "ForceRestart"}' \
        https://<bmc>/redfish/v1/Systems/<id>/Actions/ComputerSystem.Reset
```

- Take the action's URI from the system's
  `Actions["#ComputerSystem.Reset"].target`, and the values the BMC
  supports from the `@Redfish.AllowableValues` annotation on `ResetType`.
- The values include `On`, `ForceOff`, `GracefulShutdown`,
  `GracefulRestart`, `ForceRestart`, `PowerCycle` and `Nmi`.

#### Switched PDU
A switched power distribution unit turns an outlet off and on through its
own web, SNMP or ssh interface, which differs by vendor. The overlay holds
the commands and the outlet number. Switching an outlet always cuts power
without a shutdown.

#### QEMU monitor
`system_reset` resets the guest, `system_powerdown` sends it an ACPI power
button event, and `quit` stops QEMU, after which it is launched again
([test/vm-testing](../test/vm-testing.md#monitor-socket)). Dump a hung
guest's memory before a reset or a quit
([debug/crash-debugging](../debug/crash-debugging.md#qemu-guest-memory-dumps)).

### Sending SysRq
A key sent through a console is checked against kernel.sysrq, and a write
to /proc/sysrq-trigger is not
([debug/crash-debugging](../debug/crash-debugging.md#sysrq-through-the-console)).
Check a mechanism with SysRq-h, which only prints the help line
(`sysrq: HELP :`) to the console.

#### /proc/sysrq-trigger
`ssh box1 'echo h > /proc/sysrq-trigger'`. It needs a working shell as root,
so it fails on the machines that need SysRq most.

#### IPMI serial-over-LAN break
ipmitool's `~B` sends a serial BREAK, and the key follows within five
seconds (Documentation/admin-guide/sysrq.rst). With the capture above,
write the key to its key file: `echo h > <keyfile>`.

#### Serial break
A break through a serial adapter is a separate write on the console host,
next to the capture. Python's `termios.tcsendbreak()` sends it:

```
ssh <console host> python3 - <device> h <<'EOF'
import os, sys, termios
fd = os.open(sys.argv[1], os.O_RDWR | os.O_NOCTTY)
termios.tcsendbreak(fd, 0)
os.write(fd, sys.argv[2].encode())
EOF
```

#### QEMU sendkey
`sendkey alt-sysrq-h` on the monitor socket
([test/vm-testing](../test/vm-testing.md#monitor-socket)).

#### Sending keys into a capture
A console connection usually has one owner: a BMC serves one SOL session,
and the capture holds it. Let the capture send keys for the agent: it
watches a key file, sends what the agent writes there and deletes the file,
as the expect script above does. A serial adapter is the exception, since a
write can go through a second open of the device.

### Crash dumps
kdump boots a capture kernel when the running kernel panics, and the
capture kernel exposes the old memory as /proc/vmcore
(Documentation/admin-guide/kdump/kdump.rst).
- Needs: the running kernel built with `CONFIG_KEXEC` or `CONFIG_KEXEC_FILE`
  and `CONFIG_CRASH_DUMP`, booted with a `crashkernel=` reservation such as
  `crashkernel=512M`; a capture kernel image; kexec-tools and makedumpfile
  on the machine.
- Load the capture kernel, either through the distribution's kdump service
  or directly. kdump.rst lists the options for each architecture, such as
  `irqpoll nr_cpus=1 reset_devices` on x86_64. `--reuse-cmdline` copies the
  running command line and drops its `crashkernel=`:

  ```
  kexec -p <capture kernel> --reuse-cmdline --append="irqpoll nr_cpus=1 reset_devices"
  ```

- Check: `/sys/kernel/kexec/crash_loaded` reads 1, or
  `/sys/kernel/kexec_crash_loaded` on kernels before 6.19. A reservation
  that is missing shows as 0 in `crash_size` next to it.
- A kexec drops the load, so load again after every kexec
  ([test/remote-testing](../test/remote-testing.md#deploying-a-test-kernel)).
  A kexec'd test kernel needs `crashkernel=` on its own command line.
- The capture kernel's `console=` should name the captured console, so that
  its boot shows in the log.
- A distribution's kdump service saves the dump to its configured directory
  and reboots. With a direct load, the capture kernel stays up and the dump
  is saved by hand
  ([debug/crash-debugging](../debug/crash-debugging.md#machine-stall)).
- A QEMU guest needs no kdump: its memory is dumped from the host through
  the monitor
  ([debug/crash-debugging](../debug/crash-debugging.md#qemu-guest-memory-dumps)).

### Shared files
The workstation builds and the machine runs, on the same files
([test/remote-testing](../test/remote-testing.md#shared-filesystem)).
Mount a tree at the same absolute path on both sides, so that paths in
scripts and logs mean the same thing everywhere. A kexec or a reboot loses
every mount.

#### Reverse sshfs
The machine mounts the workstation's tree over an ssh connection that the
workstation opens, so the machine needs no route or credentials to reach the
workstation. dpipe connects the workstation's sftp-server to sshfs running
on the machine in passive mode:

```
nohup dpipe /usr/lib/ssh/sftp-server = ssh box1 sshfs :<tree> <tree> -o passive \
        </dev/null >/dev/null 2>&1 &
```

- Needs: sshfs and FUSE on the machine, dpipe (from vde2) and OpenSSH's
  sftp-server on the workstation. The sftp-server path differs between
  distributions. Older sshfs releases name the option `-o slave`, which
  current ones still accept.
- Check within a deadline: `ssh box1 test -f <tree>/<a known file>`.
- dpipe keeps the standard output and error of the shell that started it.
  Without the redirects, the shell call never returns.
- The mount hangs when the machine hangs. Give every mount and check a
  timeout.

#### NFS
The workstation exports the tree, and the machine mounts it with
`mount -t nfs <workstation>:<tree> <tree>`. The machine needs a route to the
workstation. An export squashes root by default (`root_squash`), so root on
the machine writes as the anonymous user.

#### 9p and virtiofs
A VM can mount a host directory through QEMU:
`-virtfs local,path=<tree>,mount_tag=<tag>,security_model=none` at launch,
and `mount -t 9p -o trans=virtio <tag> <tree>` in the guest. virtiofs needs
virtiofsd on the host; vng sets it up itself
([test/vm-testing](../test/vm-testing.md#vng-filesystem)).

#### rsync
Without a share, push the fresh binaries and scripts before every run:
`rsync -a <tree>/<dir>/ box1:<tree>/<dir>/`
([test/vm-testing](../test/vm-testing.md#getting-fresh-binaries-into-the-vm)).

### Booting a test kernel
A test kernel boots once and is never the default
([test/remote-testing](../test/remote-testing.md#before-a-kexec-reboot-power-cycle-or-kernel-deploy)).

#### kexec
- Needs: a running kernel with kexec support, kexec-tools on the machine,
  and the test kernel's image readable there.
- Load, then reboot into it. `systemctl kexec` shuts the system down
  cleanly first, and `kexec -e` jumps at once without a shutdown.
  `kexec <image> --append=...` without `-l` loads and then runs shutdown,
  which ends in the same kexec:

  ```
  kexec -l <tree>/arch/x86/boot/bzImage --reuse-cmdline --append="crashkernel=512M"
  systemctl kexec
  ```

- Start from the running command line, as `--reuse-cmdline` does, and add
  what the test needs, such as `crashkernel=` for kdump and `console=` for
  the capture.
- Check the version and build number once it is up
  ([test/remote-testing](../test/remote-testing.md#before-starting-a-test-on-a-test-machine)).
- On a machine with Secure Boot lockdown, only the file-based load
  (`kexec -s`) works, and it needs a signed kernel.

#### One-shot boot entry
Add a boot loader entry for the test kernel next to the default one, then
select it for the next boot only:
- systemd-boot: `bootctl list` shows the entry IDs, and
  `bootctl set-oneshot <id>` selects one for the next boot.
- GRUB: `grub-reboot <entry>` selects one for the next boot. When GRUB can't
  write its environment block, such as on MDRAID or LVM, the entry stays
  the default, so don't use it there.
- Never install the test kernel over the default entry's files.

#### QEMU -kernel
QEMU boots the kernel named with `-kernel` at launch, so a new kernel means a
new launch
([test/vm-testing](../test/vm-testing.md#booting-a-new-kernel)).

## Pitfalls
- ipmitool overwrites a `-P` password in its own command line once it
  parses its options, so the password shows in the process list only
  between the start and the parse. `-f` or `-E` keeps it off the command
  line entirely.

## Customization
- `$MACHINES` names each machine's mechanisms. The overlay holds the hosts,
  ports, adapter device paths, BMC addresses, where each credential is kept
  and the user's wrapper scripts, such as a capture script or a mount
  helper.
- Machines with an unusual mechanism, such as a vendor PDU or a
  KVM-over-IP device without an API, get their commands in the overlay.
