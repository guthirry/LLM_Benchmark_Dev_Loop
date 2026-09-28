# LLM Software Engineering Benchmark — ChronoGate

## Objective

Build a small but rigorously specified Python library named **ChronoGate**: a deterministic reservation engine for capacity-constrained resources.

This benchmark is intentionally designed to exercise the full engineering loop rather than only code generation. The PM must repair requirement defects, the developer must implement non-trivial temporal and capacity semantics, the test author must design adversarial tests rather than merely mirror examples, and the testing round must correctly classify failures as implementation, test, or requirements defects.

The final repository is the benchmark artifact. It must be reviewable and scoreable even if the evaluator does not run the code.

---

## Operating Model

Work through the normal orchestration loop:

**PM → Developer → Test Author → Testing → route failures back to the correct role**

Routing rules:

- Implementation does not satisfy the corrected specification → **Developer**.
- Test expectation is inconsistent with the corrected specification → **Test Author**.
- Correct behavior cannot be determined from the authoritative requirements → **PM**.
- Do not change production code merely to satisfy a defective test.
- Do not change a test merely because production code fails it.
- Do not silently invent requirements when the existing authority hierarchy can resolve them.

No human clarification is available. Resolve the task from the material in this document.

---

## Technology Constraints

- Python **3.12+**.
- Production code must use the Python standard library only.
- Tests may use `pytest` if available, otherwise they must remain straightforward to port to `unittest`.
- No database, network service, web UI, background process, or wall-clock dependency is required.
- Do not call `datetime.now()`, `time.time()`, or equivalent from domain logic. Time is supplied explicitly.
- Favor correctness, determinism, and clarity over framework complexity.

---

# 1. Requirements Authority

The material below is not all equally authoritative. When statements conflict, use this order:

1. **System Invariants**
2. **Formal Behavioral Rules**
3. **Acceptance Criteria**
4. **Raw Product Notes**
5. **Illustrative Examples**

Higher-ranked material overrides lower-ranked material.

The **System Invariants and Formal Behavioral Rules are internally consistent**.

There are **exactly three planted defects** in the lower-authority material:

- one contradictory product statement,
- one incorrect idempotency statement,
- one misleading example.

Before implementation, the PM must identify all three and produce a corrected specification. The PM must not “fix” authoritative rules to preserve a lower-authority defect.

---

# 2. System Invariants

These invariants may never be violated.

### INV-1 — No overbooking

For a resource with capacity `C`, the number of capacity-consuming reservations active at any instant must never exceed `C`.

### INV-2 — Half-open reservation windows

All reservation windows are **half-open** intervals:

`[start, end)`

A reservation ending at exactly `T` does not overlap one starting at exactly `T`.

### INV-3 — Deterministic behavior

Given the same initial state and the same ordered sequence of calls, all returned values and snapshots must be identical.

### INV-4 — Monotonic supplied time

For all timed public operations, `now` may remain equal to the previous successful timed operation, but may not move backward.

A call with regressed time fails with no mutation.

### INV-5 — Atomic public mutations

Every mutating public call is atomic.

If it fails for any reason, **all changes caused by that call must be rolled back**, including:

- status changes from maintenance,
- waitlist promotions,
- idempotency registry changes,
- changes to the last accepted time.

### INV-6 — Capacity consumers

Only records in `HELD` or `CONFIRMED` state consume capacity.

`WAITLISTED`, `CANCELLED`, and `EXPIRED` records do not.

---

# 3. Formal Behavioral Rules

## 3.1 Engine construction

The engine is initialized with a mapping of resource IDs to positive integer capacities.

Example:

```python
ReservationEngine({"room-a": 2, "room-b": 1})
```

Resource IDs must be non-empty strings. Capacities must be positive integers; booleans, zero, negative values, and non-integers are invalid.

Resource IDs are otherwise opaque strings. Do not add undocumented case-folding, trimming, or other normalization.

---

## 3.2 Required states

Every request is in exactly one state:

- `HELD`
- `CONFIRMED`
- `WAITLISTED`
- `CANCELLED`
- `EXPIRED`

A `CONFIRMED` record remains `CONFIRMED` after its reservation window has ended. Its ended time interval simply ceases to consume capacity. `EXPIRED` means an unconfirmed hold or stale waitlist request lost its opportunity; it does not mean a completed confirmed reservation.

---

## 3.3 Timestamp rules

All externally supplied timestamps are RFC 3339 / ISO-8601 timestamps containing an explicit UTC offset or `Z`.

Requirements:

