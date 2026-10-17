# Codex

- **Load when:** working inside the Codex agent harness.
- **Parameters:** none
- **Depends on:** [core/publishing](../core/publishing.md),
  [mailbox/mail-send](../mailbox/mail-send.md)

Quirks of the Codex harness that the other modules' rules depend on. Codex
reads its instructions from AGENTS.md files.

## Reference
### Mail sends
- Run the approved audit and send command of
  [mailbox/mail-send](../mailbox/mail-send.md#the-send-flow) with
  `sandbox_permissions="require_escalated"` from the first attempt. The
  restricted sandbox blocks the network and the keyring lookup a sendmail
  tool's password command makes, so the send fails there. Don't try the
  send inside the sandbox first.
- The escalation setting leaves the approved command unchanged, so the
  send gate's "runs exactly as presented" still holds
  ([core/publishing](../core/publishing.md#before-publishing)).
- Preparing the mail files and running the mail audit alone stay inside the
  sandbox.
