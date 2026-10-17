# Claude Code

- **Load when:** working inside the Claude Code agent harness.
- **Parameters:** `$SCRATCH`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md),
  [core/publishing](../core/publishing.md),
  [mail/mail-compose](../mail/mail-compose.md),
  [review/patchset-review](../review/patchset-review.md),
  [test/vm-testing](../test/vm-testing.md)

Quirks of the Claude Code harness that the other modules' rules depend on.

## Always-on rules
- Read files with the Read tool whenever possible. Write content the agent
  authors with the Edit and Write tools, never through a heredoc or shell
  redirection, because the tools show the exact change and track the
  file's state. Redirecting a command's own output to a file is fine.
  `replace_all` only for a pattern verified unambiguous in that one file.

## Reference
### Shell calls
- A process started with `&` inside a Bash call has been seen to die when
  that call's shell exits. A process that must outlive the call, such as a
  VM, starts detached as
  [test/vm-testing](../test/vm-testing.md#launching-a-qemu-vm) describes.
- `run_in_background` runs a command detached across turns and notifies
  when it exits, which makes it work the harness tracks
  ([core/waits-and-long-ops](../core/waits-and-long-ops.md#procedure)).
  Short commands such as ssh and scp run in the foreground. For work the
  harness can't see, the Monitor tool runs the bounded checks.
- Bash calls sent in one response are the parallel calls of
  [core/safe-operations](../core/safe-operations.md#before-a-shell-command-that-targets-a-repository).
- The Edit and Write tools strip trailing whitespace from the lines they
  write. A mail signature's `-- ` becomes `--`
  ([mail/mail-compose](../mail/mail-compose.md#building-a-mail-file)), and
  a diffstat line in a patch header loses the trailing space its format
  has, so a header edit stops short of the diffstat.

### Sub-agents
- A sub-agent runs on the parent's model
  ([review/patchset-review](../review/patchset-review.md#always-on-rules)).
  An unset model inherits the parent's unless the agent type's definition
  or a configured default sub-agent model names another; then pass the
  parent's model explicitly. A fork always runs on the parent's model.
- A sub-agent doesn't see the conversation. Give it the full context its
  task needs: paths, the rules that apply and what to report.
- A sub-agent's report is its own account. Check the claims a decision
  rests on before acting on them.

### Files and permissions
- The harness scratchpad is a per-session directory, which
  [core/safe-operations](../core/safe-operations.md#always-on-rules) keeps
  working files out of. They go under `$SCRATCH`.
- A permission prompt can allow a command without asking the user, which
  is why [core/publishing](../core/publishing.md#always-on-rules) never
  relies on one to stop a send.

### Recovering lost work
Session transcripts are JSONL files under `~/.claude/projects/`. Read the
session's transcript in order, trace each Edit to the final state of the
location it changed, and check that against the file before applying
anything.
