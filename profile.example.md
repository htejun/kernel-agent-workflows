# Profile example

A profile holds one user's values for the parameters the modules use, the
modules the user enables, and the user's aliases. Copy this file outside the
repository and fill it in. INTEGRATION.md walks through it.

Each parameter is a list item: the name, then either its default or
"required", then a description. Leave a parameter unset to take its default.
Parameters of modules you don't enable can stay unset even when required.

## Setup

- `$KAW_DIR`: required. Absolute path of this repository's clone, such as
  `~/src/kernel-agent-workflows`.
- `$OVERLAY_DIR`: required. Directory of overlays, one file per module named
  after the module file, such as `apply-patches.md`.
- `$INTEGRATED_COMMIT`: required. The commit of this repository last
  integrated into the setup. Updating starts from it.
- `$REVIEW_PROMPTS`: required. Path of a clone of
  https://github.com/masoncl/review-prompts, which is pulled on its own
  schedule.

## Modules

- `$MODULES`: default `default`. The modules that apply. `default` means
  every module outside the opt-in areas, minus maintain/ when `$ROLE` is
  `contributor`. Add opt-in modules by name (`default, style/voice`) and
  drop default ones with a minus (`default, -test/vm-testing`).
- `$ALIASES`: default none. Short names for modules, such as
  `review = review/patchset-review`.

## Paths and machines

- `$SCRATCH`: required. Directory for working files such as drafts,
  mailboxes and logs, with a subdirectory per topic, such as
  `~/scratch/<topic>/`.
- `$WORKTREE_ROOT`: required. Directory where a task the user hands off gets
  its own git worktree, such as `~/src`.
- `$PUBLICATION_TREES`: default none. The clones the user publishes from,
  such as the ones that applied patches and pull requests go out of.
- `$ROOT_OK_HOSTS`: default none. Dedicated test machines and VMs where the
  agent may run commands as root without asking for each one.
- `$PLAN_DIR`: default `PLAN/`. Directory, relative to the working tree,
  for plans, research notes and models that go with a piece of work.

## Review

- `$FORMAL_MODEL`: default `off`. `on` makes patchset review check each
  multi-actor protocol over shared state with a TLA+ model. Needs Java and
  the TLA+ tools.
- `$MODEL_CHECK_HOST`: default `local`. Where long model-checking runs go:
  `local`, or a host from `$ROOT_OK_HOSTS` with more cores and memory.

## Role and patch management

- `$ROLE`: default `contributor`. `contributor` or `maintainer`. A maintainer
  also applies other people's patches and sends pull requests.
- `$PATCH_MGMT`: default `git`. How patchsets are kept: `git` for plain
  branches, or the name of another tool such as `quilt` or `stgit`. Modules
  describe patch work with git. Enable the tool's opt-in area if one exists,
  such as quilt/.
