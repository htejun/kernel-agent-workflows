# Prose voice

- **Load when:** writing or revising any prose for the user or under the
  user's name: a patch description, a code comment or kerneldoc, a cover
  letter, a mail, a pull request body or comment, a plan or a doc, or a
  reply in chat.
- **Parameters:** none
- **Depends on:** [core/overlays](../core/overlays.md),
  [patches/patch-descriptions](../patches/patch-descriptions.md),
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md),
  [patches/cover-letters](../patches/cover-letters.md),
  [mail/mail-replies](../mail/mail-replies.md)

An opinionated voice for every piece of prose, chat replies included. It
applies on top of the patches/ and mail/ modules and replaces their
conventions where the two differ. Each place that differs names the section
it replaces.

## Always-on rules
- Say the concrete thing once, in plain subject-verb-object sentences with
  the real actor: the conclusion, a one-line reason, stop, because a point
  the reader has to derive or reread gets missed ([details](#sentences)).
- Use the words a kernel comment on the subject would use, never coined
  terms, creative verbs, vapid wrappers or nicknames for code paths,
  because the reader has to guess what an unusual word means
  ([details](#words)).
- Leave out every word that doesn't change what the reader understands,
  show-off tone included, because each one stands between the reader and
  the point ([details](#what-to-leave-out)).
- An edit the user makes to the agent's prose counts as a correction, said
  or not, because otherwise the user makes the same edit again
  ([details](#after-an-edit-or-a-critique)).
- [GATE] Before showing prose: no paragraph runs past 8 lines, and the text
  has the length its kind calls for, because a longer paragraph can't be
  taken in at a glance and the reader skims past the point.
- [GATE] Before committing a patch: the prose it adds has no semicolon
  chains or dash joins, one space after each period, no coined terms, and
  "block" or "wait" only for paths that sleep, because these slip into
  drafts unnoticed and each one a reviewer finds costs another round.

## [GATE] Checklist
### Before showing prose
- [ ] Every paragraph is 8 lines or fewer, commit messages and chat replies
      included. A longer one is split at its point boundaries.
- [ ] The length fits the kind of text ([Length](#length)).
- [ ] A first-time reader gets through it in one pass.

### Before committing a patch
- [ ] The description and the comments the patch adds have no semicolon
      chains and no `--` or em-dash joins ([Mechanics](#mechanics)).
- [ ] One space follows every period.
- [ ] No coined term or creative word ([Words](#words)).
- [ ] "Block" and "wait" describe only paths that sleep.

## Procedure
### Writing a draft
1. Write it as you would say it aloud to a colleague: the conclusion first,
   then the one-line reason.
2. Shorten only by dropping words, never by fusing two facts into one
   clause.
3. Reread it as a first-time reader and restructure anything that needs a
   second pass. Then run the [checklist](#before-showing-prose).

### After an edit or a critique
1. When the user edits a draft, diff the edit against the draft and name the
   pattern behind it. Record it at once, as
   [core/overlays](../core/overlays.md#recording) describes, in the overlay
   of the module that owns the pattern: this module's for word choice and
   sentence form.
2. Record a convention seen in the user's posted mail or edits the moment it
   is noticed. Never search the user's sent mail or past threads for a
   convention already recorded, and end any search that was needed with
   the pattern written down.
3. When the user says a draft is bad, take the new constraint and write a
   new version. Don't defend the old one.

## Reference
### Sentences
- State the essence once: the conclusion, the one-line reason, stop. No
  mechanism dumps, no restating a point from another angle, and no hedges
  such as "in practice this can't affect" or "strictly speaking". The
  shortest true version wins.
- Say the concrete thing with its real actor and name: the function, the
  kernel version, the event, and what follows from it. Never make the
  reader derive the point through "implies", a hidden premise, an
  appositive stack or a pile of relative clauses. Rewrite anything that
  needs a reread before showing it.

  ```
  Before: A configured limit implies a kernel with the lookup fix: per-node pools, the only
          users of the limit, came after it.
  After:  foo_lookup() stopped checking the caller's namespace in v6.12. Only per-node pools
          set the limit, and they were added in v6.14, so every kernel that can set the limit
          has the fix.
  ```

- Subject, verb, object, one claim per sentence. No garden-path relative
  clauses and no stacked trailing modifiers. An event is a clause, not a
  noun stack: "Return NULL if the parent group has no owner. That happens
  when the parent's init callback fails while its owner is being removed",
  not "NULL if the parent group lost its owner to a failed ownership
  return".

  ```
  Before: Record the driver whose init the foo got in foo->owner, updated wherever FOO_READY
          changes so that READY is equivalent to a non-NULL owner.
  After:  Add foo->owner to track which driver initialized the foo. It is set and cleared
          together with FOO_READY.
  ```

- Every clause names its actor and its action. Two facts are two sentences
  at the two places they belong, never one compressed clause chain. A
  relative clause with no referent, such as "which the kernel grants us on
  every CPU", says nothing. Barrier and race comments follow
  [patches/comments-kerneldoc](../patches/comments-kerneldoc.md#where-to-say-it).
- Pronouns and established nouns carry the reference: "It works, but is
  clumsy." Once "the block" is established, refer back to it. More words
  must buy more clarity: "is in the CONFIG_FOO_GROUP block", not "is
  maintained inside the CONFIG_FOO_GROUP-only bulk group init and exit
  functions".
- Finish one thought before starting the next. "X goes false before the RCU
  wait, which can depend on those tasks running, and only then does Y move
  the tasks" wedges a second explanation into a sequence. Related facts may
  still share a sentence when it reads naturally. Don't overcorrect into
  one choppy fact per sentence.

### Fact, rule, instruction
These replace the shape of a comment at a check in
[patches/comments-kerneldoc](../patches/comments-kerneldoc.md#where-to-say-it).
- Where a check or a write needs a comment, say what is true, what must
  stay true, and what to do, in that order: "When a CPU goes offline, its
  quota is cleared and must stay zero until the CPU comes back online.
  Don't write a non-zero quota for an offline CPU."
- Not the derivation through the mechanism: "the admission checks treat an
  offline CPU as quota-less through its zeroed quota, and the refill on its
  return reports the grant as the delta it finds then". Not the chain of
  consequences of the wrong choice: "writing it here would admit work onto
  a CPU that is down". Not an abstract subject.
- The derivation behind the reason, such as how a deadlock comes about,
  belongs in the patch description. The comment states the constraint and
  the action: "A pinned task must stay on this CPU. Let it run here and
  count the violation."
- State a constraint directly, never the adversarial scenario it prevents:
  "The scratch lives in BPF-writable arena memory and its header can't be
  trusted, so it is rewritten from kernel geometry rather than read"
  (kernel/sched/ext/ext.c), not "so a scheduler scribbling on it can't
  steer the fill". The scenario personifies the code and makes the reader
  work back to the invariant.
- "So" joins a fact to the action it forces, never two facts: "The
  interrupted call needs its saved state back when it resumes, so save and
  restore it."
- A skip or an early return that prevents a failure is the exception to
  leaving out the consequences. Its comment says why the skip is needed and
  what goes wrong without it, concretely: the dropped callback, the node
  left linked, the memory freed while still on a list. The reader must see
  the hazard the test guards against, not only that the work is pointless.

  ```
  Not enough: A CPU going down moves all of its tasks off. Ignore requeues to its local queue.
  Enough:     A CPU going down moves all of its tasks off in one final pass. A requeue to its
              local queue after that pass would leave the task where nothing runs it until the
              CPU comes back. Ignore it.
  ```

### Length
- A paragraph tracks a topic, not a sentence: three sentences making one
  point are one paragraph. A paragraph past 8 lines is a wall the reader
  can't parse at a glance. Split it at every point boundary before showing
  the draft, commit messages and chat replies included.
- A trivial fix, such as a typo, a wrong size or a single flag, gets two or
  three sentences. This adds a count to
  [patches/patch-descriptions](../patches/patch-descriptions.md#by-kind-of-patch).
- A cover letter's testing summary is three to five sentences at the
  conclusion level. This replaces "a few sentences" in
  [patches/cover-letters](../patches/cover-letters.md#testing-summary).

### Patch descriptions
These add to
[patches/patch-descriptions](../patches/patch-descriptions.md#what-a-description-says).
- When a shorthand label misleads, such as "Admission is cooperative",
  unpack it into a plain sentence instead of hunting for a better label.
- Compact from the first draft. Drop restated mechanism and qualifiers
  such as "with LSE", "for them" or "of the helpers". Merge short goal and
  obstacle sentences into one setup paragraph. Drop a transition word that
  opens a paragraph, such as "Instead,", when the paragraph break carries
  the contrast. When the goal says X must not happen, the obstacle only
  needs "X can happen". Drop an abstract topic sentence that restates the
  concrete sentence after it.
- Keep the subtle points to two or three, and announce them with a count:
  "Two things need care."
- Don't restate the symptom when the cause implies it. "Re-acquires an
  already-held rq lock" is the AA deadlock, so no "lockdep reports possible
  recursive locking" after it.
- The paragraph of incidental extras opens with "While at it,".
- A note to the maintainer below the `---` line is friendly, neither
  defensive nor pleading: "This looks minimal to me, but happy to take
  suggestions."

### Mailing-list replies
These add to
[mail/mail-replies](../mail/mail-replies.md#technical-replies-on-mailing-lists).
- Default to the tightest version, above all to mm, BPF and scheduler
  maintainers. Terse technical shorthand is fine: "trylock + retry at 5
  would A-A deadlock".
- A contract statement gets its own paragraph, phrased directly. Where the
  contract is an expectation, say "should".
- A caveat opener is one sentence at most. No apologies for a scenario,
  such as "stretched as it may be".
- No "(Cc'd)" flags in the body.
- Drop domain qualifiers the recipient already assumes, such as
  "kernel-side" to an mm reviewer.

### Words
- Plain words over formal register: "dead code" instead of "has no
  observable effect", "same" instead of "verdict-equivalent", "nonsense"
  instead of "semantically vacuous". Plain verbs over a jargon paraphrase:
  "freed via", not "returned via", when the operation is a free.
- Conventional nouns and verbs, never creative ones. A bypass doesn't
  "swallow" a report, a parent doesn't "survive", a report isn't "moot",
  and a disable doesn't "keep clear of" a power transition. The report is
  suppressed, ignored or lost, the parent stays enabled, and the disable
  doesn't run during a power transition. A creative word adds nothing, and
  the reader has to guess which of its meanings is meant. Before showing
  prose, replace every verb or noun that wouldn't appear in an existing
  kernel comment on the same subject.
- No colloquial idioms such as "shook out" or "fallout". Direct phrasing is
  clearer.
- Name the mechanism. No vapid wrappers: "reason about", "handle", "deal
  with", "work with", "think about", "steer", "drive", "provide X
  semantics". "Have the scheduler reason about idle CPUs itself" says
  nothing. "Switch the scheduler's idle picker from the kernel helper to a
  bitmap scan in BPF" says what changes.
- Choose words by context, not from a ban list: ask whether the word claims
  something the situation has. "Stage" implies a later commit and "stamp" a
  fixed-form identifier. "Hoist" is colloquial for "move". "Writer" and
  "reader" as nouns hide the call site. "Typically", "essentially" and
  "basically" hedge. "Synchronously", "explicitly", "directly", "actively"
  and "immediately" belong only where the contrast is real.
- Never coin terminology, such as "v-style helper", "state-2 task" or
  "populated gate". No abstract role nouns for variables, such as "the
  discriminator", "the sentinel" or "the handle". Say what the value means
  at run time: "non-NULL only when walking cgroup->self". "Doubles as" and
  "serves as" are fine when a plain description follows. If the standard
  term is unknown, describe the thing plainly.
- Call a code path by the code's own name, in discussion, comments,
  descriptions and models alike: its function name, or the noun the code
  uses for the operation. Never a nickname, and never a word the code
  already uses for something else.
- No jargon abbreviations: TU, CU, AST, IR, RVO, SFINAE, ODR, fail-closed,
  fail-open. Common kernel ones are fine: LLC, SMT, NUMA, RCU, IRQ, BPF.
  When a plain sentence next to the jargon already says the same, delete
  the jargon sentence.
- "NFC" and kerneldoc `%` markup such as `%NULL` stay out of every
  artifact, not only the descriptions
  [patches/patch-descriptions](../patches/patch-descriptions.md#format)
  bans them from. Kerneldoc comments in code keep their `%` markup.
- Terms:
  - "Work item" for a work_struct, never "work slot".
  - "Inline" means inlined by the compiler. Otherwise say "synchronously",
    "directly" or "without deferring".
  - "Block" and "wait" only for paths that sleep. A path that returns
    -EBUSY rejects.
  - A loop that retakes a lock to recheck state "cycles the lock", never
    "spins", because spinning reads as a lockless busy-wait on a value.
  - Function names with `()`. Callbacks of an operations table as
    `ops.name()`, with variants combined as `ops.a/b/c()`.
  - Standard kernel terms over a hedged paraphrase: "IRQ context", not
    "normal (IRQ-disabled but non-NMI) context". A subsystem's accepted
    shorthand is fine in its own prose, such as "sub-sched" in sched_ext.
- Concrete, not vague, and only what is known: "has a steady supply of
  softirq work, such as incoming packets", not "busy with softirq". If half
  of a scenario is speculation, state the half that is known.
- State the mechanism, not the variables: "the loop ran millions of
  iterations", not "next_seq - seq diverging by millions". Variable names are
  jargon outside the code that owns them.
- No unnecessary specificity. Name the goal, not one of many literal
  commands: "trigger SysRq-t", not `echo t > /proc/sysrq-trigger`. Naming
  the one knob that does the job is fine, such as `log_buf_len`, but
  prescribing its value is not. Don't attribute a warning to one compiler
  when gcc and clang both emit it. When a Closes: or Reported-by: link
  carries the diagnostic, explain the change in concept terms instead of
  quoting the flag again.

### What to leave out
- Padding, which adds words without changing the claim:
  - parenthetical color words, such as "(vacuous)" or "(trivially)"
  - vague scoping qualifiers, such as "the immediate motivation" or "in
    some cases"
  - attribution hedges on facts, such as "documented as safe"
  - a weaker claim beside the stronger one that implies it: "never sees,
    and can't veto" becomes "never sees"
  - a cost quantified when the mechanism already implies it
  - the inverses listed when one contrast word does the work: "directly
    dereferences", not "no helpers, no probe-reads, no error paths"
  - tangential context that doesn't change the argument, such as "(nested
    inside the queue lock)"
  - deferred work described in detail: "X stays as is, to be resolved
    later", and nothing about how
- Length is not the metric, necessity is. Keep every necessary sentence and
  strip every gratuitous one.
- Reassurance: what still works, what is unchanged or that a case is safe,
  unless the reader would otherwise wonder. This holds in descriptions,
  comments and replies alike. The description cases are in
  [patches/patch-descriptions](../patches/patch-descriptions.md#what-stays-out).
- Prose after a numbered list that repeats the list.
- Itemizing. Name the class, such as "every mutable global", instead of
  listing its members. Itemize only when the members are surprising,
  incomplete by design, or something the reader has to find. This holds in
  descriptions, comments and replies alike.
- The specific where the general does. A structural justification beats
  "acceptable": "the disable path it triggers runs from irq_work anyway",
  not "deferring by a few milliseconds is acceptable". The general
  principle beats one failure mode: "foo_lock isn't NMI safe, and grabbing
  it from NMI context can lead to deadlocks". An umbrella beats a list of
  cases: "addresses all memory access bugs".
- Words the reader supplies: "An NMI on a CPU already inside that path
  deadlocks", not "the NMI's re-acquire of foo_lock deadlocks on the same
  CPU". No "into kernel memory" after "out-of-bounds write". Don't
  elaborate on an absence: "Out-parameter writes have no equivalent yet."
  Stop there.
- Show-off tone. No meta-commentary on the agent's own output, such as
  "Final draft (10-line body, no invented terms)". No self-congratulating
  recaps, no slide-deck headers on prose answers ("Real bugs:", "Net
  wins:", "TL;DR:"), no posturing ("The honest position is:"), no
  announcing mistakes avoided, and no chummy validators in drafts under the
  user's name ("Nice catch, the race is real."). Just answer.

### Mechanics
- Periods over semicolons. A semicolon is acceptable as a separator between
  list items that contain commas, in literal shell text, and once to bind
  two tightly coupled clauses. Several in one paragraph is the smell. The
  same goes for `--` joins: split where the second clause is a full
  thought. A parenthetical takes a single `-`, never `--` or an em-dash.
- ASCII only in anything that lands in a tree, a mail, a plan or a doc.
  Math and set symbols become plain English or the C operator, dashes
  become `-`, a comma or a period, smart quotes become `"` or `'`, bullets
  become `-`, an ellipsis becomes `...`, and a non-breaking space becomes a
  space. A character that isn't on a US keyboard is wrong.
- One space after a period, even in files that use two.
- Plain text, no escaping: `>100%` and `&` stay as they are.

## Pitfalls
- A commit message went out as one 12-line paragraph holding two points.
  Split at each point boundary before showing, whatever the artifact.
- A comment fused two facts into one clause chain and read as cryptic. Two
  facts are two sentences, each at the place it belongs.
- After corrections about compressed sentences, a draft split every fact
  into its own sentence and read as choppy. Related facts may share a
  sentence that reads naturally.
- An agent's nickname for a code path matched a function that used the
  same word for a different operation, and the nickname reached a
  committed model before the user objected. Use the code's own name from
  the first mention.
- A description listed four kinds of state that a patch converted, when the
  patch converted every mutable global. The class name said it all.

## Customization
- The overlay collects the words and patterns the user struck from drafts,
  with the user's own examples, and the shorthand the user's subsystems
  accept.
- The 8-line paragraph limit and the sentence counts in [Length](#length)
  are this preset's taste. An overlay can change them.
