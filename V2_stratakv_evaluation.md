# StrataKV evaluator and run guide

Version 1.0 — 2026-09-28. Operator/evaluator material.

## What to give each model

Give each candidate **only `stratakv_benchmark.md`**, in a fresh writable
repository. This guide belongs outside the candidate's accessible workspace.
Keeping two files in this directory does not enforce isolation: copy the prompt
into a separate candidate workspace and restrict that workspace's access.
Do not expose the other model's implementation, findings, or scores.

This directory supplies a complete candidate task and an evaluation blueprint.
It does **not** yet contain an executable independent oracle/probe suite. Build,
validate, and freeze that suite before collecting scored candidate runs. Do not
ask either candidate to define the independent evaluator for its own submission.

## Why this task is harder than ChronoGate

ChronoGate tested a sophisticated but single-process, in-memory state machine.
Cloud and FN_M both cleared the difficult domain behavior. Their remaining
observed failures were narrow input issues, so adding more near-identical
reservation examples would give little new information.

StrataKV adds distinct skills and interacting correctness obligations:

| New demand | What it tests |
|---|---|
| Snapshot preparation with overlapping transactions | Separation of read time from commit time; dependency tracking |
| Negative reads, prefix phantoms, and ABA histories | Reasoning about changes that cannot be inferred from current values |
| Journal framing, sealing, and restart | Byte-level implementation and a correct durability boundary |
| Immutable retry receipts | Exactly-once request identity despite uncertain responses |
| Compaction with anchors and pins | Retaining the minimum sufficient history without breaking old readers |
| Atomic generation switches | Crash recovery across several files and cleanup ordering |
| Real thread schedules | Lock scope, linearization, and races beyond sequential unit tests |
| Independent model, mutation tests, and trace reduction | Ability to falsify an implementation and diagnose a minimal cause |

The task deliberately uses integer revisions and an exact grammar. It should
not be decided by another debatable timestamp-format interpretation. Storage
schemas, errors, public APIs, and crash points are fixed to make independent
testing possible. Documentation and raw native-test counts cannot compensate
for a broken core guarantee.

This is an intended increase in difficulty, not an empirically calibrated claim
about the models. A one-point difference from a single run is weak evidence.

## Run protocol

1. Freeze the prompt, evaluator, environment, scoring rows, seeds, and resource
   budget before running either model. Record SHA-256 hashes and source versions.
2. Use isolated empty repositories and the same Python version, CPU/RAM limits,
   filesystem, tools, phase instructions, and model sampling settings where
   comparable. Record actual model identifiers, quantization, and serving setup.
3. Use four fresh phases: PM; development; test authoring/execution; documentation
   and review. Each receives the full prompt and repository. No score depends on
   remembering a previous chat. Preserve the phase handoff artifacts.
4. Give both models the same token and tool budgets per phase. Record wall time
   separately: inference throughput is an operational metric, not correctness.
   Set budgets before the run; do not give the first struggling candidate a
   private extension. Any common budget revision requires a new comparison run.
5. Allow native tests and self-discovered fixes throughout the authorized loop.
   Save the first delivered candidate artifact before external feedback.
6. Run the frozen evaluator on isolated copies. Report first-submission scores.
   Candidate code and tests remain unchanged in the archived submission.
7. If comparing repair ability, provide the same policy of minimal failing
   traces and the same repair budget to each. Freeze another artifact and report
   post-feedback scores separately. Never replace the initial score silently.
8. Prefer at least three independent runs per model if resources allow. Report
   all runs, median, range, and the count of runs hitting a correctness gate.
   With one run, label the result as a single-run comparison.

Harness crashes, unavailable tools, and interrupted sessions are recorded
separately. Use the same disclosed restart policy for both models. A candidate
that voluntarily submits an incomplete implementation has an incomplete
artifact; that is different from an infrastructure interruption.

Do not run unknown candidate code against important directories. The evaluator
creates dedicated temporary store directories for every test and never uses a
candidate-supplied arbitrary deletion path.

## Scoring mechanics

The candidate prompt contains the complete public weights and correctness gates.
The independent functional rows below total 70 points. For each row, freeze a
small set of concrete cases before either run. Its points are:

```text
row weight × passing frozen cases / total frozen cases
```

An individual case passes only when its return/error, logical state, and any
required byte-level assertions all pass. Avoid twenty copies of one easy case
outweighing one distinct hard obligation: keep the functional row weights fixed.
Use the same adapter, cases, timeouts, and ordering for both candidates.

