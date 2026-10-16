# Evidence discipline

- **Load when:** diagnosing a failure, stating a cause, or making a claim
  about code, data or a past event.
- **Parameters:** none
- **Depends on:** none

## Always-on rules
- A claim is either proven with evidence or reported as unknown, with more
  data gathered when the task needs the cause. Never fill the gap with a
  plausible story, even one that later turns out right, because a theory
  stated as the answer gets built into designs and patches.
- A root cause must fit every constraint of the report before it is called
  the cause, because a cause that misses one explains a different bug and
  is only a finding.
- Copy identifiers from their source: message IDs, URLs, commit IDs, and
  function and API names. When one can't be checked, describe the mechanism
  in general terms instead, because a reconstructed identifier is often
  subtly wrong and sends the reader to the wrong place.

## Procedure
### Stating a cause
1. Write down the report's full set of constraints: the configuration it
   happened in, the exact symptom, and how often it happens.
2. Check the candidate cause against every constraint. It must be reachable
   in the reported configuration, including Kconfig dependencies and build
   exclusions. It must produce the exact symptom: a callback that never runs
   needs a cause that skips the call, not one that makes it from the wrong
   context. It must fit the reported frequency.
3. If the candidate passes, state it with the evidence that shows it. If it
   fails, record it as a finding and keep looking, or report the cause as
   unknown along with what the evidence has ruled out.
4. Where the conclusion is stated, label every link that rests on an
   assumption: "assumed, not traced: ...".

### When a claim is challenged
1. Trace through the code or the data before answering. Never restate the
   claim more confidently.
2. If the claim doesn't survive, say which part was not backed by evidence.
   "I think X but haven't verified it" beats asserting X.

## Reference
### What counts as evidence
- Hard evidence admits no other reading: a trace or probe of the exact event
  in the claimed mechanism, the detector itself firing (a hung-task report,
  a WARN, a lockdep splat), or a state dump taken at the moment.
- Timing, durations, counts, A/B rates and "it went away with the fix" are
  indirect. They can prompt a hypothesis but never prove one.
- A snapshot shows state, not the order of events. When two explanations fit
  the data, keep both. If the task needs the cause and its budget allows,
  add a trace that tells them apart. Otherwise report the limits of the
  observation.
- A reproducer counts only when the detector fires and a trace shows the
  claimed mechanism. A symptom alone proves only the symptom.
- Timestamps from different sources need a common timebase before they can
  be compared.
- A failed search is evidence about the search pattern, not about the
  material. Search to locate, then read what was found before concluding
  that something is absent.

### How much proof a task needs
Evidence discipline governs what may be claimed. It does not demand
exhaustive proof. An overall assessment may end with an unresolved cause,
reported as one. A causal answer is required only when the task asks for
it, within the task's effort budget
([core/validation-scope](../core/validation-scope.md)). The tracing method,
with its hypothesis ledger and stop criteria, is in
[debug/event-tracing](../debug/event-tracing.md).

## Pitfalls
- Hedged causes. "Looks like" or "probably" in front of a cause marks a
  guess. Get the evidence or call the cause unknown.
- Settling for any mechanism. When tracing finds no bug, don't adopt a
  mechanism because it produces a number of the right size. Report "not
  found yet" and what was ruled out.
- Treating a downstream fix as a root cause. Clamping a metric so that it
  looks sane hides the overrun that produced it.
- Calling a hang reproduced without tracing it. A hang was once declared
  reproduced from slow cycles and a hung-task report before the operation
  that was supposed to stall had been traced. A detector firing shows that
  something hung, not which path.
