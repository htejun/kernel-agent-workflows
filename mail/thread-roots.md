# Thread roots

- **Load when:** choosing what an outgoing mail threads under: a new version
  of a patchset, a patch re-posted from a series, a single patch, an
  acknowledgment, a review reply, or a reply to an automated report.
- **Parameters:** `$SEND_TOKEN`
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [mail/mail-send](../mail/mail-send.md),
  [patches/cover-letters](../patches/cover-letters.md)

## Always-on rules
- [GATE] Before setting a mail's thread root: take the first matching case
  of the precedence. A full patchset version starts a fresh thread, a patch
  re-posted from a series replies to its own original, an acknowledgment
  replies to the original posting, and a standalone patch that could go
  under an existing discussion is the user's call, because a mail under the
  wrong root is lost to the readers of its thread.

## [GATE] Checklist
### Before setting a mail's thread root
- [ ] The case is the first one in [the precedence](#procedure) that fits.
- [ ] A case the user decides was asked in the readiness summary and
      answered.
- [ ] Every Message-ID in In-Reply-To and References was copied from the
      thread mbox or the sent copy, never written from memory
      ([core/evidence-discipline](../core/evidence-discipline.md#always-on-rules)).

## Procedure
Take the first case that fits.

1. A full patchset version, cover letter and all patches: a fresh top-level
   thread. No In-Reply-To or References to the previous version, which is
   linked only by the `vN:` lines of the cover letter
   ([patches/cover-letters](../patches/cover-letters.md#procedure)). This is
   not a question for the user.
2. One patch re-posted from a series, such as `[PATCH v2 3/5]` replacing
   patch 3: In-Reply-To and References point at that patch's original
   Message-ID, never the cover letter's, because each patch is the root of
   its own follow-ups. This is not a question either.
3. A standalone single-patch version, or any patch that could plausibly go
   under an existing discussion, such as a reviewer's reply or a bug report
   thread: the user decides. Ask in the readiness summary of
   [mail/mail-send](../mail/mail-send.md#the-readiness-summary), "Thread as
   (a) a fresh top-level thread [default] or (b) In-Reply-To <msgid>?", and
   wait for the answer. $SEND_TOKEN approves the send, not the threading
   choice.
4. An acknowledgment of applied patches: thread on the original posting,
   the cover letter for a patchset and the patch for a single patch. Never
   thread it on a reviewer's reply, which buries it one level down.
5. A per-patch review reply: thread on the patch itself, with References
   listing the cover letter, then the patch.
6. A reply to an automated report, such as a build robot or a CI bot:
   reply in the report's thread and keep its recipients, adding a Cc for
   the developer who should look. Never forward it privately. When the bot
   sets Reply-To, the reply goes to that address, with the bot's sender and
   the lists it copied in Cc.

## Reference
- In-Reply-To names the parent message. References lists the chain of
  Message-IDs from the thread root down to the parent.

## Customization
- An overlay records the addresses of the review bots on the user's lists.