Tests must call the specified API and inspect only specified managed files.
Do not depend on private method names or require a particular indexing strategy.
Thread tests use barriers/events and bounded joins; a timeout alone is first
investigated as a possible test/harness issue before assigning a deadlock defect.

### A. Snapshot, value, and overlay semantics — 9 points

Each row is worth 1 point.

| Probe | Obligation |
|---|---|
| A01 | Read and scan historical revisions, including zero and exactly F. |
| A02 | Distinguish missing, deleted, and present-null cells. |
| A03 | Overlay put/delete/put, preserve untouched cell versions, and use null for pending versions. |
| A04 | Collapse repeated writes, but preserve absent-key deletes and equal-value puts as mutations. |
| A05 | Own-write get and empty scan still register their declared dependencies. |
| A06 | Deep-copy nested inputs, returned cells/receipts/pages, and exported intents in both directions. |
| A07 | Enforce the exact value grammar, depth, size, key, and prefix rules. |
| A08 | Preserve builder state on invalid operations; enforce close/store-poison lifecycle precedence. |
| A09 | Export is pure and canonical; direct valid intents behave like builder exports. |

### B. Optimistic conflict rules — 12 points

Each row is worth 1 point. Test both commit orders where meaningful.

| Probe | Obligation |
|---|---|
| B01 | Same-key blind writes from the same base conflict. |
| B02 | Disjoint writes from the same old base both succeed. |
| B03 | Declared reads prevent write skew across different written keys. |
| B04 | Point observation of absence conflicts with a later insert. |
| B05 | Point observation conflicts with insert-then-delete. |
| B06 | A→B→A and same-value writes still conflict. |
| B07 | An initially empty prefix detects a later matching insert. |
| B08 | Prefix predicates detect insertion followed by deletion. |
| B09 | Updates and deletes of existing matching keys conflict. |
| B10 | Changes outside guarded prefixes/points/writes do not conflict. |
| B11 | Stale read-only intents conflict and do not consume IDs. |
| B12 | These rules remain correct for bases at F after compaction and reopen. |

### C. Validation order and retry receipts — 9 points

Each row is worth 1 point.

| Probe | Obligation |
|---|---|
| C01 | Malformed request/intent shapes, duplicates, unknown fields, and bool counters produce ValidationError. |
| C02 | Full syntax validation precedes retry/base/conflict resolution. |
| C03 | Existing matching ID precedes base-too-old and current conflicts; return original receipt. |
| C04 | Existing differing ID precedes base-too-old/conflict; raise IdempotencyConflict. |
| C05 | Permuted read/prefix/write lists and reordered object members preserve identity; True versus 1 does not. |
| C06 | Different base, dependency set, or nested list order changes identity. |
| C07 | Rejection changes neither logical state nor managed bytes and leaves ID reusable. |
| C08 | Empty write-set receipts survive reopen and compaction without allocating a new revision. |
| C09 | Retried receipts and lookup remain immutable across later writes/deletes; retry appends zero bytes. |

### D. Journal and recovery — 14 points

Each row is worth 2 points. Within D02 and D03, distribute cases evenly across
all named hook positions and relevant record kinds.

| Probe | Obligation |
|---|---|
| D01 | Exact framing, canonical bytes, checksums, sequence/revision progression, and one delta record per first commit. |
| D02 | All four commit crash positions for a multi-key write; verify reopened state, receipt, old-handle poison, and retry. |
| D03 | All four commit crash positions for an empty write set; separate R from S. |
| D04 | Interrupted append prefixes across header/payload/digest/trailer; truncate only the torn tail, then append/reopen successfully. |
| D05 | Complete bad magic/checksum/trailer/JSON/schema at the last and an interior frame fails without changing bytes. |
| D06 | Well-framed but inconsistent sequences, revisions, IDs, mutation ordering, and checkpoint counters fail closed. |
| D07 | Fresh-process recovery, pure reads/close, and selected-file validation before any cleanup or truncation. |

D04 starts with an intact known-good committed prefix and appends a prefix of a
new valid frame to simulate an unfinished attempt. Do not remove an acknowledged
commit and then pretend the implementation violated durability by failing to
recover bytes you deleted. Complete-frame corruption is a separate failure model.

### E. Compaction and snapshot pins — 11 points

Each row is worth 1 point.

