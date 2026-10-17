# Contributing

These rules apply to every change to kernel-agent-workflows, whether a person
or an agent makes it.

## Everything here is generic

This repository is meant to be public. Every file and every commit must be
useful to any kernel developer and must say nothing about a particular
person, machine, organization or piece of private work. Never add:

- names of people, including in incidents, examples and credits
- email addresses, except public mailing lists and reserved example
  addresses such as `user@example.org`
- hostnames, IP addresses, user names or home directory paths
- account names, key names, token names, or where credentials are kept
- details of particular machines: model, CPU count, topology, location
- names of employers, teams or internal projects
- content from private mail or chat
- work that is not public yet: unposted patches, internal designs, branch
  names of private work
- dates

Write examples with placeholders: `example.org` for mail, `box1` for a test
machine, and profile parameters for paths and values. Code in public kernel
trees may be cited by function, file and commit. The URL of a public project,
such as a tool's repository, may be cited even when it contains a user name.

Personal facts belong in a user's profile and overlays. When a rule came from
an incident, keep the lesson and drop every identifying detail: "an
acknowledgment once went out three times because the send command was re-run
to inspect it."

The tools catch known names and patterns. They can't catch a paraphrase, such
as a description of someone's machine, so read every diff for private facts
before committing.

## Always-on rules

- kernel-agent-workflows is generic, because it is meant to be public: no
  names of people, addresses, hosts, paths, accounts, credentials, machine
  details, employers, private mail, unposted work or dates. Personal facts
  go to a profile or an overlay ([details](#everything-here-is-generic)).
- Before every commit to kernel-agent-workflows, run its tools/check-leaks
  and tools/check-modules, and read the diff for private facts, because the
  tools miss paraphrases.
- In kernel-agent-workflows, state each rule in exactly one module and link
  to it from elsewhere, so that a change lands in one place
  ([details](#modules)).

## Modules

A module is one markdown file under an area directory. It covers one unit of
work that a user can enable or disable, such as applying patches or sending
mail. TEMPLATE.md is the skeleton.

- Name a module after its work, in lowercase words joined by hyphens:
  `apply-patches`, never a step number. Names are unique across areas
  because overlays are named after them.
- The title is in sentence case. One or two sentences may follow the header
  fields when the title doesn't say what the module covers.
- The header fields come right after the title:
  - Load when: the situation that calls for the module. INDEX.md is built
    from these lines, so write them for matching against a task.
  - Parameters: every parameter the module uses, and no others.
  - Depends on: the modules whose rules this module's steps rely on, as
    links. An agent reads a dependency when a step links to it. A module
    outside the opt-in areas never depends on an opt-in module.
- The sections, each optional, in this order:
  - Always-on rules: the one-line forms of the rules an agent keeps in mind
    on every task, each with its reason. CORE.md is built from them, so keep
    them few and short.
  - [GATE] Checklist: conditions to verify before a named action, grouped
    under "### Before <action>". Each group has an Always-on rule that
    starts with "[GATE] Before <action>:", so that every gate reaches
    CORE.md.
  - Procedure: steps in order.
  - Reference: facts and idioms with no order.
  - Pitfalls: failure modes and anonymized incidents, each with its lesson.
  - Customization: what profile parameters and overlays usually change.
- A module needs a Procedure or a Reference section.
- Modules outside style/ hold the conventions of the work. A style preset
  adds voice and taste beyond them, and wins where the two differ.
- State a rule once, in the module that owns the action. Elsewhere, link to
  that module's section with a relative link of the form
  `../<area>/<module>.md#<section>`, even from the same area, so that links
  read the same everywhere.
- Until a module exists, name it in plain text, such as
  `review/patchset-review`. `tools/check-modules --strict` reports such
  names, and the published repository passes it.
- Modules don't assume a patch-management tool. Developers use plain git,
  stacked git, quilt and others. Modules describe patch work with git
  commits and branches. Steps for one tool go in that tool's opt-in area,
  such as quilt/. tools/check-modules enforces this.

## Parameters

Values that differ between users are parameters, written `$UPPER_SNAKE`.
Each one is defined in profile.example.md with its default, or marked
required, and a description. Users set their own values in their profile.

- Never write a user's actual value into a module, not even as an example.
- In code blocks and code spans, a `$NAME` that is not a parameter is a shell
  variable. Never give a shell variable a parameter's name.
- Defaults are the established practice these workflows come from. Where
  that practice differs from upstream kernel documentation, the module's
  Customization section names the upstream form for trees that want it.

## Writing

- Plain sentences, one fact each, with the actor named. Each rule carries its
  reason in a sentence.
- Imperative mood for steps.
- One word for each thing: CPU and per-CPU, patchset, post (not submit),
  test machine, and gate only for a [GATE] rule. A check the kernel runs,
  such as a WARN, is a validation check.
- No history in module text: no dates, no rule tags, no "used to". Git log
  and UPGRADING.md hold history.
- ASCII only, no tabs, no trailing whitespace. Prose fills to 80 columns and
  code blocks to 100. Tables and single unbreakable tokens are exempt.

## Tools

- `tools/install-hooks` installs pre-commit, commit-msg and pre-push hooks
  in your clone that run the two checks below.
- `tools/check-leaks` scans files, staged content, commit messages, commit
  identities and history for personal information. Point
  `git config kaw.denylist` at a private deny-list kept outside the
  repository that lists your own names, hosts and accounts. Set
  `git config kaw.identity "Name <address>"` to require that identity on
  every commit.
- `tools/check-modules` checks module structure, parameters, links and
  widths. `tools/check-modules --write` regenerates INDEX.md and CORE.md.
  Never edit those two by hand.
- `tools/overlay-status <overlay dir>` reports what a user's overlays have
  collected for flushing.

## Flushing overlays

Most changes start as rules a user taught their agent, collected in the
user's overlays. When enough have collected, the agent flushes them into the
modules ([core/overlays](core/overlays.md#flushing)): each generic rule is
rewritten for any kernel developer under the rules of this file, local
facts stay in the overlay, and the result goes up as a pull request. A
flush is reviewed like any other change, with the leak check first.

## Commits

- Subjects are `area/module: Summary` for a module and `repo: Summary` for
  everything else.
- A change that needs action in users' profiles or overlays adds an
  UPGRADING.md entry in the same commit: a new module, a new or renamed
  parameter, a renamed, split or removed module, or a changed default.
