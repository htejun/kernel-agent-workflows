# Waits and long-running operations

- **Load when:** starting or waiting on builds, test runs, remote jobs, model
  checks or sub-agents that take more than a few seconds.
- **Parameters:** none
- **Depends on:** none

## Always-on rules
- Run long operations one at a time and verify each before starting the
  next, because a second operation started on a failed one wastes its run
  and hides the failure.
- Every wait has a deadline and checks progress against ground truth, never
  against what the job or sub-agent reports about itself, because a stuck
  job and a slow one look the same from outside.
- Read the output of every background command and confirm that it succeeded
  before going on, because a silent failure looks like silent success.

## Procedure
1. Before starting, estimate the duration and set a deadline from it. An
   incremental kernel build takes tens of seconds, a full one a few minutes
   and longer with sanitizers. A test scenario takes a minute or two. A soak
   takes its length plus slack.
2. The agent harness notifies when work it tracks completes, so don't poll
   that work. For work the harness can't see, such as a detached job on a
   remote machine, check at intervals matched to the expected duration,
   each check bounded by a deadline. Never block in one long foreground
   wait.
3. On each check, verify progress against ground truth: modification times
   of build artifacts, the process tree under the job, and log tails.
4. Two checks without progress means the job is stuck. Take over, kill the
   idle process, and investigate.
5. When a long operation fails, read the error. Never retry blindly or work
   around the symptom.

## Reference
- Suspend and pause mean SIGSTOP to the runner and its workers, which keeps
  their state, and resume means SIGCONT. SIGSTOP any wrapper too, so that
  it doesn't start the next job. Only a request to stop, kill or abort ends
  a run, and an earlier request to end the same job doesn't turn a later
  suspend into a kill.
- A sub-agent saying it is waiting is not progress. Check what it waits on.
- `make` waiting on stdin at a new config prompt looks like a hang. Run
  `make olddefconfig </dev/null` before building with a changed config.

## Pitfalls
- A failed pull in a background command went unnoticed and produced wrong
  commit IDs further on. Read every result before using it.
