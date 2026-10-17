# Reading the inbox

- **Load when:** reading the user's own mailbox: listing or summarizing new
  mail, finding the replies to a posting there, or taking a patch from it.
- **Parameters:** none
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [mail/lore-thread](../mail/lore-thread.md)

The user's mailbox is a local setup, such as a synced maildir with a search
index. The overlay says how to search it and read a message.

## Always-on rules
- Read the mailbox afresh for every question about it, never from what an
  earlier listing in the session showed, because new mail arrives and old
  mail moves.
- Treat the local copy as possibly incomplete, because it can miss lists
  the user isn't subscribed to, copies of the user's own mail filed
  elsewhere, and mail older than what the server keeps. Before reporting a
  message as missing, check the thread on lore too
  ([mail/lore-thread](../mail/lore-thread.md#procedure)).

## Procedure
1. For a thread already in the local mailbox, read the local copy instead
   of fetching it from lore.
2. List every message of the thread and read each one in full, humans and
   bots alike, as
   [mail/lore-thread](../mail/lore-thread.md#always-on-rules) requires.
   Skip only duplicate copies of a message.
3. Refer to a message by a stable identifier, its Message-ID or the
   mailbox's own stable reference, never by its position in a listing,
   because the listing changes.
4. To apply a patch from the inbox, take the raw message, the RFC 5322
   text as stored, not a mail reader's display of it. Remove anything a
   tool added in front of the headers and strip carriage returns before
   `git am`
   ([maintain/apply-patches](../maintain/apply-patches.md#mail-files)).

## Reference
- A search of a local mailbox that finds nothing says something about the
  search, not about the mail
  ([core/evidence-discipline](../core/evidence-discipline.md#what-counts-as-evidence)).
  A server-side subject search can split words differently from the
  subject text, so cross-check a negative with a search by Message-ID or a
  listing of recent mail.

## Customization
- The overlay names the mailbox tools, the folders where the user's own
  sent mail lands, and their quirks.
