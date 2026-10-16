# Checkpatch

- **Load when:** running scripts/checkpatch.pl on patches before posting or
  applying them, or deciding what to do about a checkpatch warning.
- **Parameters:** `$WIDTHS`
- **Depends on:** [patches/line-widths](../patches/line-widths.md)

## Always-on rules
- checkpatch is advisory: report its findings and never change a patch for
  one without asking, because many of its warnings are false positives.

## Procedure
1. From the top of the kernel tree, run checkpatch on the commits of the
   patchset:

   ```
   scripts/checkpatch.pl -g <base>..HEAD
   ```

   It reads the commits from git and needs no patch files.
2. For patch files, such as a received patchset, pass them all in one
   call: `scripts/checkpatch.pl <dir>/*.patch`. Feed a patch file that git
   tracks in the current repository through stdin instead:
   `scripts/checkpatch.pl - < <file>`. checkpatch's git_is_single_file()
   check treats a tracked file as source code, not as a patch, and reports
   bogus errors such as trailing whitespace on blank context lines.
3. Leave out the [ignored warnings](#ignored-warnings): add
   `--ignore COMMIT_LOG_LONG_LINE` when the description width in `$WIDTHS`
   is above 75, and set the others aside by hand.
4. Report the rest with the patch each came from. Ask before changing a
   patch for any of them.

## Reference
### Ignored warnings
- COMMIT_LOG_LONG_LINE, description lines over 75 columns, when
  descriptions fill to the description width in `$WIDTHS` and pass the
  mechanical audit of
  [patches/line-widths](../patches/line-widths.md#auditing-a-patchset).
- The commit description style error (GIT_COMMIT_ID) on a reference written
  as `<sha12> ("subject")`
  ([patches/commit-tags](../patches/commit-tags.md#commit-references))
  without the word "commit", when the user's style leaves that word out.
  The same error also reports a wrong ID or a subject that doesn't match
  the commit, and those are real, so it isn't passed to `--ignore`.
- VOLATILE, "Use of volatile is usually wrong", in BPF programs, where
  `const volatile` globals hold values that the loader sets before the
  program runs. Kernel code keeps the warning.

## Customization
- `$WIDTHS` decides whether COMMIT_LOG_LONG_LINE is ignored.
- An overlay lists further warnings the user ignores, each with its reason,
  and where the user's format-patch output goes.
