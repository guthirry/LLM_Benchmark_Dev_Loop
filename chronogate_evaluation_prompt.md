# ChronoGate LLM Benchmark — Independent Evaluation

You are the **independent evaluator** for a software-engineering benchmark completed by two LLM-driven development workflows.

Your task is to thoroughly inspect, test, compare, and score both submissions.

Do **not** assume either submission is correct because its own documentation or test suite says so. Treat all self-reported results as claims that require verification.

---

# Locations

The original benchmark specification is:

```text
/appdata/LLM_Benchmark20260813/chronogate_llm_benchmark.md
```

The two submissions are:

```text
/appdata/LLM_Benchmark20260813/V1/
```

and:

```text
/appdata/LLM_Benchmark20260813/V2/
```

Treat them as anonymized **V1** and **V2**. Do not attempt to infer which underlying model produced either submission.

---

# Primary Objective

Determine which submission demonstrates stronger **software-engineering ability**, not merely which one happens to have more passing tests.

The benchmark is specifically intended to distinguish ability in:

- requirements analysis,
- authority/conflict resolution,
- implementation correctness,
- temporal and interval reasoning,
- idempotency,
- transactional/atomic behavior,
- adversarial test design,
- detection of defective tests,
- defect classification and routing,
- traceability,
- engineering discipline.

The final verdict should therefore consider both the resulting implementation and the quality of the reasoning/evidence embodied in the repository.

---

# Evaluation Procedure

Follow this process in order.

## Phase 1 — Independently understand the benchmark

Read:

```text
/appdata/LLM_Benchmark20260813/chronogate_llm_benchmark.md
```

Before examining either submission in detail, derive your own understanding of:

1. the authoritative requirements,
2. the requirements authority hierarchy,
3. the three intentionally planted lower-authority defects,
4. the expected correct resolution of each defect,
5. the most failure-prone implementation behaviors,
6. the most important adversarial tests that a strong solution should contain.

Do not let either submission's `SPEC.md`, tests, or commentary redefine the benchmark.

The benchmark states that the System Invariants and Formal Behavioral Rules are internally consistent and that there are exactly three planted defects in the lower-authority material. Use that fact when evaluating the PM work.

---

# Phase 2 — Inventory both submissions

Recursively inspect all relevant files beneath:

```text
/appdata/LLM_Benchmark20260813/V1/
```

and:

```text
/appdata/LLM_Benchmark20260813/V2/
```

Identify at minimum:

- corrected specification,
- requirements corrections,
- production source,
- automated tests,
- developer notes,
- test plan,
- test results,
- traceability documentation,
- final review,
- any additional files that materially affect the result.

Do not penalize a submission merely for harmless differences in project structure.

Do penalize missing required artifacts where their absence reduces auditability or demonstrates failure to follow the assignment.

---

# Phase 3 — Evaluate V1 independently

Evaluate V1 against the original benchmark, without comparing it to V2 yet.

Inspect the actual implementation and actual tests.

Do not rely on `TEST_RESULTS.md` alone.

Determine whether:

- its corrected specification is actually correct,
- it found all three planted defects,
- it introduced any unjustified new semantics,
- the production implementation satisfies the authoritative rules,
- its tests actually exercise the difficult behavior,
- any tests encode incorrect lower-authority assumptions,
- rollback is truly atomic,
- idempotency is semantic rather than superficial,
- interval-capacity checking is mathematically correct,
- waitlist promotion is correct,
- timestamps are normalized and compared correctly,
- boundary behavior is correct,
- test failures were classified and routed appropriately,
- documentation accurately reflects the code.

Run the supplied tests if practical.

Record V1's score before doing the comparative analysis.

---

# Phase 4 — Evaluate V2 independently

Repeat exactly the same independent evaluation for V2.

Do not adjust the scoring standard based on what V1 did.

Record V2's score before directly comparing the two.

---

# Phase 5 — Independent verification

The submissions' own tests are not sufficient evidence.

Where practical, construct additional evaluator-owned checks targeting likely failure modes.

You may create temporary evaluation scripts/tests **outside V1 and V2**, but do not modify either submission.

Concentrate particularly on cases likely to distinguish superficially correct implementations from genuinely correct ones.

At minimum, independently investigate the following.

## Requirements authority

Verify that the submission correctly resolves the three planted conflicts instead of:

- following the lower-authority text,
- quietly changing an invariant,
- ignoring the conflict,
- or creating a test that codifies the defective statement.

## Half-open intervals

Test adjacency such as:

```text
[A, B)
[B, C)
```

and exact-boundary cases.

## Capacity greater than one

Use cases where several reservations overlap different portions of a candidate interval.

A naive implementation such as:

```text
number of reservations overlapping candidate < capacity
```

must be detected if present.

Check actual simultaneous occupancy over the interval.

## Timestamp equivalence

Test equivalent instants expressed with different offsets.

Also inspect:

- fractional seconds,
- canonical UTC output,
- naive timestamp rejection,
- comparison by instant rather than string.