| Probe | Obligation |
|---|---|
| E01 | Per-key anchor at/below the floor, plus every later version; exact floor reads. |
| E02 | Tombstone anchors prevent resurrecting old values, including after restart. |
| E03 | Multiple simultaneous pins block only floor increases that would invalidate a live base. |
| E04 | Closing builders releases pins; exported intents do not pin; old receipt retries still succeed. |
| E05 | Invalid/decreasing/future/no-op floors have the specified effects and no hooks. |
| E06 | after_checkpoint crash preserves old selected state and floor. |
| E07 | after_journal and before_switch crashes preserve old state despite complete unselected files. |
| E08 | after_switch and after_cleanup crashes recover the new floor and full logical state. |
| E09 | Continue with another commit and compaction after each crash; sequence/generation identities remain consistent. |
| E10 | Physical checkpoint pruning is real; journal starts empty; unselected managed files are removed only after validation. |
| E11 | All old fingerprints/receipts survive; missing anchors and inconsistent selected snapshots are corruption. |

### F. Events and cursors — 5 points

Each row is worth 1 point.

| Probe | Obligation |
|---|---|
| F01 | One sorted event per final mutation; null put differs from deletion. |
| F02 | Pages split a multi-key transaction without skips or duplicates; correct next on empty pages. |
| F03 | Exclusive bounds, index=-1, non-existing indices, and exact validation precedence. |
| F04 | At-floor cursors work, under-floor cursors fail, None starts at the retained beginning. |
| F05 | Reopen/retry/crash introduce no duplicate events; reads, empty commits, and compaction introduce none. |

### G. Real concurrency — 10 points

Each row is worth 2 points. Validate the allowed outcome set, not a preferred
winning thread or scheduling order.

| Probe | Obligation |
|---|---|
| G01 | Same-base disjoint commits both succeed; competing same-key writes have exactly one winner. |
| G02 | Write-skew and prefix-phantom schedules allow no forbidden second commit. |
| G03 | Concurrent same ID/same intent yields one frame and equal receipts; differing intents have one IdempotencyConflict. |
| G04 | Readers/event consumers observe complete sealed states, never partial multi-key writes or duplicate revisions. |
| G05 | Pin/compaction race has a legal ordering; active builders do not monopolize a store-wide lock. |

## Compound schedules that should carry real weight

Include these in the applicable rows, not as unweighted afterthoughts. Vary keys,
payloads, history depth, and commit order in withheld cases.

1. **Uncertain write response → compaction → old retry.** Commit at after_seal,
   crash, reopen, mutate the same keys, compact past the original base, retry the
   original ID/intent, and compare original receipt, bytes, events, and R/S.
2. **Empty predicate → ABA → floor boundary.** Pin B, read empty prefix, insert
   then delete under it, compact to B (allowed), and commit the old intent.
   Verify conflict, no receipt allocation, and no spurious conflict outside the
   prefix.
3. **Read-only journal sequence → crash → later write.** Separate S from R across
   multiple empty write sets, crash on another empty commit, reopen, retry, and
   verify the next nonempty commit's revision and physical sequence.
4. **Tombstone anchor → unselected-generation crash → reopen.** Several keys
   have different anchor revisions; one is deleted. Interrupt each compaction
   phase, reopen, check historical reads, then compact again.
5. **Page inside a revision → retry → compaction.** Stop after the first event
   of a multi-key commit, replay its request ID, obtain the remaining page, then
   move the floor and verify the stale cursor fails explicitly.
6. **One declared range, disjoint writers, one intruder.** Prepare several
   builders at the same B; permit unrelated writes and abort only the builder
   whose declared predicate or written key was affected.
7. **Corrupt complete frame plus orphan files.** Create an invalid selected
   generation and valid old/unselected artifacts. Verify CorruptStore with all
   file bytes unchanged; no fallback, deletion, or truncation is allowed.
8. **Retry race after recovery.** Two threads resubmit a sealed but unreturned
   request after a fresh-process reopen. Require identical old receipts, one
   durable historical record, and no extra sequence or event.

## Native verification — 15 points

### Independent differential model — 5 points

- 2: model uses a substantively independent state representation/algorithm and
  does not reuse production semantic helpers; explain what makes it independent.
- 2: actual seeded differential runs meet the prompt's minimum and cover success,
  rejection, state, receipts, history, events, and compaction.
- 1: crash/reopen and real-thread tests validate outcomes with executable
  evidence rather than only checking that calls do not throw.

### Meaningful mutation sensitivity — 6 points

Award 1 point per each of six distinct semantic mutant families actually killed
by a relevant assertion, up to 6. Require a repeatable patch/copy/runner and an
unmutated passing baseline. Import, syntax, discovery, infrastructure, and
arbitrary timeout failures do not count as mutation kills.

Eligible families include lost predicate reads, value-comparison ABA detection,
incorrect retry ordering, missing receipt persistence, premature publication,
overaggressive compaction, incorrect generation selection, and cursor skipping.
The candidate need not use those names. Multiple minor variants of one defect
family count once. Independently rerun the claimed mutations; a prose table
alone earns no execution credit.

