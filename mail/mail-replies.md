# Mail replies

- **Load when:** drafting any mail or public comment under the user's name:
  a reply on a mailing list, a review reply to a contributor, an
  acknowledgment of applied patches, a review ack, or a review marked as
  AI-generated.
- **Parameters:** `$IDENTITY`
- **Depends on:** [mail/thread-roots](../mail/thread-roots.md)

## Always-on rules
- A draft under the user's name carries only points the user raised or
  confirmed, and stays neutral about the user's stance, because the mail
  speaks for the user.
- Keep replies short: the point or the ask first, then only what supports
  it, with no recap, offer or pleasantry, because every extra sentence
  stands between the reader and the ask.
- Review replies to contributors ask and suggest instead of ordering, and
  never ask for tests unless the user asks for that, because the
  contributor owns the patch and the user decides what to ask of them.

## Reference
### Replies
- Lead with the technical point or the ask in one sentence. Show a code
  snippet when that is the clearest way to say it. Length follows content.
  When the user asks for a shorter version, cut padding, never the ask.
- One greeting line and one closing line, in the user's form, and nothing
  else ceremonial.
- Leave out:
  - the author's prose quoted back to agree with it;
  - a code comment quoted only to react to it;
  - editorial remarks about smells or design philosophy;
  - a restatement of what a shown snippet does;
  - an explanation of a cause the recipient can diagnose. "Patch 3 doesn't
    apply, please rebase and resend." is the whole message;
  - pros-and-cons lists, a recap at the end, labeled mini-sections such as
    "Background:" or "TL;DR:", and closing offers such as "happy to spell
    that out".
- Don't restate what the recipient knows better: their product, device,
  workload, setup or reproduction steps. An ask to test is "Can you give it
  a test and let me know how it goes?", not a named test suite or
  configuration on their own setup.
- State a clarification directly, instead of opening with an agreement that
  hides the distinction.
- When a reply discusses a setting, such as a sysfs knob or a boot
  parameter, say separately when it can be changed and when a change takes
  effect, because a reader who conflates the two expects the wrong
  behavior.
- Present design sketches as suggestions, leaving room for a better
  solution found while implementing.
- Define every shorthand symbol before using it in a formula.
- Leave out standards or portability objections that have no concrete
  consequence for the implementation and interface under review. For
  example code, documenting its capacity limits can be enough. Keep review
  requests proportional to the purpose of the code.

### Technical replies on mailing lists
- Quote inline with `> ` and answer after each quoted block. Put
  `On <date>, <Name> wrote:` above the first quote only.
- No section headers and no "Sketch:" style introductions.
- Write a scenario as a plain numbered list. When quoting the recipient's
  scenario, follow their format.
- Lead with the cause, not the effect. Cut downstream detail that doesn't
  change the conclusion.

### Review replies to contributors
- Questions and suggestions, never orders: "Maybe add an enum with all the
  valid bits?", "Can you add a blank line between the paragraphs?", "Just
  noting the version it was added in is enough." Never a bare imperative
  tail such as "Drop the rest." Soften the tone, not the technical point.
- When a contributor's description misstates prior behavior but the intent
  is clear, such as "previously silently ignored" meaning that no error was
  raised, don't push back on it.
- Don't raise a kerneldoc-style `@name` on a local variable in a
  contributor's comment as a nit.

### Acknowledgments of applied patches
- The body is one sentence, `Applied to <tree>/<branch>.`, then the user's
  closing and signature. No greeting.
- A single-patch acknowledgment quotes nothing from the patch. A patchset
  acknowledgment quotes the cover letter's shortlog and says
  `Applied 1-N to <tree>/<branch>.`
- Changes made while applying:
  - Tags only: `Applied to <tree>/<branch> with the following tags
    added:`, then the tags, indented.
  - A trivial fix-up: one clause in the applied line, such as "with an
    unnecessary line break removed", and no diff.
  - A dropped tag, such as the Fixes tag stripped from a selftest patch
    ([patches/commit-tags](../patches/commit-tags.md#selftest-patches)): one
    clause in the applied line, such as "with the Fixes tag dropped".
  - Code changes: `Applied to <tree>/<branch> with the following
    changes:`, then the changes as `- ` bullets, with the diff.
  - Changes too big for that: post a new version as a reply instead.
- A follow-up suggestion is one bullet: the gap, the fix and the ask,
  nothing more.
- Write narrative only for a concrete addition, such as a real follow-up
  bug, a tag change or a new version.
- No second-person reference to a reviewer's reply, such as "with your
  Reviewed-by tag". The acknowledgment threads on the original posting,
  where "your" has no antecedent, and the added tags show in the commit.

### Review acks
- Acking a posted series: reply to the cover letter. Quote its opening
  paragraph under `On <date>, <Name> wrote:`. Put any nits as per-patch
  questions (`- 2/5: ...`). Then `For the series,` or `The patchset looks
  good to me.`, a blank line, `Acked-by: $IDENTITY`, the closing and the
  signature.
- Acking each patch, when the user asks for a reply to each: reply to the
  patch itself ([mail/thread-roots](../mail/thread-roots.md#procedure)),
  with References: naming the cover letter and then the patch, and with no
  greeting. Quote the whole description through its tag block under
  `On <date>, <Name> wrote:`, then `Acked-by: $IDENTITY`, the closing and
  the signature.
- A bare "ack" from the user after a review means this plain ack, even when
  the review listed points for a reply. Nits go in only when the user names
  them.

### Drafts under the user's name
- No speculative questions, alternative designs or framings added while
  drafting. If one seems worth raising, ask the user first.
- Neutral about the user's stance. No eagerness ("I definitely want to do
  this") and no decisions ("I'll decide") for steps the user only mentioned
  or defers to an expert on. "I'll proceed as advised" is the shape.
- Keep the user's qualifications about recalled behavior and its precision,
  such as "IIRC" or "roughly".
- When the user's edits narrow a reply, keep that scope in later drafts.
  Don't bring back removed findings about companion patches or nearby API
  policy unless asked.
- Mark a review as AI-generated only when the user asks, which the user
  does when they haven't checked every point themselves. A review the user
  went through goes out as the user's own.
  - On a pull request comment, open with a line such as `AI-generated
    review. Suggestions:`, then a numbered list, one unwrapped paragraph per
    item.
  - In a mail reply, put a line of its own after the greeting, such as
    `The following is an AI-generated review.`, then the usual inline-quoted
    reply.
  - A marked review has no first person, because its findings are not the
    user's: "Two issues found", "neither has been reproduced", not "I
    noticed".

## Customization
- An overlay holds the user's greeting, closing and signature, and the
  exact wording the user wants on an AI-generated review.