## Hold expiry

Pay close attention to:

```text
hold_expires_at == now
```

and the required maintenance-before-command behavior.

## Waitlist start boundary

Pay close attention to a waitlisted reservation where:

```text
start == now
```

during maintenance.

This is deliberately important to the benchmark.

## Waitlist ranking

Verify:

1. priority,
2. `requested_at`,
3. lexical `request_id`.

Also verify that an unfit high-ranked request does not block a lower-ranked request that fits.

## Idempotency

Test retries:

- immediately,
- after confirmation,
- after cancellation,
- after expiration,
- using an equivalent timestamp offset representation,
- using a genuinely different canonical payload.

Verify that `now` is not part of the idempotency payload.

Verify that request IDs are never reusable.

## Atomicity

This deserves especially careful inspection.

A failed call must restore **everything**, potentially including:

- maintenance-expired holds,
- stale waitlist expirations,
- promotions,
- status changes,
- idempotency state,
- last accepted engine time.

Test atomicity for both:

- individual mutating calls,
- `apply_batch`.

A solution that rolls back obvious request changes but fails to undo maintenance side effects is incorrect.

## Batch semantics

Verify:

- maintenance occurs once at batch start,
- commands execute sequentially,
- earlier command effects are visible to later commands,
- cancellation-triggered promotion occurs immediately,
- one later failure rolls back the entire batch,
- rollback includes the initial maintenance.

## Monotonic time

Verify:

- equal `now` is legal,
- earlier `now` is rejected,
- rejected time regression produces no mutation.

## Snapshot

Verify:

- determinism,
- JSON serializability,
- canonical timestamps,
- stable ordering,
- no side effects,
- no implicit maintenance.

---

# Evaluator-Owned Tests

You are encouraged to write additional adversarial tests if this materially improves confidence.

Do not alter either contestant's source or tests to make them pass.

Put evaluator-created files somewhere separate, for example:

```text
/appdata/LLM_Benchmark20260813/evaluator_tmp/
```

Cleanliness of the benchmark submissions must be preserved.

If the two implementations expose slightly different reasonable module layouts, adapt the evaluator harness rather than treating harmless packaging differences as correctness failures.

---

# Scoring Rubric

Score each submission independently out of **100 points**.

Use the following weights.

## 1. Requirements Analysis and PM Work — 20 points

Evaluate:

- correct use of authority hierarchy,
- discovery of all three planted defects,
- correct resolution of those defects,
- precision of `SPEC.md`,
- avoidance of unjustified new requirements,
- quality of `REQUIREMENTS_CORRECTIONS.md`.

Suggested interpretation:

- **18–20:** excellent requirements reasoning; all planted defects correctly identified and resolved.
- **14–17:** largely correct with minor omissions/imprecision.
- **8–13:** meaningful requirements mistakes but partially competent.
- **0–7:** fundamentally misunderstands requirement authority or key semantics.

---

## 2. Production Implementation Correctness — 25 points

Evaluate actual behavior, with particular weight on:

- temporal capacity correctness,
- state transitions,
- maintenance ordering,
- waitlist behavior,
- timestamp semantics,
- idempotency,
- atomic rollback,
- batch semantics,
- deterministic snapshots.

Severe hidden correctness defects should materially reduce this score even if supplied tests pass.

---

## 3. Test Design and Fault Detection — 20 points

Evaluate:

- adversarial quality,
- boundary coverage,
- ability to catch plausible wrong implementations,
- independence from implementation internals,
- whether tests reflect authoritative requirements,
- whether intentionally misleading material was recognized rather than blindly encoded,
- whether rollback and temporal edge cases are genuinely tested.

A very large test suite is not necessarily a good test suite.

Reward discriminating tests over redundant volume.

---

## 4. Engineering Loop and Defect Classification — 15 points

Evaluate evidence that the workflow correctly distinguished:

- implementation defects,
- test defects,
- requirements defects.

Inspect `TEST_RESULTS.md` and supporting repository history/artifacts available in the submission.

Reward cases where a bad test was corrected rather than production code being distorted to satisfy it.

Penalize suspicious "everything passed first try" reporting only when repository evidence suggests the report is inaccurate or superficial. Do not penalize genuine correctness.

---

## 5. Traceability and Auditability — 10 points

Evaluate:

- requirements-to-code traceability,
- requirements-to-test traceability,
- consistency between documentation and implementation,
- usefulness of the final review,
- ability of another engineer to understand why the system behaves as it does.

---

## 6. General Software Engineering Quality — 10 points

Evaluate:

- code clarity,
- cohesion,
- sensible decomposition,
- error handling,
- maintainability,
- unnecessary complexity,
- hidden mutable-state hazards,
- determinism,
- technical discipline.

Do not reward architecture for its own sake.

A small, clear, correct implementation may be better than an elaborate one.

---

# Severity Rules

Not all mistakes are equivalent.

Treat the following as **major defects** because they undermine central benchmark goals:

