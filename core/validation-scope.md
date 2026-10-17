# Validation scope

- **Load when:** planning or running tests, reviews or measurements, or
  deciding when verification is enough.
- **Parameters:** none
- **Depends on:** [core/evidence-discipline](../core/evidence-discipline.md),
  [test/test-layout](../test/test-layout.md)

## Always-on rules
- Size validation to the decision it supports: choose the coverage, the
  confidence and an effort budget before starting, because checks grown
  without a target turn into a separate project.
- Verify a feature with the quickest adequate check. Existing tests, small
  scripts and temporary probes are fine, because framework integration and
  submission-ready tests are separate work that the user asks for.

## Procedure
1. Name the decision the validation must support and the kind of check it
   needs: a smoke check, an overall regression assessment, a statistical
   comparison, or a causal investigation of a bug. Their completion criteria
   differ.
2. Take the scope and accuracy the user asked for. There is no universal
   time limit, and a time cap given for one campaign is not a standing cap
   for later work.
3. Set an effort budget that counts setup, builds, harness work, diagnosis,
   reviews, evidence handling and reporting.
4. Reuse completed coverage after checking whether the relevant code
   changed. Prefer existing tests and standard benchmarks.
5. Add instrumentation only for an open question that affects the decision.
   Exhaustive branch coverage, rare race coverage and precise confidence
   bounds each need their own reason.
6. Keep one result directory per campaign and report concisely in the
   reply. What the directory records is in
   [test/test-layout](../test/test-layout.md#result-directories).
7. Check progress against the requested result as the work goes. When most
   of the work goes into repairing the harness, rechecking unchanged
   evidence or bookkeeping, stop and reassess: simplify the check, use
   existing observations, or report the limitation.

## Reference
- A bounded assessment can finish with unmeasured cases and unknown causes.
  Keep setup failures, observed regressions and unmeasured coverage apart,
  and never claim equivalence from an inconclusive comparison.
- Bound retries and keep failed attempts. A narrower new check does not turn
  an earlier failure into a pass.
- A new build needs its identity recorded, not nested snapshots or
  manifests. Extra hashes and provenance checks are for a concrete ambiguity
  or an explicit requirement.
- Local test patches stay in the development branch until the user asks to
  drop them.
- Serious kernel failures still stop the affected run, and their logs and
  crash state must be kept. A budget never authorizes unsafe cleanup or
  data loss.
- What may be claimed from the results follows
  [core/evidence-discipline](../core/evidence-discipline.md).
