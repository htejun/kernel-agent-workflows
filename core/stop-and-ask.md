# Stop and ask

- **Load when:** a step fails, surprises, or needs a choice, and the agent
  must decide whether to solve the problem or to ask the user.
- **Parameters:** none
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md)

## Always-on rules
- Solve mechanics and stop only for genuine blockers, because asking about
  a solvable problem wastes the user's time and guessing past a blocker does
  damage.
- When stopping, never hide the problem by skipping, narrowing or forcing a
  step, because the user decides what to do next from what actually failed.

## Procedure
1. When a step surprises, read the module that owns the step before
   diagnosing anything.
2. Decide whether the problem is mechanics or a blocker, using the list of
   blockers below.
3. Mechanics: solve it and carry on. Populating a test VM, a shell quirk, a
   tool flag and a stale build artifact are all mechanics.
4. Blocker: stop and report the step that failed, the exact state it left
   behind, what is known and what is unknown, and two or three options.
   Keep any workaround out of the question.

## Reference
### Genuine blockers
- A destructive or irreversible operation.
- A risk of losing data.
- An ambiguous target, such as which branch a patch goes to.
- A root cause that is not proven
  ([core/evidence-discipline](../core/evidence-discipline.md)).
- An operation that changes branch structure, such as a merge between
  integration branches or a rewrite of published history.
- A step that should have been clean and wasn't: a patch that applies only
  with offset or fuzz, an unexpected merge conflict, or a failed apply.
  During a rework of its own patch stack that the user asked for, the agent
  resolves its own apply failures.

### Standard flows
The build, merge, push and acknowledgment steps of a flow the user
authorized run without a pause per step. They stop only at the gates their
modules define and at genuine blockers.