- wrong resolution of a planted requirement conflict,
- naive capacity checking,
- wrong half-open boundary behavior,
- incorrect hold-expiry semantics,
- incorrect waitlist-start semantics,
- incorrect canonical idempotency,
- incomplete rollback,
- incorrect batch rollback,
- allowing time regression,
- defective waitlist ordering/skipping.

A major defect should have a meaningful scoring impact even if the repository's own tests never expose it.

A cosmetic documentation issue should not have comparable weight.

---

# Do Not Reward Test Gaming

A submission must not receive a high score merely because:

```text
its own implementation passes its own tests
```

Look for coupled mistakes where implementation and tests agree with each other but both disagree with the benchmark.

This is one of the most important parts of the evaluation.

Examples include:

- both code and tests treating expiry as inclusive when the formal rules say otherwise,
- both code and tests comparing timestamp strings rather than instants,
- both code and tests reproducing the misleading illustrative example,
- tests omitting rollback of maintenance effects,
- tests encoding a naive overlap-count algorithm.

---

# Comparative Analysis

Only after independently scoring V1 and V2 should you compare them directly.

Identify:

- where V1 is materially stronger,
- where V2 is materially stronger,
- defects shared by both,
- defects unique to each,
- differences that are stylistic rather than substantive,
- which submission shows stronger engineering judgment,
- which submission has the more trustworthy test suite,
- which submission would be safer to maintain or extend.

If one submission has cleaner code but the other is more correct, correctness takes precedence.

If one submission has more tests but the other's tests are more adversarial and discriminating, recognize that distinction.

---

# Required Final Report

Produce a detailed report at:

```text
/appdata/LLM_Benchmark20260813/EVALUATION.md
```

The report should use this structure.

# ChronoGate Benchmark Evaluation

## Executive Verdict

State:

- winner: `V1`, `V2`, or `TIE`,
- V1 total score,
- V2 total score,
- margin,
- evaluator confidence: `HIGH`, `MEDIUM`, or `LOW`.

Give a concise explanation of what primarily determined the result.

---

## Benchmark Interpretation

Briefly document your independent interpretation of:

- the three planted defects,
- their correct resolutions,
- the highest-risk technical requirements.

This establishes the evaluator's ground truth before discussing submissions.

---

## Scorecard

Use a table:

| Category | Weight | V1 | V2 |
|---|---:|---:|---:|
| Requirements Analysis / PM | 20 | | |
| Implementation Correctness | 25 | | |
| Test Design | 20 | | |
| Engineering Loop / Classification | 15 | | |
| Traceability / Auditability | 10 | | |
| General Engineering Quality | 10 | | |
| **Total** | **100** | **X** | **Y** |

Scores must be justified by subsequent findings.

---

## V1 Evaluation

### Strengths

### Defects and Weaknesses

For each substantive defect, include:

- severity: `CRITICAL`, `MAJOR`, `MODERATE`, or `MINOR`,
- affected requirement,
- relevant file/function/test,
- actual behavior,
- required behavior,
- scoring impact.

### Test Suite Assessment

### Documentation and Workflow Assessment

---

## V2 Evaluation

Use the same structure and level of scrutiny as V1.

---

## Independent Verification Results

Describe evaluator-owned tests or manual code-path analyses.

Prefer a compact table where useful:

| Scenario | V1 | V2 | Correct Behavior |
|---|---|---|---|
| Hold expiry boundary | | | |
| Waitlist start boundary | | | |
| Capacity >1 geometry | | | |
| Idempotency after state change | | | |
| Single-call rollback | | | |
| Batch rollback | | | |
| Offset-equivalent timestamps | | | |
| Waitlist skipping | | | |

Add other scenarios when they materially distinguish the submissions.

---

## Head-to-Head Differences

Identify the most important direct differences between V1 and V2, ranked by significance.

Focus on engineering capability rather than superficial formatting.

---

## Five Most Discriminating Findings

List the five observations that most strongly separate the two submissions.

For each, explain why it is diagnostically useful for judging LLM software-engineering ability.

---

## Final Ranking

Give the final ranking and explain why the winning submission is stronger.

Explicitly state whether the result was driven mainly by:

- requirements reasoning,
- implementation correctness,
- test quality,
- workflow/diagnostic reasoning,
- or a combination.

---

# Final Evaluator Principles

Be rigorous but fair.

Do not search for arbitrary stylistic faults merely to create separation.

Do not award points simply for volume.

Do not trust self-assessment without verification.

Do not alter the submissions.

Do not grade relative to the weaker model. Grade both against the benchmark.

If you discover an actual flaw or ambiguity in the benchmark itself, document it separately and avoid unfairly penalizing either submission for it. Distinguish an evaluator-discovered benchmark problem from one of the intentionally planted defects.

Most importantly:

**Judge whether each submission demonstrates the behavior of a competent software-engineering team, not merely whether an LLM managed to produce code that turns its own test suite green.**
