# Test layout

- **Load when:** writing or running a test script, starting a test
  campaign, or deciding where the plans, research notes, test scripts or
  test results of a piece of work go.
- **Parameters:** `$PLAN_DIR`
- **Depends on:** [core/validation-scope](../core/validation-scope.md),
  [test/remote-testing](../test/remote-testing.md),
  [debug/event-tracing](../debug/event-tracing.md)

## Reference
### Directories
Three directories at the root of the working tree hold the work around a
change, each with a subdirectory per feature:

```
$PLAN_DIR/INDEX.md, $PLAN_DIR/<feature>/    plans, research and context
TEST/<feature>/                              test scripts and tools: shell, bpftrace, Python
RESULT/<feature>/{results,campaigns}/        raw output, never committed
```

- `RESULT/.gitignore` holds `*`, so nothing under RESULT/ is committed.
- Helpers shared by several features, such as a flag decoder, sit at the
  top of TEST/.

### Plan directory
- `$PLAN_DIR` holds the plans, research and context of a piece of work.
  `$PLAN_DIR/INDEX.md` links the detail documents.
- INDEX.md gets no progress entries for development rounds.
  Implementation, review rework, renames and retests get none, because the
  commits and their history already say what changed. A chronological
  record belongs only to a debugging investigation, where the order of
  events matters.

### Result directories
- A campaign's result directory
  ([core/validation-scope](../core/validation-scope.md#procedure)) records
  the source revision and local changes, the binary identity, the kernel
  and its configuration, the machine, the commands, the raw logs and the
  outcomes. Common metadata is recorded once, not per run.
- Each run writes its own log file, never overwriting an earlier one.
  Number the files (`sched-0.log`, `sched-1.log`) or name them with a
  timestamp.
- A script that runs several tests writes a separate log file per test.

### Script conventions
- Derive the tree's root from the script's location, and let the
  environment override it. Take the duration as an argument with a default,
  and the CPU count from `nproc`. Never hardcode either, because the same
  script runs from every worktree and on every test machine:

  ```bash
  SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
  TREE_ROOT="${TREE_ROOT:-$(cd "$SCRIPT_DIR/../.." && pwd)}"
  DURATION=${1:-300}
  NPROC=$(nproc)
  ```

- Refer to files through the root, such as `$TREE_ROOT/tools/...`, never
  through the path of one particular tree or machine.
- Run scripts in place on the test machine through the shared filesystem,
  never from a copy, because a copy breaks the relative paths
  ([test/remote-testing](../test/remote-testing.md#shared-filesystem)). A
  run longer than a minute starts detached
  ([test/remote-testing](../test/remote-testing.md#running-a-test)). A smoke
  run can stay in the foreground:
  `ssh box1 bash <tree>/TEST/<feature>/script.sh 10`.
- Write the script's PID to the result directory, so that the signal mask
  of a detached run can be checked
  ([test/remote-testing](../test/remote-testing.md#remote-shells)). Send the
  output through tee to a log in the result directory over the shared
  filesystem. The log is then on the workstation as it is written, and
  survives a dropped ssh connection and a kexec:

  ```bash
  OUTDIR="$TREE_ROOT/RESULT/<feature>"
  OUTLOG="$OUTDIR/run-$(date +%Y%m%d-%H%M%S).log"
  echo $$ > "$OUTDIR/pid"
  exec > >(tee "$OUTLOG") 2>&1
  ```

- Save the PID of every process the script starts, and kill by those PIDs.
  Never kill by a name pattern, such as `pkill -f`, which can match the
  script itself or processes of other runs. Start a workload whose workers
  aren't its direct children, such as stress-ng, in its own process group
  with `setsid`, and kill the group.
- Clean up in a trap on EXIT that kills the saved PIDs with SIGKILL and
  stops bpftrace as
  [debug/event-tracing](../debug/event-tracing.md#bpftrace-notes)
  describes. The script sets `BPFTRACE_STOPPED=1` once it has stopped
  bpftrace itself. The trap never calls a bare `wait`, which also waits for
  the tee process and hangs:

  ```bash
  setsid stress-ng --switch "$NPROC" &
  LOAD_PID=$!
  <component-under-test> &
  TEST_PID=$!

  cleanup() {
      if [ -n "${BPFTRACE_PID:-}" ] && [ "${BPFTRACE_STOPPED:-0}" = "0" ]; then
          kill -INT "$BPFTRACE_PID" 2>/dev/null || true
          sleep 3
          kill -9 "$BPFTRACE_PID" 2>/dev/null || true
      fi
      kill -9 -- "-$LOAD_PID" 2>/dev/null || true
      kill -9 "$TEST_PID" 2>/dev/null || true
  }
  trap cleanup EXIT
  ```

- Under `set -e`, end a kill that may find the process gone with
  `|| true`: `kill $pid 2>/dev/null || true`. The redirect hides only the
  message, and the exit status is still 1.
- Wait for a process to exit with a polling loop that has a timeout, never
  a bare `wait`, which can hang.

## Customization
- `$PLAN_DIR` moves the plan directory.
- An overlay can name other places for test scripts and results, such as a
  scratch directory in a repository that keeps no TEST/ directory.
