# Publishing

- **Load when:** about to publish anything others can see: send mail, push
  to a remote others can read, post on GitHub, create a release or upload
  a package.
- **Parameters:** `$SEND_TOKEN`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [mail/mail-compose](../mail/mail-compose.md)

## Always-on rules
- [GATE] Before publishing: everything is prepared, the readiness summary
  showed the exact command, the user's most recent message contains
  $SEND_TOKEN as a standalone word, and the command runs exactly as
  presented, because what is published can't be recalled.
- A push to any remote others can read or that backs a pull request, the
  user's fork included, is publishing. The push of an instructed apply and
  the signed tag before a pull request don't wait for $SEND_TOKEN, because
  the instruction for those flows covers them.
- "Post", "send", "ship", "go ahead", "do it", "yes" and "ok" mean prepare
  and wait, because only $SEND_TOKEN approves publishing.
- Never run a publishing command expecting a permission prompt to stop it,
  because running the command is the publishing and the harness may allow
  the command without asking.
- Finish every step but the publishing command before asking for
  $SEND_TOKEN, because anything composed, pushed or assembled after the
  approval was never approved.
- The agent sends mail itself only when
  [mailbox/mail-send](../mailbox/mail-send.md) is enabled, and otherwise
  hands the user the files and the command
  ([who sends mail](#who-sends-mail)), because sending from the user's
  account is opt-in.

## [GATE] Checklist
### Before publishing
- [ ] Every step except the publishing command is done: files complete,
      tags signed and pushed, request-pull output in the mail.
- [ ] The readiness summary was presented with the complete files
      ([The readiness summary](#the-readiness-summary)).
- [ ] The user's most recent message, re-read just now, contains
      $SEND_TOKEN as a standalone word, or the
      [second-factor retry](#second-factor-tokens) applies.
- [ ] The same thing was not published before. Check its record, such as
      the sendmail log, `git ls-remote` or the pull request page.
- [ ] The command is the presented string, character for character. Any
      difference means stop, present the actual command, and wait for a
      fresh $SEND_TOKEN.

## Procedure
1. Prepare everything the publishing needs, including the checks of the
   module that owns it, such as [mail/mail-compose](../mail/mail-compose.md)
   for mail, and [mailbox/mail-send](../mailbox/mail-send.md) when it is
   enabled.
2. Present the readiness summary alongside the complete files.
3. Wait. Before running the command, re-read the user's most recent
   message for $SEND_TOKEN.
4. Run the presented command once, character for character.
5. Check the result the way the owning module says, and inspect the
   resulting state instead of running the command again
   ([core/safe-operations](../core/safe-operations.md#always-on-rules)).

## Reference
### What counts as publishing
- Mail, from git send-email or a sendmail tool.
- GitHub: creating a pull request or an issue, commenting on either,
  reviewing or approving a pull request, a public gist, a release, an
  attachment upload, a POST to a public API endpoint.
- Publishing a package, such as `cargo publish`, and web hooks.
- A push to any remote others can read or that backs a pull request, the
  user's fork included: a release tag, a new branch, an update or a
  force-push. The pushes that don't wait belong to
  [maintain/apply-patches](../maintain/apply-patches.md#applying) and
  [maintain/pull-request](../maintain/pull-request.md#creating-and-sending).
- Anything else publicly visible or indexable. A draft shown to the user
  publishes nothing.

### Who sends mail
- The agent sends mail itself only when
  [mailbox/mail-send](../mailbox/mail-send.md) is enabled. It then follows
  that module and the gate here.
- Otherwise the agent gives the user the mail files and the exact command,
  built as [mail/mail-compose](../mail/mail-compose.md) describes, and the
  user sends them. The gate here doesn't apply to a send the user makes.
- After the user sends, take the Message-ID of the posting from the user or
  from lore, never from memory.

### The readiness summary
Present it alongside the complete files. It states what goes where, the
details the owning module lists, and the exact shell command: absolute
paths, every option, every file named, no `...` and no globs. For a
force-push, it states the old and the new tip of the branch. $SEND_TOKEN
approves that command string, not the prose around it.

### Second-factor tokens
- Some services, such as a git server that wants a second factor for
  pushes, reject a command until the user supplies a token. Try the
  command first: a token given earlier often stays valid for a while and
  covers several pushes.
- On an authentication error, ask the user for the token as plain text.
  Validate it in a separate, non-interactive command. Never wait at an
  interactive prompt.
- Then retry the unchanged command. When the command was approved with
  $SEND_TOKEN, the token keeps that approval, so don't ask for $SEND_TOKEN
  again. The owning module's checks still apply, and anything already
  published is never published again.

## Pitfalls
- An acknowledgment went out with no approval: the agent ran the send
  expecting a permission prompt to stop it, and the prompt allowed it.
- Pull request branches were force-pushed after a CI failure report,
  without asking.
- A push was reported as the agent's own success when a hook had refused
  both of its attempts and the user had pushed by hand. The remote-tracking
  reflog's "update by push" entry doesn't say whose push it was. Read the
  remote's tip with `git ls-remote` in the same shell call as the push and
  compare it with the commit pushed. When the outcome is still unclear,
  ask the user before reporting it.

## Customization
- `$SEND_TOKEN` sets the approval word.
- An overlay records the user's remotes, the user's own trees whose pushes
  belong to other flows, and how the user's services validate a
  second-factor token.
