# Module template

Copy the block below to start a module. CONTRIBUTING.md explains each part.

```markdown
# Title in sentence case

- **Load when:** the situation that calls for this module, in one sentence.
- **Parameters:** `$PARAM_ONE`, `$PARAM_TWO` (or none)
- **Depends on:** [area/module](../area/module.md) (or none)

One or two sentences on what the module covers, when the title doesn't say it.

## Always-on rules
- The one-line form of a rule the agent keeps in mind on every task, with its
  reason. Each bullet is copied into CORE.md.
- [GATE] Before <action>: the one-line form of the checklist below.

## [GATE] Checklist
### Before <action>
- [ ] A condition to verify before the action, every time.

## Procedure
1. Steps, in order.

## Reference
Facts and idioms that have no order.

## Pitfalls
- A failure mode, how it shows, and what to do instead. Incidents are
  anonymized.

## Customization
- What a profile parameter or an overlay usually changes here.
```

## Overlay template

A user's overlay for a module lives outside this repository, in the
overlay directory, named after the module's file. core/overlays.md says
what goes in each section.

```markdown
# <module> overlay

## Local facts
- A machine, an account, a path, a person or a local exception that the
  module leaves out.

## Pending flush
- <date>: a rule learned in use, in plain words, with the user's own words
  when they gave them. It waits here until it is flushed into the module.
```
