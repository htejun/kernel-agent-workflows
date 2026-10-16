# Integration

Instructions for an agent asked to set up, update or remove these workflows
for its user. Read the whole file before starting. Ask the user whenever a
step needs a choice. Never guess a path, account or address.

## What a setup consists of

- A clone of this repository at `$KAW_DIR`.
- A clone of https://github.com/masoncl/review-prompts at `$REVIEW_PROMPTS`.
  Patchset review loads its kernel prompts. The two repositories are pulled
  separately.
- A profile: one markdown file outside both clones with the user's parameter
  values, `$MODULES` and `$ALIASES`. It starts as a copy of
  profile.example.md.
- An overlay directory, `$OVERLAY_DIR`. An overlay holds the user's local
  facts and exceptions for one module, is read after the module, and wins on
  conflict. It never restates the module's rules.
- The include block below, in the instruction file of each agent harness the
  user runs, such as CLAUDE.md for Claude Code and AGENTS.md for Codex.

## Setting up

1. Survey. Find the instruction files the user's agents load at startup and
   the workflow documents they point to. Read them, and list the rules,
   procedures and local facts that overlap with modules here.
2. Choose modules. Explain the areas in README.md and ask which opt-in
   modules to enable and which default modules to drop. Leave out any module
   the user declines. Record the answer as `$MODULES`.
3. Interview. For each enabled module, ask for every parameter in its
   Parameters field, using the description in profile.example.md. Ask a few
   questions at a time, in this order: `$ROLE` and `$PATCH_MGMT`, trees and
   branches, mail, gates, testing, the rest. Offer the default where one
   exists.
4. Back up every file this setup will edit, such as the instruction files
   and workflow documents, next to the profile.
5. Write the profile. Copy profile.example.md, fill in the answers, and set
   `$INTEGRATED_COMMIT` to the output of `git -C $KAW_DIR rev-parse HEAD`.
6. Write overlays for the local facts the survey and the interview turned
   up: machine details, accounts, people, paths, local exceptions. Keep each
   fact in exactly one place.
7. Resolve conflicts. For each existing rule that contradicts an enabled
   module, show both to the user and ask which wins. Record a local
   exception in the module's overlay. Change or remove the user's old rule
   only with the user's approval.
8. Add the include block to each instruction file.
9. Check. Pick two or three tasks the user does often. For each, describe
   which modules load, which gates apply and which parameter values are
   used, and have the user confirm.

## The include block

Replace `<kaw dir>` and `<profile>` with absolute paths.

```
## Kernel workflows
At the start of every task, read <kaw dir>/CORE.md, <kaw dir>/INDEX.md and
<profile>. Follow INDEX.md to match tasks to modules. Names written
`$UPPER_SNAKE` in those files and in modules are parameters set in the
profile.
```

## Updating

1. Pull: `git -C $KAW_DIR pull`. Pull review-prompts too, when the user
   wants.
2. Read the upgrade entries added since the last update:
   `git -C $KAW_DIR diff $INTEGRATED_COMMIT HEAD -- UPGRADING.md`. Act on
   each entry, and ask the user about new parameters and new modules.
3. List the added and changed modules with
   `git -C $KAW_DIR diff --stat $INTEGRATED_COMMIT HEAD`, and show the user
   the changes to CORE.md. For each enabled module that changed, check its
   overlay and raise any line that now repeats or contradicts the module.
4. Set `$INTEGRATED_COMMIT` to the new HEAD.

## Removing

Remove the include block from each instruction file, and restore any rule
that setup changed from the backup made in step 4. The profile and overlays
can stay for later or be deleted.
