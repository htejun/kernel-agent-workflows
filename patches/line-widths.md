# Line widths and fill

- **Load when:** writing or editing patch descriptions, code comments or
  code, or auditing widths before a commit or a posting.
- **Parameters:** `$WIDTHS`, `$KAW_DIR`, `$SCRATCH`
- **Depends on:** [core/git-safety](../core/git-safety.md)

$WIDTHS holds three column limits, in order: patch description bodies,
code comments and kerneldoc, and code.

## Always-on rules
- Fill descriptions, comments and code to $WIDTHS in both directions: no
  line over its limit, and no line that could still take the first word of
  the next line, because a paragraph broken short leaves a ragged edge that
  the next edit reflows, which puts unrelated line changes into that later
  patch.
- Check widths with tools/textfill, never by eye, because fills checked by
  eye drift no matter how often the rule is written down.
- [GATE] Before committing a patch: `textfill check` passes on the staged
  draft, and the edited text was reflowed, scoped and kept ASCII, because
  a fill fixed after the commit needs an amend.
- [GATE] Before posting a patch or patchset: `textfill check` passes on
  every patch, exported into a new, empty directory, because a check of
  the patch just edited misses text written earlier.

## [GATE] Checklist
### Before committing a patch
- [ ] `textfill check` on the staged draft
      ([Checking a commit](#checking-a-commit)) prints nothing, or each
      finding is one of the [Waivers](#waivers).
- [ ] Every paragraph with a word-level edit was reflowed whole.
- [ ] Only text this patch owns was refilled. A contributor's text in an
      apply is never refilled
      ([maintain/apply-patches](../maintain/apply-patches.md#always-on-rules)).
- [ ] clang-format was not run.
- [ ] The added text is ASCII.

### Before posting a patch or patchset
- [ ] `textfill check` over every patch of the patchset, exported into a
      new, empty directory ([Auditing a patchset](#auditing-a-patchset)),
      prints nothing, or each finding is one of the [Waivers](#waivers).

## Procedure
### Filling prose
1. Compose the sentences first, then wrap them with tools/textfill, giving
   the limit from $WIDTHS and the line prefix, such as a tab and ` * ` for
   a comment in a function:

   ```
   $KAW_DIR/tools/textfill fill -w WIDTH -p "$(printf '\t') * " < paragraph.txt
   ```

   Never wrap multi-line prose by hand.
2. After any word-level edit, reflow the whole paragraph. Breaks placed for
   the old word count leave short lines.

### Filling code
1. Count columns with tabs at 8.
2. A statement that fits on one line within the code limit is one line. If
   the first token of a continuation line, plus any trailing operator, fits
   on the line above, pull it up.
3. When parenthesis-aligned continuations force bad packing, consider a
   local variable.

### Checking a commit
1. Write the commit message to a file, `msg`: the subject line, a blank
   line, then the body and the tags.
2. Stage the changes, leaving no unstaged changes in the files the commit
   touches. Build a draft patch from the message and the staged diff, and
   check it, passing the three $WIDTHS values. The options can be left out
   when $WIDTHS has its default. When amending, diff against the parent
   with `git diff --cached HEAD^`.

   ```
   { printf 'Subject: '; cat msg; echo ---; git diff --cached; } > $SCRATCH/<topic>/draft.patch
   $KAW_DIR/tools/textfill check --desc D --comment C --code K $SCRATCH/<topic>/draft.patch
   ```

3. To refill what it flags, run `textfill refill` on the draft from the top
   of the tree. It rewrites the flagged comment paragraphs in the source
   files and the flagged description paragraphs in the draft. Stage the
   source files again and take the message back from the draft:

   ```
   sed -e '1s/^Subject: //' -e '/^---$/,$d' $SCRATCH/<topic>/draft.patch > msg
   ```

4. Commit with `git commit -F msg`, or `git commit --amend -F msg` for an
   existing commit, under the gate in
   [core/git-safety](../core/git-safety.md#before-a-commit-or-amend).

### Auditing a patchset
1. Export each commit of the patchset into a new, empty directory, so that
   no patch file from an earlier run gets checked:

   ```
   d=$(mktemp -d $SCRATCH/<topic>/fill.XXXXXX)
   git format-patch -o "$d" BASE..HEAD
   ```

2. Check them all, passing the three $WIDTHS values:

   ```
   $KAW_DIR/tools/textfill check --desc D --comment C --code K "$d"/*.patch
   ```

3. Any output means the work isn't done. Fix each finding in its commit or
   confirm that it is one of the [Waivers](#waivers).

## Reference
### Limits
- The description limit applies to the body. Subject lines have no limit.
  Reference list lines such as `[N] <sha12> ("subject")` may run over, and
  textfill skips them.
- The comment limit counts total columns, including the indentation as
  rendered with tabs at 8.
- A repository's .clang-format is not the authority on widths, and
  clang-format is never run on a patch, because it rewraps to its own limit
  and touches text the patch doesn't own.
- rustfmt owns the formatting of Rust code. textfill checks C code only.
- How checkpatch's description width warning is treated is in
  [review/checkpatch](../review/checkpatch.md#ignored-warnings).

### Waivers
- A print call whose format string pushes the line past the code limit
  stays on one line: printk(), the pr_*() family, fprintf() and similar.
  A message split across lines can't be found with grep, and no break of a
  string literal keeps the line short without splitting the message. Waive
  such findings by eye. Everything else textfill reports stands.
- Lines that follow a convention of the surrounding code, such as long
  one-line stubs or tail comments on enum values, are checked by eye and
  waived, not changed.

### Scope
- Refill only text the current patch owns. Never refill comments owned by
  other patches or text that was there before.
- In an aligned block, such as enum values, struct fields or a table, an
  edit that shifts the alignment column realigns the whole block instead of
  leaving one row at minimum spacing. Don't introduce alignment where there
  was none.

### tools/textfill
- `check PATCH...` reads patch files of one commit each, with plain From:
  and Subject: headers or as git format-patch writes them. It checks the
  description and the comment and C code lines the patch adds, in both
  directions, and exits 1 on findings.
- It also flags a `{` alone on an added C line where
  Documentation/process/coding-style.rst puts the brace at the end of the
  line before: after a control statement header, even one that spans several
  lines, after `else`, `do` or `=`, or after another header, such as a
  struct, union or enum one. `--no-braces` turns this check off. After a
  line that ends in `;`, `}`, `:` or a block's `{`, the brace opens a naked
  block scope, and `--no-naked-blocks` turns that check off. A brace after a
  function header or one that opens an initializer element is left alone.
- It fills prose in blocks: a paragraph, or a bullet or numbered item with
  its continuation lines, in descriptions and comments alike. It skips
  tags, reference list lines and preformatted lines. A preformatted line is
  indented and neither a bullet nor a bullet's continuation, such as an
  oops trace or a log in a description, or a diagram in a comment. A line
  that holds one unbreakable token, such as a URL, may run over.
- `fill -w WIDTH -p PREFIX` fills one paragraph from standard input.
- `refill PATCH` rewrites the flagged comment paragraphs in the source
  files and the flagged description paragraphs in PATCH. Run it from the
  top of the tree, with the working tree at the patch's result.
- `--desc`, `--comment` and `--code` set the limits, 76, 80 and 94 by
  default. `--tag NAME` adds a tag name to the ones it skips, which are
  Signed-off-by:, the review and attribution tags, Cc:, Link:, Fixes: and
  Closes:.

## Customization
- $WIDTHS sets the three limits. An overlay gives a tree that uses other
  limits its own.
- An overlay can add tag names for `--tag` and print functions to waive.
