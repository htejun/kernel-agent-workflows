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

## Identity and gates

- `$IDENTITY`: required. The name and address used in Signed-off-by and in
  the From of kernel mail, such as `Jane Developer <jane@example.org>`.
- `$SEND_TOKEN`: default `SEND`. The word that must appear on its own in
  the user's latest message before anything is published.
- `$COMMIT_WORD`: default `commit`. The word that authorizes a git commit.

## Trees and builds

- `$TREES`: default none. One entry per tree the user works on: name, path,
  upstream, branch model, the lists and people to copy, and, for
  maintainers, where its pull requests go and its for-next branch, the
  prefix and public URL of its pull request tags, and the branch that
  patches changing only selftests go to.
- `$TREE_STATUS`: default `$KAW_DIR/tools/kernel-tree-status`. The command
  that refreshes the mainline release state and the unsent fixes of each
  maintained tree, and writes them to `$TREE_STATE`.
- `$TREE_STATE`: default `~/.local/state/kernel-tree-status/state.json`
  (under `$XDG_STATE_HOME` when that is set). The state file that
  `$TREE_STATUS` writes. The tool's `--state` option moves it.
- `$BUILD`: default `make -j$(nproc)`. The kernel build command, used for
  every build stage.
- `$WIDTHS`: default `76 80 94`. Column limits for patch descriptions, code
  comments and code. checkpatch warns about description lines over 75
  columns, and that warning is ignored with this default. The code limit is
  above coding-style.rst's preferred 80 columns. Set 80 for trees that keep
  it.

## Mail

- `$MAIL_READ`: default `lore`. How threads are read: `lore` with b4 and
  lei, or the name of a local mailbox setup described in an overlay.
- `$SENDMAIL`: default `msmtp`. The sendmail-compatible command for mail
  that is not git send-email output.
- `$MAIL_ACCOUNTS`: default none. Which sendmail account to use for each
  From address, as `address = account` entries such as
  `jane@example.org = default`. The mail audit gets every entry.
- `$SELF_COPY`: default none. An address every outgoing mail is copied to,
  so that the user keeps a copy of everything sent.
- `$MAIL_AUDIT`: default `$KAW_DIR/tools/mail-audit`. The command that
  checks a mail file before it is sent.
- `$SENT_LOG`: default none. A file where posted patchsets and pull
  requests are recorded.
- `$SUBJECT_STYLE`: default `cover`. How patch subjects carry the version
  and the target. `cover` puts the version and the target tree and branch
  on the cover subject only, `[PATCHSET v2 tree/branch]`. Its patches keep
  `[PATCH i/M]`, and a single patch is `[PATCH tree/branch]`. `standard` is
  the form of the kernel's submitting-patches document: `git format-patch
  -v N` marks the cover and every patch `[PATCH vN i/M]`.

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