### Reproducibility and reduction — 4 points

- 1: replay tool faithfully reproduces saved action sequences and failures.
- 2: shrinker demonstrably reduces a failing differential trace while retaining
  the same violated obligation; it does not merely turn the trace into invalid
  setup. Preserve failure classification, not necessarily identical messages.
- 1: commands, seeds, run logs, and fixture generation make the results repeatable.

Do not give extra credit for more test methods or more lines of code.

## Architecture and evidence — 10 points

- 4: correct explanation matching actual lock scope, linearization points,
  persistence operations, fsync boundaries, and recovery states. Confirm in code.
- 3: maintainable separation of concerns and honest complexity/resource analysis;
  normal commits do not serialize the entire store. Efficient code earns credit
  through its design and evidence, not a machine-dependent speed threshold.
- 3: traceability matches real requirements/functions/tests; failure history,
  mutation results, and final limitations match reproducible observations.

Do not reward a beautiful description of an algorithm that is absent from the
implementation. Do not penalize an honest limitation twice merely because it was
also documented. Missing evidence receives no evidence credit; it is not proof
that a behavior is implemented incorrectly.

## Requirements reasoning — 5 points

The six conflicting lower-authority sources are **PN-1, PN-2, PN-3, PN-4, PN-5,
and EX-2**. PN-4 contains both receipt-retention and tombstone-retention mistakes.
PN-6, PN-7, EX-1, and EX-3 are consistent with the formal contract.

- 3: allocate equally across the six conflicting sources, requiring the correct
  controlling rule and correction. PN-4 needs both corrections for full credit.
- 1: preserve correct lower-authority statements and avoid inventing unsupported
  product changes or treating taste as ambiguity.
- 1: PM design review correctly analyzes at least three cross-feature hazards.

Correct EX-2 continuation is `(7,1)` after a and b, so c at `(7,2)` remains
eligible. The caller's missing response does not determine whether a journal
frame was sealed. Retry must return its original receipt without checking that
its old base is still available. Conflict detection is by mutation history,
including tombstones and equal-value changes, not current-value equality.

## Gates, scores, and interpretation

Calculate raw artifact and capability scores using the public formulas. Apply
the lowest applicable public correctness cap to each. Show raw and capped scores
along with the minimal reproducer and violated rule. A defect can fail multiple
independent cases; report one root cause with its affected obligations, not a
long list presented as unrelated discoveries.

If a probe turns out to rely on an unstated requirement or invalid setup, remove
or repair it **for all candidates**, rerun affected cases, and version the
evaluation. Do not fix the evaluator only for the favored model. Distinguish a
prompt defect, evaluator defect, candidate defect, and harness failure.

Use this result template, filling actual observations:

| Measure | Cloud | FN_M |
|---|---:|---:|
| Independent functional points /70 | TBD | TBD |
| Native verification /15 | TBD | TBD |
| Architecture/evidence /10 | TBD | TBD |
| Requirements /5 | TBD | TBD |
| Raw artifact /100 | TBD | TBD |
| Raw coding-agent capability /100 | TBD | TBD |
| Correctness gate, if any | TBD | TBD |
| Final artifact score and letter | TBD | TBD |
| Final capability score and letter | TBD | TBD |
| First-delivery unresolved defects | TBD | TBD |
| Post-feedback score, if separately run | TBD | TBD |
| Median/range across repeated runs | TBD | TBD |
| Tokens, tool calls, elapsed time | TBD | TBD |

Prior ChronoGate scores are background only. Do not translate 98 on ChronoGate
into a prior score on StrataKV or assume the old rank will hold. Focus the
decision on reproducible core failures, verification ability, repair quality,
and consistency across runs.

## Evaluator validation before deployment

Before using the numeric results, check the independent harness against a known
conforming implementation and intentional semantic mutants. This reference is
evaluator-owned and must not be visible to candidates. The grader itself needs
counterexamples demonstrating it catches the errors it claims to measure.

Check particularly that the oracle distinguishes R from S, does not depend on
private object identity, applies syntax-before-retry precedence, preserves
receipt identity below F, and treats current-generation selection as the sole
recovery authority. Verify failure cases leave files unchanged by hashing every
managed file before/after in a dedicated temporary directory.

For a pilot, report feasibility, completion rate, runtime, and which rubric rows
separate solutions. If both models still saturate the benchmark, design and
freeze a new version before another comparison; do not add asymmetric surprise
requirements mid-run.
