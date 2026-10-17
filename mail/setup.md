# Mail reading and composing setup

- **Load when:** the user has no setup for reading threads from lore or for
  composing patch mail, asks how to set one up, or a step needs b4, lei or
  git send-email and it is missing.
- **Parameters:** `$IDENTITY`
- **Depends on:** none

A reference setup for the default mail modules: b4 and lei fetch from lore,
and git send-email builds and dry-runs patch mail. Sending from the user's
account and reading the user's own mailbox are in the opt-in mailbox/ area.

## Procedure
1. Install b4 and git send-email from the distribution's packages. Some
   distributions ship git send-email in a separate package. lei comes with
   public-inbox and is optional.
2. Set the identity git send-email puts in From:

   ```
   git config --global sendemail.from "$IDENTITY"
   ```

3. Check that `git send-email --dry-run` on a generated patch prints the
   message and its recipients. A dry run sends nothing.

## Reference
- `b4 am <msgid>` writes an mbox of the series, ready for `git am`, with
  the trailers from the replies added. `b4 mbox <msgid>` writes the whole
  thread. [mail/lore-thread](../mail/lore-thread.md#procedure) shows how
  the two are used and the lore URLs.
- lei keeps searches of lore in local mailboxes. `lei q` runs a query: `-I
  https://lore.kernel.org/all/` searches lore, `-o <dir>` writes the
  results, and `-t` adds every message of each matching thread. `lei up
  <dir>` updates a saved search. lei-q(1) describes the query syntax.

## Customization
- A user who wants the agent to send mail enables
  [mailbox/mail-send](../mailbox/mail-send.md), with
  [mailbox/send-setup](../mailbox/send-setup.md) as the reference setup. A
  user who wants it to read their own mailbox enables
  [mailbox/inbox](../mailbox/inbox.md).
