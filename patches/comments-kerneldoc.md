# Code comments and kerneldoc

- **Load when:** writing, revising or reviewing code comments or kerneldoc
  in a patch.
- **Parameters:** none
- **Depends on:** [patches/line-widths](../patches/line-widths.md)

## Always-on rules
- A comment states what the code can't show: the contract, the invariant,
  the constraint or the reason behind a non-obvious choice. Never a
  narration of the code, because narration goes stale with the code and
  buries the fact the reader needs.
- A comment gets its final wording in the patch that introduces it,
  because a rewrite by a later patch in the same patchset makes reviewers
  read it twice and leaves interim text in the history.
- [GATE] Before committing a patch: no added comment narrates the code or
  refers to sibling patches or future plans, because each goes stale as
  soon as the code or the patchset changes.

## [GATE] Checklist
### Before committing a patch
- [ ] No added comment narrates the code
      ([What stays out](#what-stays-out)).
- [ ] No added comment refers to sibling patches or future plans.

## Procedure
### Checking comments in a patch
1. Before showing a patch, go over every comment it adds or changes, not
   only the one a reviewer called out.
2. Each sentence states a fact or an instruction, with its actor named.
3. The invariant is stated at the check that relies on it.
4. No clause repeats what the code already shows, and nothing refers to
   sibling patches or future plans.
5. Widths pass `textfill check`
   ([patches/line-widths](../patches/line-widths.md#auditing-a-patchset)).

## Reference
### What a comment is for
- The hidden invariant or contract callers depend on, the reason behind a
  non-obvious choice, locking and context preconditions, edge-case
  semantics the type doesn't show, and what a return value means when it
  has several causes.
- Every sentence states a fact. "Use beside the allocation." parses as a
  sentence and states nothing. Either say the fact or delete the sentence.

### What stays out
- A play-by-play of the body, a restatement of the signature, or a line
  that is true of any function with that name.
- References to sibling patches or future plans, such as "Later patches
  add -EPERM". Describe today's behavior.
- A comment on a helper whose name and short body say it all, and a
  paraphrase of the next call.
- A clause the reader can see from the code.

### Where to say it
- At a check, state the invariant that holds there and what a violation
  means, not how the situation arises.
- At a use site, cite the contract the layer boundary documents, after
  checking that it covers the site. Never re-derive it from another
  layer's internals, explain how an underlying framework works, or add
  footnotes that teach another subsystem's semantics, such as "(IRQs
  disabled qualifies)" after "RCU read-side", because that mechanism
  belongs to the other layer and can change under the comment. The one
  exception is a helper whose purpose is the translation, and it says only
  what it produces.
- A lockdep assertion checks the lock that protects the data, not a lock
  usually held with it.
- A declaration of a function that has kerneldoc elsewhere gets no comment
  on what the function does, because the kerneldoc is the description. At
  most one line says why the declaration is out of place, such as "declared
  here until the headers are synced".
- A barrier or race comment says who wrote what and what the reader on the
  other side sees, one fact per short sentence, in plain words. Never
  compress the mechanism into a noun phrase the reader has to unpack.
- Enum and flag entries get one-line meanings. Protocol detail, such as who
  sets and clears a flag, under which lock, and the race windows, lives in
  the function that enforces it, and the entry refers to that function
  ("see foo_exit()"). Don't lengthen neighboring one-liners for symmetry.
- Never two adjacent comment blocks. Merge and organize them.

### Comments across a patchset
- When swapping a mechanism in an existing comment, such as a flag for a
  state or a lock for an atomic, keep the sentences that say why and edit
  only what changed. Restore any fact the new version lost. A new case gets
  a new paragraph, and text that is still accurate stays.
- When a later patch in the patchset changes behavior an earlier patch
  would document, the comment or kerneldoc goes in with its final wording:
  in the later patch, or in final form in the earlier patch if that patch
  introduces the text. The earlier patch's description states the interim
  behavior in one sentence without naming the later patch. Code the earlier
  patch needs for its own correctness stays, with comments local to that
  code, and the later patch removes the code and its comments together.

### Form
- A multi-line comment puts `/*` and `*/` on their own lines, except where
  the file already uses the other form, such as BPF uapi headers and the BPF
  subsystem. Match the surrounding file. A closing `*/` glued to the last
  line of text is wrong everywhere.
- Interface functions, such as inline functions in headers and functions
  used from other files, get kerneldoc. So does any substantial function
  comment, static helpers included, as kernel-doc.rst recommends. A trivial
  helper with a one-line note keeps a plain comment.

### Kerneldoc
- Follow Documentation/doc-guide/kernel-doc.rst. The summary line of a
  function is `function_name() - Brief description`, and that of a type is
  `struct name - Brief description`. Match the capitalization of the
  summaries already in the file.
- The summary is imperative: "Destroy @cgrp", not "This function destroys
  @cgrp".
- Describe the return value in a `Return:` section, as kernel-doc.rst
  does.
- Kerneldoc doesn't justify the function's existence. "Allows callers to
  obtain X without poking at the opaque struct" restates the signature.
- Kerneldoc keeps its `%CONST` and `@arg` markup.

## Customization
- An overlay can record a subsystem's comment form, the user's kerneldoc
  conventions and examples from the user's own trees.