- Naive timestamps are invalid.
- Equivalent instants expressed with different offsets are semantically equal.
- Internally, comparisons must be by instant, not by timestamp text.
- Fractional seconds from zero through six digits are supported and are significant in comparisons.
- Output timestamps use the canonical UTC form `YYYY-MM-DDTHH:MM:SS.ffffffZ`, always with six fractional digits.
- Leap seconds and precision finer than microseconds do not need to be supported.

Example of equivalent instants:

- `2026-01-15T15:00:00Z`
- `2026-01-15T10:00:00-05:00`

---

## 3.4 Reservation creation

A request is created with:

- `request_id`
- `user_id`
- `resource_id`
- `start`
- `end`
- `priority`
- supplied `now`

Priorities, from highest to lowest:

1. `GOLD`
2. `SILVER`
3. `BRONZE`

Validation requirements:

- `request_id` and `user_id` must be non-empty strings.
- `resource_id` must exist.
- `priority` must be one of the three allowed values.
- `end > start`.
- `start > now` at the time a new request is first accepted.

After validation and maintenance, the engine evaluates whether the new reservation can be added without violating capacity at **any instant** in `[start, end)`.

If it fits for the entire interval:

- state = `HELD`
- `requested_at = now`
- `hold_expires_at = min(now + 5 minutes, start)`

If it does not fit:

- state = `WAITLISTED`
- `requested_at = now`
- `hold_expires_at = null`

A capacity check must consider actual concurrent occupancy over time. Merely counting how many existing records overlap the candidate interval is not sufficient when capacity is greater than one.

---

## 3.5 Maintenance phase

Every timed public mutating call follows this conceptual order inside its atomic transaction:

1. Parse and validate `now`.
2. Reject time regression.
3. Perform maintenance at `now`.
4. Execute the requested command.
5. Perform any command-triggered waitlist promotion required by the rules below.
6. Commit only if the complete call succeeds.

Maintenance does the following in this order:

### A. Expire holds

Any `HELD` record with:

`hold_expires_at <= now`

becomes `EXPIRED`.

### B. Expire stale waitlist records

Any `WAITLISTED` record with:

`start <= now`

becomes `EXPIRED`.

### C. Promote eligible waitlist records

For each resource independently, sort remaining `WAITLISTED` records by:

1. higher priority first,
2. earlier `requested_at` first,
3. lexicographically smaller `request_id` first.

Walk the sorted list once.

For each candidate:

- if it fits for its entire interval, promote it to `HELD`,
- set `hold_expires_at = min(now + 5 minutes, start)`,
- immediately account for its capacity before considering the next candidate,
- if it does not fit, leave it `WAITLISTED` and continue evaluating later candidates.

A high-ranked request that does not fit does **not** block a lower-ranked request that does fit.

---

## 3.6 Confirm

`confirm(request_id, now)` behaves as follows after maintenance:

- `HELD` → `CONFIRMED`
- `CONFIRMED` → return the existing record unchanged; this is idempotent
- `WAITLISTED` → invalid state
- `CANCELLED` → invalid state
- `EXPIRED` → invalid state
- unknown request → unknown-request error

Because maintenance runs before the command, a hold whose `hold_expires_at == now` is already expired and cannot be confirmed.

Confirming does not change capacity consumption because both `HELD` and `CONFIRMED` consume capacity.

---

## 3.7 Cancel

`cancel(request_id, now)` behaves as follows after maintenance:

- `HELD` → `CANCELLED`
- `CONFIRMED` → `CANCELLED`
- `WAITLISTED` → `CANCELLED`
- `CANCELLED` → return unchanged; this is idempotent
- `EXPIRED` → invalid state
- unknown request → unknown-request error

If cancellation frees capacity, run waitlist promotion for that resource immediately before committing the call.

---

## 3.8 Idempotent reservation creation

`request_id` is the idempotency key for reservation creation and is never reusable.

The canonical idempotency payload consists of:

- `user_id`
- `resource_id`
- canonical instant for `start`
- canonical instant for `end`
- `priority`

A repeated `reserve(...)` with the same `request_id` and the same **canonical payload**:

- does not create another record,
- returns the current record for that request after normal maintenance,
- may therefore return a later state than the original call.

The supplied retry `now` is not part of the idempotency payload.

A repeated `request_id` with any different canonical payload raises `IdempotencyConflict` and the entire call rolls back.

A cancelled or expired request ID still remains permanently registered.

---

## 3.9 Batch operation

Implement:

```python
apply_batch(commands, now)
```

Supported command types inside the batch:

- `reserve`
- `confirm`
- `cancel`

Rules:

