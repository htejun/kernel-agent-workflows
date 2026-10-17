# Safe operations

- **Load when:** running commands that delete, overwrite or force anything,
  working across several repositories, running commands as root, writing
  working files, or touching a publication tree.
- **Parameters:** `$SCRATCH`, `$WORKTREE_ROOT`, `$PUBLICATION_TREES`,
  `$ROOT_OK_HOSTS`
- **Depends on:** [core/stop-and-ask](../core/stop-and-ask.md)

## Always-on rules
- [GATE] Before a destructive or forced operation: look at the target, find
  out why the normal operation failed, and get the user's approval, because
  forced and destructive operations discard state that can't be recovered.
- Never re-run a state-changing command to inspect its result, because it
  acts again: a send sends again and a push pushes again. Inspect the
  resulting state instead.
- Never build, test or experiment in $PUBLICATION_TREES, because the user
  publishes from them and may be working in them live.
- Ask before every command that runs as root, showing the exact command,
  even when the request implies it, except on $ROOT_OK_HOSTS, because a
  privileged mistake on a shared machine can't be undone.
- Keep working files under $SCRATCH, never in /tmp or a per-session
  directory, because a remount or the end of a session loses them.
- [GATE] Before editing a code or patch file: the edit is an explicit
  replacement in that one file, never sed, awk or perl run in place,
  because a pattern that matches more than intended changes code nobody
  reviewed.
- [GATE] Before a shell command that targets a repository: it starts in
  that repository, touches no second one, and runs alone if it changes
  directory, because parallel calls can share one shell and the last
  change of directory wins.
- Never edit a generated file by hand. Change its inputs or its generator,
  regenerate it, and check the generated diff for unrelated changes.
- Work the user and the agent do together happens in place. A task the
  user sends the agent off to do gets its own git worktree under
  $WORKTREE_ROOT.

## [GATE] Checklist
### Before a destructive or forced operation
- [ ] The normal operation was tried, and why it failed is understood.
- [ ] The target was inspected: its contents, and whether a setting
      redirects it to a location other repositories or users share.
- [ ] Anything that can't be regenerated is backed up under $SCRATCH.
- [ ] The user approved the operation, unless the task the user gave
      already names it
      ([core/stop-and-ask](../core/stop-and-ask.md#genuine-blockers)).

### Before editing a code or patch file
- [ ] The edit is an explicit replacement made with the editor tool, one
      file at a time.
- [ ] A replace-all covers only a pattern verified unambiguous in that one
      file.
- [ ] No sed, awk or perl runs in place over the file. A rename across
      several files is one explicit edit per file.

### Before a shell command that targets a repository
- [ ] The command starts by changing to that repository's directory, or
      passes the directory with `git -C`.
- [ ] It chains no command for a second repository.
- [ ] If it changes directory, it runs alone, never in parallel with other
      calls.
- [ ] After "pathspec did not match", "does not exist in index" or a reset
      working directory, the work stopped and the current directory was
      checked, because a command may have run in the wrong place.

## Reference
### Working files
- Create the topic's directory under $SCRATCH at the start of a task, and
  check each path before writing to it.

### Publication trees
Publication trees are for reading code, looking up branches and commit IDs,
and preparing and sending patches and pull requests. No builds, config
changes, instrumentation, VM runs or test checkouts happen there. The builds
that are part of publishing do run there: the build of a patch just applied
to the tree, and the test merge and build of a pull request.

### Commands to treat with care
- Scripts don't run sudo themselves unless the user asks for it, so that
  elevation stays visible at the command the user approves.
- Commands that discard work without a prompt include `git reset --hard`,
  `git checkout -f`, `git clean -f`, and the force options of patch-stack
  tools.

## Pitfalls
- A path resolved through configuration can point outside the current
  repository. An installer once wrote git hooks into the configured hooks
  directory, which a global setting pointed at hooks shared by every
  repository, and replaced them. Resolve the target and look at it before
  writing.

## Customization
- `$PUBLICATION_TREES` and `$ROOT_OK_HOSTS` list the user's trees and
  dedicated test machines.
- An overlay can record worktree naming conventions and what the user has
  authorized on each test machine.
