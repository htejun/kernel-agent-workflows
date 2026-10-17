# Overlays

- **Load when:** the user states a new rule, corrects the agent, says
  "remember", or asks to review the overlays or flush them into this
  repository.
- **Parameters:** `$OVERLAY_DIR`, `$KAW_DIR`, `$INTEGRATED_COMMIT`,
  `$WORKTREE_ROOT`
- **Depends on:** [core/publishing](../core/publishing.md),
  [core/git-safety](../core/git-safety.md),
  [submit/github-pr](../submit/github-pr.md)

An overlay is the user's own file for one module. It holds the local facts
the module leaves out and the rules learned in use that are waiting to flow
back into the module.

## Always-on rules
- Record a new rule or a correction from the user in the overlay of the
  module that owns the action, in the same session, because a correction
  that isn't written down gets repeated.
- Never edit a module of this repository for a lesson learned in use. It
  goes into an overlay first and reaches the module through a flush,
  because modules are shared and a change to one needs the generic review.
- After this repository is updated, rebase every overlay onto its changed
  module: the overlay's intention keeps priority and its wording and
  placement adapt to the new module, because the overlay records what the
  user decided ([details](#after-an-update)).

## Procedure
### Recording
1. A parameter value or a module to enable or drop goes into the profile,
   not an overlay.
2. Find the module that owns the action in INDEX.md. When no module covers
   the work, the rule doesn't belong in an overlay: tell the user, so that
   it goes into the user's own instructions instead.
3. Open `$OVERLAY_DIR/<module>.md`, named after the module's file. Create it
   from the overlay template in TEMPLATE.md when it doesn't exist.
4. Put a fact, such as a machine, an account, a path or a person, under
   "Local facts".
5. Put every rule, new or changed, under "Pending flush": the rule in plain
   words, the user's own words when they gave them, and the date. Whether
   it is generic or the user's own is decided at the flush.
6. Follow the rule for the rest of the session, and tell the user which
   overlay it went to.

### Flushing
Flush when the user asks, or suggest it when
`$KAW_DIR/tools/overlay-status $OVERLAY_DIR` shows enough pending items to
be worth a review.
1. Follow [Updating](../INTEGRATION.md#updating) first, so that the pending
   items sit on the current modules.
2. Create a worktree of `$KAW_DIR` under `$WORKTREE_ROOT`, on a new branch
   from the upstream main branch. The clone at `$KAW_DIR` stays on main,
   because every task reads it.
3. For each pending item, decide whether any kernel developer would want
   it. A rule only the user wants stays in the overlay and moves to "Local
   facts". A generic rule is written into the module that owns it in the
   worktree, following CONTRIBUTING.md: no personal details, one home per
   rule, the template and the module's voice.
4. Run `tools/check-leaks` and `tools/check-modules` in the worktree, and
   read the diff for private facts the tools can't catch.
5. Commit under the commit gate
   ([core/git-safety](../core/git-safety.md#before-a-commit-or-amend)).
6. Push the branch, to the user's fork or to the repository when the user
   maintains it, and open a pull request as
   [submit/github-pr](../submit/github-pr.md) describes. Both are
   publishing ([core/publishing](../core/publishing.md)).
7. Mark each flushed item in its overlay with the pull request's URL. A
   marked item stays in force and is not flushed again.
8. After the pull request is merged, follow Updating again. Its rebase drops
   the items the modules now state, and keeps the intention of any item the
   review changed. Remove the worktree.

### After an update
Pulling this repository can change, rename, split or remove modules that
overlays build on. Rebase the overlays onto the new modules the way a patch
is rebased onto a new base: each item keeps its intention, and its form
adapts to what changed.
1. Run `$KAW_DIR/tools/overlay-status $OVERLAY_DIR`. It reports overlays
   whose module no longer exists. Find where the content went from the
   UPGRADING.md entries and the commit log, and move each item to the
   overlay of the module that now owns it.
2. For every module that changed since `$INTEGRATED_COMMIT`
   (`git -C $KAW_DIR diff $INTEGRATED_COMMIT HEAD -- <module>`), read the
   change and its overlay together. For each overlay item, decide:
   - The module now says the same thing: drop the item, because it was
     flushed or adopted upstream.
   - The rule the item builds on moved to another module: move the item to
     that module's overlay.
   - The module changed the rule the item overrides: keep the item's
     intention, and reword it against the new text so that it still
     overrides exactly what the user meant. An item never silently loses
     its effect because the base moved.
   - The module now refers to new sections, parameters or names: update
     the item's references to match.
   - The change leaves the item's intention unclear: ask the user, and keep
     the item as it was until the user answers.
3. Show the user every item that was dropped, reworded or moved, with the
   reason, before updating `$INTEGRATED_COMMIT`.

## Reference
- An overlay is read after its module and its intention always has
  priority. Its "Local facts" hold what the module leaves out on purpose.
  Its "Pending flush" holds rules on their way into the module, and rules
  the user keeps for themselves move to "Local facts" at a flush.
- A local fact never restates a module's rule. A pending rule may change
  one, and wins until it is flushed.
- Every overlay has the two sections of the overlay template in
  TEMPLATE.md. `tools/overlay-status` reports overlays without them.
- Overlays stay outside this repository, so they may name machines,
  accounts and people.

## Customization
- `$OVERLAY_DIR` sets where the overlays live.
- An overlay can record how often the user wants a flush suggested.