- All commands in the batch use the same supplied `now`.
- Maintenance runs once at the start of the batch.
- Commands are then executed in list order.
- Command-specific effects occur immediately; for example, a cancellation may promote a waitlisted request before the next command is executed.
- A command later in the same batch observes state changes caused by earlier commands.
- If any command fails, the **entire batch** rolls back to the exact pre-call state, including the maintenance phase and last accepted time.
- Return one result per command, in command order, if the batch succeeds.

A batch itself does not weaken the idempotency rules.

---

## 3.10 Snapshot

Implement:

```python
snapshot()
```

`snapshot()`:

- takes no `now`,
- performs no maintenance,
- has no side effects,
- returns only JSON-serializable data,
- uses canonical UTC timestamps,
- is deterministically ordered.

At minimum, the snapshot must expose enough data to independently inspect:

- resource capacities,
- every request and its state,
- reservation interval,
- priority,
- requested time,
- hold expiry where applicable,
- last accepted engine time.

Choose and document one deterministic ordering scheme and use it consistently.

---

# 4. Acceptance Criteria

### AC-1
Two intervals `[10:00, 11:00)` and `[11:00, 12:00)` do not overlap.

### AC-2
Capacity must be enforced at every instant, including when several existing reservations overlap different sub-ranges of a candidate interval.

### AC-3
Waitlist promotion uses priority, then request time, then request ID, and skips candidates that currently cannot fit.

### AC-4
Equivalent RFC-3339 timestamps with different offsets behave as the same instant.

### AC-5
A duplicate reservation request with the same canonical payload is idempotent even if the record has since changed state.

### AC-6
A duplicate request ID with a different canonical payload fails without mutation.

### AC-7
A hold at its expiry instant follows the maintenance-before-command rule.

### AC-8
A cancellation that releases capacity promotes eligible waitlist entries before the call returns.

### AC-9
A failed batch restores the complete pre-batch state.

### AC-10
A failed single mutating call also restores any maintenance changes that would otherwise have happened during that call.

### AC-11
A time-regressing call fails without changing the engine state or last accepted time.

### AC-12
The snapshot is deterministic and side-effect free.

---

# 5. Raw Product Notes — Intentionally Lower Authority

These are informal notes copied from product discussions. They are not automatically correct.

### PN-1
“Customers have five full minutes to confirm, so confirming at exactly `hold_expires_at` should still succeed.”

### PN-2
“For retry safety, if the same request ID is submitted again, compare the timestamp strings exactly. For example, `15:00Z` and `10:00-05:00` should count as different payloads.”

### PN-3
“When a confirmed booking is cancelled, the first waitlisted booking should usually get the newly opened capacity, unless it still does not fit.”

### PN-4
“Retries must never create duplicate reservations.”

### PN-5
“The engine should be deterministic enough that two independent runs can be diffed.”

---

# 6. Illustrative Examples — Intentionally Lower Authority

Examples are explanatory only. They must not override formal rules.

## EX-1 — Adjacency

Resource capacity = 1.

- A reserves `[10:00, 11:00)`.
- B reserves `[11:00, 12:00)`.

Both can be `HELD` because the intervals are adjacent, not overlapping.

## EX-2 — Capacity is temporal, not an overlap-count shortcut

Resource capacity = 2.

Existing capacity consumers:

- A: `[10:00, 11:00)`
- B: `[11:00, 12:00)`

Candidate C:

- C: `[10:30, 11:30)`

C fits. At no instant are A, B, and C all active together.

An implementation that simply counts “two existing records overlap C” and rejects C is wrong.

## EX-3 — Waitlist skipping

Capacity = 1.

Suppose two waitlisted requests exist in promotion order:

- GOLD request G does not fit because another confirmed reservation still overlaps G's interval.
- BRONZE request B targets a different interval that is currently free.

Promotion leaves G waitlisted and may promote B.

## EX-4 — Cancellation exactly at a waitlisted request's start

Capacity = 1.

- Existing confirmed request A occupies `[09:30, 10:30)`.
- Request B for `[10:00, 11:00)` is waitlisted because it overlaps A.
- At exactly `10:00`, A is cancelled.

“B is immediately promoted to `HELD` at 10:00 because capacity is now free.”

---

# 7. Required Public API

The implementation must provide a practical equivalent of this interface:

```python
class ReservationEngine:
    def __init__(self, resources: dict[str, int]): ...

    def reserve(
        self,
        *,
        request_id: str,
        user_id: str,
        resource_id: str,
        start: str,
        end: str,
        priority: str,
        now: str,
    ) -> dict: ...

    def confirm(self, *, request_id: str, now: str) -> dict: ...

    def cancel(self, *, request_id: str, now: str) -> dict: ...

    def apply_batch(self, commands: list[dict], *, now: str) -> list[dict]: ...

    def snapshot(self) -> dict: ...
```

