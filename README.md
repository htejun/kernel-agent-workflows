# kernel-agent-workflows

Workflows for AI coding agents that work on the Linux kernel with you:
writing and posting patches, reviewing them before they go out, maintaining a
tree, testing kernels and debugging failures. The workflows are plain
markdown files. Any agent harness that follows instructions from files can
use them, including Claude Code and Codex. The repository is kaw for short,
as in the `kaw.*` git config keys its tools read. It lives at
https://github.com/htejun/kernel-agent-workflows.

They come from daily kernel development and maintenance. Most rules exist
because something went wrong without them: a mail sent twice, a patch
applied to the wrong branch, a theory reported as a root cause. Take what
helps you and leave the rest.

## What's in it

Workflows are grouped into areas. Each workflow is one file, called a module,
and you enable the modules you want.

| Area | Covers |
|---|---|
| `core/` | how the agent reasons, how much to validate, when it asks, how it waits, risky operations |
| `patches/` | patch descriptions, code comments, cover letters, tags, line widths |
| `review/` | checkpatch, build checks, patchset review, formal model checks |
| `mail/` | reading threads from the public archives, composing mail, threading, writing replies |
| `mailbox/` | sending mail, the sent log, reading your inbox (opt-in) |
| `submit/` | posting patches and new versions, dependencies, GitHub pull requests |
| `maintain/` | branch model, applying patches, git safety, merges, pull requests |
| `test/` | machine access, test result layout, remote test machines, VMs, performance comparisons |
| `debug/` | crash analysis, bug report triage, event tracing |
| `code/` | kernel C idioms that are easy to get wrong |
| `style/` | opinionated writing and coding style presets (opt-in) |
| `quilt/` | patch stacks with quilt on top of git (opt-in) |
| `subsys/` | subsystem rules for BPF and sched_ext (opt-in) |
| `agents/` | quirks of specific agent harnesses (opt-in) |

INDEX.md lists every module with one line on when it applies. Modules
describe patch work with plain git. If you keep patchsets with another tool,
enable its area where one exists.

## How it works

```
your agent's instruction file
  |
  +-- include block, added once
        |-- reads CORE.md ............ short rules that apply to every task
        |-- reads INDEX.md ........... when each module applies
        |-- reads your profile ....... your values and enabled modules
        |
        +-- on each task
              |-- matches INDEX.md --> area/module.md
              |-- reads your overlay for that module, if any
              +-- follows the module with your profile's values
```

- CORE.md holds the rules an agent keeps in mind on every task, including
  one line for each gate.
- INDEX.md says when each module applies. The agent matches the task against
  it, names the module and reads it.
- Your profile sets the parameters the modules use, such as your trees or
  how you send mail, and lists the modules you enable.
- An overlay is your own file for one module. It holds your local facts,
  such as the details of your test machines, and the rules you taught the
  agent in use. It is read after the module, and its intention always has
  priority.

Your profile and overlays live outside this repository, so updating is a
`git pull`.

## Getting started

1. Clone this repository and, separately, the kernel review prompts that
   patchset review uses:

   ```
   git clone https://github.com/htejun/kernel-agent-workflows
   git clone https://github.com/masoncl/review-prompts
   ```

2. Ask your agent:

   > Set up the kernel workflows in <path to the clone> by following its
   > INTEGRATION.md.

   The agent reads your existing instructions, asks which modules you want,
   and asks about your trees, mail, test setup and preferences. It backs up
   the files it will touch, writes your profile and overlays, and adds an
   include block to your agent's instruction file (CLAUDE.md for Claude
   Code, AGENTS.md for Codex). Where your existing rules disagree with a
   module, it asks which one wins.

3. Read what it wrote. The profile and overlays are short markdown files you
   can edit by hand.

## Day to day

Ask for what you want done. The agent picks the module, says which one, and
follows it. For example:

- "Apply this series to the fixes branch" loads `maintain/apply-patches`.
- "Post v3 of the series" loads `submit/patchset-posting`,
  `mail/mail-compose` and `core/publishing`, and `mailbox/mail-send` when
  you let the agent send mail.
- "Review the patches before I send them" loads `review/patchset-review`.
- "The test box hung, find out why" loads `debug/crash-debugging`.

Actions that are hard to undo wait for you. The agent prepares a mail or a
push completely, shows you the exact command, and runs it only after you
reply with the send word, `SEND` by default. Without the opt-in mailbox/
area, the agent never sends mail itself: it hands you the files and the
command. Commits wait for the word you choose for them.

## Updating

Ask your agent to update the kernel workflows by following INTEGRATION.md.
It pulls, reads the new entries in UPGRADING.md, asks about new parameters
and modules, and rebases your overlays onto the changed modules: what you
decided keeps priority, adapted to the new text, and anything dropped or
reworded is shown to you. review-prompts updates on its own. Pull it
whenever you like.

## Customizing

- Enable or disable modules in your profile.
- Change a parameter in your profile.
- Tell the agent a new rule or correct it. It records the rule in the
  module's overlay and follows it from then on.
  `tools/overlay-status <overlay dir>` shows what your overlays have
  collected.
- When your overlays have collected rules other developers would want, ask
  the agent to flush them: it generalizes them into the modules and opens a
  pull request (CONTRIBUTING.md).

## Requirements

- The scripts in tools/ need Python 3, git and bash.
- Some modules need more tools when you enable them:
  - b4 and lei to read threads from lore
  - git send-email to build and dry-run patch mail
  - a sendmail-compatible tool such as msmtp to send mail from the opt-in
    mailbox/ area
  - checkpatch.pl from the kernel tree
  - review-prompts for patchset review
  - drgn for crash dumps
  - bpftrace for tracing
  - QEMU or virtme-ng for VM testing
  - the TLA+ tools and herd7 for formal model checks

## Glossary

- **Module**: one workflow file, such as `maintain/apply-patches.md`.
- **Area**: a directory of related modules.
- **Profile**: your file of parameter values, enabled modules and aliases.
- **Parameter**: a value that differs between users, written `$UPPER_SNAKE`
  in modules and set in your profile.
- **Overlay**: your own file for one module: local facts, and rules
  learned in use that wait to be flushed into the module.
- **Alias**: your short name for a module, set in your profile.
- **Include block**: the few lines in your agent's instruction file that
  load these workflows.
- **Gate**: a check the agent must pass before an action, such as the send
  word before mail goes out.
- **Publication tree**: a git tree you publish from, such as the tree your
  pull requests come from.
- **Devel branch**: the branch that collects changes for the next merge
  window.
- **Fixes branch**: the branch that collects fixes for the current release
  cycle.

## Contributing

Everything in this repository is generic: no names, addresses, hosts,
accounts or private work. CONTRIBUTING.md has the rules and the tools that
enforce them.
