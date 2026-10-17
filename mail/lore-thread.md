# Reading mail threads

- **Load when:** reading the reviews, feedback or replies on a posting,
  fetching a patchset or a thread from lore, or looking for a message on a
  mailing list.
- **Parameters:** `$SENT_LOG`, `$SCRATCH`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md)

## Always-on rules
- Read every message of a thread before reporting on it. Never filter by an
  assumed sender, because a bot's review counts like a person's.
- Report a message or a reply as missing only after reading the relevant
  threads end to end, because a search that finds nothing says something
  about the search, not about the mail
  ([core/evidence-discipline](../core/evidence-discipline.md#what-counts-as-evidence)).

## Procedure
1. Get the Message-ID: from $SENT_LOG for the user's own postings, and from
   the mail or its lore URL otherwise. Copy it, never reconstruct it
   ([core/evidence-discipline](../core/evidence-discipline.md#always-on-rules)).
2. When [mailbox/inbox](../mailbox/inbox.md) is enabled and the thread is
   in the local mailbox, read it there instead. Otherwise fetch the patches
   and the whole thread:

   ```
   b4 am -o $SCRATCH/<topic> <msgid> && b4 mbox -o $SCRATCH/<topic> <msgid>
   ```

   `b4 mbox` saves the thread as an mbox, the same content as
   `https://lore.kernel.org/all/<msgid>/t.mbox.gz`.
3. Don't trust `b4 am` alone. Its output keeps the patches and the trailers
   it recognized, and it can miss reviews, objections and newer versions.
   Read the thread mbox for reviews, objections, NAKs, tags such as
   Reviewed-by, Acked-by and Tested-by, newer versions and maintainer
   feedback.
4. List every message in the thread and read each one in full. Skip only
   duplicate copies of a message. Never cut the list down with `head` or a
   sender pattern.

## Reference
- Message URLs: `https://lore.kernel.org/all/<msgid>/`. Prefer `/all/` over
  a list's own path such as `/lkml/`, because `/all/` finds the message
  whichever lists it went to. The short form
  `https://lore.kernel.org/r/<msgid>` redirects to the message.
- The thread view is `https://lore.kernel.org/all/<msgid>/T/#u`.
- `b4 am -c` checks whether newer revisions of the series exist.
- When a web fetch of a lore page returns a JavaScript challenge page
  instead of the content, fetch it with a headless browser that runs the
  challenge.

## Customization
- Reading the user's own mailbox is the opt-in
  [mailbox/inbox](../mailbox/inbox.md).