Reasonable custom exception types are expected. At minimum, distinguish:

- validation failure,
- time regression,
- idempotency conflict,
- unknown request,
- invalid state transition.

The exact module layout is your design decision.

---

# 8. PM Deliverables

Before development begins, create:

## `SPEC.md`

A corrected, implementation-ready specification derived from the authority hierarchy.

It must:

- preserve all authoritative behavior,
- explicitly resolve all three planted lower-authority defects,
- define observable behavior precisely enough for independent test authors,
- avoid introducing new product semantics not required by the benchmark.

## `REQUIREMENTS_CORRECTIONS.md`

For each planted defect, record:

- source clause/example,
- why it is defective,
- higher-authority rule that resolves it,
- corrected interpretation.

If the PM believes there is an additional true requirements ambiguity, record it separately with a rationale. Do not manufacture ambiguity merely to satisfy the workflow.

---

# 9. Developer Deliverables

Create the implementation plus:

## `DEV_NOTES.md`

Document:

- architecture and data model,
- capacity-checking strategy,
- atomic rollback strategy,
- timestamp canonicalization strategy,
- waitlist promotion strategy,
- any assumptions,
- complexity considerations,
- known limitations, if any.

The implementation must not depend on the tests to define behavior.

---

# 10. Test Author Deliverables

Create a serious automated test suite and:

## `TEST_PLAN.md`

The test plan must explain the test design, not merely list filenames.

The suite should include, at minimum, meaningful coverage of:

- half-open interval boundaries,
- capacity > 1 with non-trivial overlap geometry,
- timestamp offset equivalence,
- fractional-second behavior,
- hold expiry boundary,
- start-time boundary,
- waitlist priority and tie-breaks,
- skipping an unfit higher-ranked waitlist request,
- promotion after cancellation,
- promotion after hold expiry,
- idempotent reserve retry after a state change,
- idempotency conflict rollback,
- single-call maintenance rollback on failure,
- batch ordering effects,
- complete batch rollback,
- time regression,
- repeated same `now`,
- invalid state transitions,
- snapshot determinism and purity.

At least several tests should be adversarial constructions intended to catch plausible-but-wrong implementations, not just happy-path examples.

Do **not** blindly turn every raw product note or illustrative example into a test expectation. Test against the corrected specification.

---

# 11. Testing-Round Deliverables

Run or otherwise evaluate the suite against the implementation and create:

## `TEST_RESULTS.md`

For every failure encountered during the loop, record:

- failing test,
- observed behavior,
- expected behavior,
- classification:
  - `IMPLEMENTATION_DEFECT`,
  - `TEST_DEFECT`, or
  - `REQUIREMENTS_DEFECT`,
- routing decision,
- resolution.

If a test was changed, explain why the test was wrong.

If production code was changed, explain which requirement it violated.

If the PM specification was changed, explain why existing authoritative text was insufficient.

Do not erase evidence of earlier failed iterations from this report.

---

# 12. Traceability and Final Review

Create:

## `TRACEABILITY.md`

Map each system invariant and acceptance criterion to:

- implementation location(s),
- test(s) that verify it.

Create:

## `FINAL_REVIEW.md`

Include:

- final outcome,
- number and type of loop-backs,
- unresolved issues,
- whether all three planted requirements defects were identified,
- whether any defective test was caught rather than accommodated,
- the five most failure-prone behaviors in this implementation and why.

---

# 13. Quality Bar

A high-quality solution should demonstrate all of the following:

- It recognizes requirement authority instead of treating all prose as equally valid.
- It finds all planted requirement defects before or during implementation.
- It implements interval capacity correctly rather than using a naive overlap count.
- It handles boundary instants consistently.
- It treats equivalent timestamp representations semantically.
- It makes atomic rollback real, including maintenance side effects.
- It handles waitlist ordering and skipping correctly.
- It writes tests capable of falsifying plausible incorrect implementations.
- It distinguishes a bad test from bad production code.
- It leaves a coherent evidence trail from requirements through implementation and testing.

The goal is not maximal code volume. The goal is disciplined software engineering under subtly adversarial requirements.

---

# 14. Submission

The final submission is the complete repository/directory produced by the workflow.

At minimum it must contain:

```text
SPEC.md
REQUIREMENTS_CORRECTIONS.md
DEV_NOTES.md
TEST_PLAN.md
TEST_RESULTS.md
TRACEABILITY.md
FINAL_REVIEW.md
<production source>
<automated tests>
```

Do not submit only a prose solution. The evaluator must be able to inspect the specification, implementation, tests, and engineering decision trail together.
