# StrataKV — durable transactional storage engineering benchmark

Version: 1.0. Date: 2026-09-28.

## Objective

Build **StrataKV**, a Python library for a small, durable, versioned key/value
store. Clients prepare overlapping transactions against historical snapshots,
declare point and prefix reads, and commit with optimistic conflict detection.
The store must recover after interrupted writes, retain durable retry receipts,
and compact history without breaking readers or recovery.

This is a complete engineering task: reconcile the requirements, implement the
library, test it independently, investigate failures, and produce accurate
engineering evidence. A passing self-authored suite alone is insufficient.
Correctness will also be evaluated through the exact public API and file format
below, including combinations of features.

Use the provided authority rules. No human product clarification is available.
When an actual gap remains, record a narrow decision and its consequences.

## Execution and technology

- Python 3.12 or later; standard library only, including tests.
- Import the public API with `from stratakv import Store, ...` from the repository
  root without installation or changes to `PYTHONPATH`.
- Run the complete native suite with `python3 -m unittest discover -s tests -v`.
- No databases, `sqlite3`, third-party persistence libraries, network services,
  subprocess storage engines, or external infrastructure.
- Domain decisions must not depend on wall time, random IDs, process IDs, hash
  iteration order, or the environment. There are no timestamps or TTLs.
- Real files and `os.fsync` are required. An in-memory imitation of durability
  is not sufficient.
- One open `Store` instance owns a directory at a time. Multiple processes or
  separately opened handles writing the same directory are out of scope.
  Multiple threads using the **same** store are in scope.
- Linux local-filesystem semantics are the benchmark environment. The specified
  crash model is authoritative; physical device failures are out of scope.
- A store-wide lock during individual calls is allowed. Holding a lock for an
  entire transaction lifetime is not: transactions must be able to overlap.
- An operator may use separate PM, development, test-authoring, and documentation
  sessions. Each receives this complete prompt and the current repository.
  Required knowledge must be in files, not remembered from another session.

## 1. Authority and invariants

Authority, highest first:

1. Invariants in this section.
2. Formal rules and exact schemas in sections 2–10.
3. Acceptance obligations in section 11.
4. Informal notes and examples in section 12.

Sections 1–11 are intended to be consistent. Some lower-authority material is
deliberately wrong. Identify and correct all conflicts; do not assume every
informal note is defective. Do not alter the formal contract to accommodate one.

| ID | Invariant |
|---|---|
| INV-1 | A read at revision B sees exactly the latest committed version at or before B. A transaction overlays its own pending writes on that snapshot. |
| INV-2 | A newly accepted transaction has no mutation after its base revision in its declared point reads, prefix reads, or written keys. This includes absence observations, tombstones, and value changes that later reverse. |
| INV-3 | Each accepted write transaction publishes all its mutations atomically at one new revision. No observer sees a partial commit. |
| INV-4 | Every successfully returned first commit has a durable receipt. Recovery retains all sealed commits, even when their callers never received a response. |
| INV-5 | A request ID identifies one canonical intent forever. Matching retries return the original receipt; different intents never reuse it. |
| INV-6 | Compaction preserves every supported snapshot, every durable receipt, and all conflict information needed for any allowed base revision. |
| INV-7 | A domain-rejected call changes no durable or logical store state. An injected crash is a different outcome: section 8 determines whether the interrupted operation committed. |
| INV-8 | Given the same initial files and ordered calls, public results are deterministic and detached from caller-owned mutable objects. Concurrent calls must have a legal serial ordering consistent with completed-before-started calls. |
| INV-9 | Recovery accepts only a validated committed prefix. A torn final journal frame can be discarded; a complete corrupt frame cannot be silently skipped or replaced with an empty store. |

## 2. Values, canonicalization, and errors

### F-01 — Data grammar

- A key matches the complete ASCII regex `[a-z][a-z0-9_./-]{0,63}`.
- A prefix is an ASCII string of length 0–64 containing only
  `abcdefghijklmnopqrstuvwxyz0123456789_./-`. Empty matches all keys.
  A syntactically valid prefix need not match any possible key.
- A request ID matches `[A-Za-z0-9_-]{1,64}` completely.
- A revision, journal sequence, or generation is a nonnegative Python `int`;
  `bool` is not an integer for these fields. No fixed-width arithmetic wraparound
  is permitted.
- A value is `None`, `bool`, a signed 64-bit integer, `str`, a `list` of values,
  or a `dict` with string keys and value values. Floats, including NaN and
  infinity, bytes, tuples, sets, non-string dictionary keys, and cyclic containers
  are invalid. Use the built-in types; custom subclasses are not test inputs.
- Values have at most 16 container levels; the outermost list/dict is level 1.
  Scalars at the deepest level are allowed. Shared references without cycles are
  allowed and serialize by value. A value's canonical encoding is at most
  65,536 bytes.
- These value depth/size/integer bounds apply to each user-stored value, not to
  surrounding API/file envelopes. Counter fields follow their own nonnegative
  integer rule above; wrapping a valid value in an intent does not consume its
  allowed container depth.
- Unless stated otherwise, dictionaries have exactly the listed fields. Unknown
  fields and missing fields are errors. Invalid typed input raises
  `ValidationError`, not a leaked `KeyError`, `TypeError`, or `IndexError`.

Canonical JSON is the UTF-8 encoding of:

```python
json.dumps(value, sort_keys=True, separators=(",", ":"), ensure_ascii=True,
           allow_nan=False)
```

The restricted grammar is checked before encoding. `True` and `1` are different
values and produce different intent fingerprints. Object member order is not
significant; list element order within a value is significant.

### F-02 — Errors and ownership

Export these distinct exceptions, all inheriting `StrataError(Exception)`:

`ValidationError`, `ClosedStore`, `ClosedTransaction`, `SnapshotTooOld`,
`FutureRevision`, `Conflict`, `IdempotencyConflict`, `BusySnapshot`,
`CorruptStore`.

Also export `InjectedCrash(BaseException)`, outside that hierarchy. Error
messages are informative but their wording is not graded.

Every input stored beyond a call is copied by value. Every public returned
dictionary/list is detached from internal state, including nested values,
transaction exports, cells, receipts, and event pages.

For a closed or crash-poisoned store, operations raise `ClosedStore` before
validating their arguments. `close()` is the exception: it is idempotent.
Transaction operations check their owning store first, then whether the
transaction is closed, then validate their arguments. `Transaction.close()` is
idempotent and may also be used after the store closes.

## 3. Required public API

These names, argument forms, return schemas, and error categories are required.
Internal modules and algorithms are your choice.

```python
class Store:
    @classmethod
    def open(cls, path, *, fault=None): ...
    def status(self) -> dict: ...
    def read(self, key, *, at=None) -> dict | None: ...
    def scan(self, prefix="", *, at=None) -> list[dict]: ...
    def begin(self, *, at=None): ...  # returns Transaction
    def commit(self, intent, *, request_id) -> dict: ...
    def lookup(self, request_id) -> dict | None: ...
    def events(self, *, after=None, limit=100) -> dict: ...
    def compact(self, *, floor) -> dict: ...
    def close(self) -> None: ...

class Transaction:
    def get(self, key) -> dict | None: ...
    def scan(self, prefix="") -> list[dict]: ...
    def put(self, key, value) -> None: ...
    def delete(self, key) -> None: ...
    def export(self) -> dict: ...
    def close(self) -> None: ...
```

`path` is a nonempty string or `pathlib.Path` naming the store directory. Create
an absent directory; an existing fresh directory is empty. The evaluator gives
an existing writable parent. `fault` is `None` or a callable taking one string.
Reject invalid types with `ValidationError`. Initialization itself is not a
fault-injection target. Symlinks, permission failures, concurrent `close()`, and
callbacks re-entering the store are outside the tested scope.

Example of normal usage, without prescribing implementation:

```python
store = Store.open(directory)
tx = store.begin()
try:
    tx.get("inventory/a")
    tx.put("inventory/a", {"available": 7})
    intent = tx.export()
finally:
    tx.close()
receipt = store.commit(intent, request_id="adjust_001")
```

## 4. Snapshots and transaction preparation

### F-03 — Revisions and cells

A new store has current revision `R=0`, retention floor `F=0`, and journal
sequence `S=0`. `status()` returns exactly `{"revision": R, "floor": F}`.

Each first accepted commit with one or more final mutations increments R by
exactly one. An accepted commit with no mutations records a receipt but leaves
R unchanged. All first accepted commits increment S by one. Retries, rejected
calls, reads, and compaction increment neither counter.

`read(key, at=B)` returns the latest version at or before B:

```json
{"key":"inventory/a","value":{"available":7},"version":3}
```

It returns `None` if no version exists or that version is a tombstone. A stored
JSON null is a present cell with `"value": null`, distinct from absence.
`scan(prefix, at=B)` returns the same cells for matching present keys, sorted by
key using ASCII lexicographic order.

`at=None` selects R at that call's linearization point. Otherwise at is a
nonnegative integer: B<F raises `SnapshotTooOld`; B>R raises `FutureRevision`.
Check key/prefix and at syntax before range errors. Historical reads at B=F are
supported. Reads and status have no durable side effects.

### F-04 — Preparation and snapshot pins

`begin(at=B)` uses the same revision validation and creates a live transaction
pinned to B. The transaction sees B even after other commits advance R. Each
transaction has its own point-read set, prefix-read set, and final mutation map.

- `get(key)` always adds that key to the point-read set, including reads of
  absence or of an own pending write.
- `scan(prefix)` always adds that prefix to the prefix-read set, including an
  empty result. It observes the complete prefix, not only returned keys.
- `put` replaces that key's pending mutation. `delete` replaces it with a
  tombstone. Neither implicitly adds a read, although written keys are always
  conflict checked at commit.
- Reads combine the snapshot with the final pending mutation for each key.
  A pending put is a cell with `"version": null`; a pending delete is absent.
  Unchanged snapshot cells keep their original revision.
- Repeated writes to one key collapse to one final mutation. Deleting an absent
  key still constitutes a mutation. Writing an equal value still constitutes a
  mutation. Put-then-delete is one tombstone, not an empty write set.
- Invalid calls leave the transaction's reads, prefixes, and mutations unchanged.
- `export()` is pure. It returns the current intent described below, detached
  from the builder; later builder changes cannot alter a previous export.
- `close()` releases the snapshot pin and makes that builder unusable. It does
  not commit or discard an already exported intent. A builder remains live until
  explicitly closed, even if one of its exports was committed.
- Store closure invalidates all builders and releases their pins. Pins are
  process-local and do not survive reopen.

A live builder at B prevents compaction to any floor greater than B. An exported
intent alone does **not** pin history. A client can close its builder and later
find that a first commit of that intent is too old.

### F-05 — Intent schema and identity

An intent has exactly these fields:

```json
{
  "base": 3,
  "reads": ["inventory/a"],
  "prefixes": ["inventory/"],
  "writes": [
    {"key":"inventory/a","op":"put","value":{"available":6}},
    {"key":"inventory/b","op":"delete"}
  ]
}
```

`reads`, `prefixes`, and `writes` are lists. Reads and prefixes contain valid,
unique strings. Writes contain one entry per unique valid key; a put has exactly
`key`, `op`, `value`, and a delete exactly `key`, `op`. An empty list is valid in
all three positions. Duplicates are invalid, even when equal.

A builder exports sorted read keys, sorted prefixes, and writes sorted by key.
`commit` also accepts these three lists in any order and normalizes them before
fingerprinting. Do not drop a narrower prefix covered by a broader one: the
declared set itself participates in identity.

The fingerprint is the lowercase SHA-256 hex digest of the canonical JSON bytes
of the normalized intent. It includes base, reads, prefixes, and final writes.
It excludes the request ID. Callers may submit valid intents directly without
using a builder. Only declared reads are protected; inferring undeclared reads
from application code is outside scope.

## 5. Commit, conflicts, and durable retries

### F-06 — Required commit order

Within one atomic call, perform this observable order:

1. Check store liveness.
2. Validate request ID and the complete intent grammar, including all values and
   duplicates. Normalize and compute the fingerprint.
3. Look up the request ID. If its stored fingerprint matches, return a detached
   copy of its **original** receipt immediately. If it differs, raise
   `IdempotencyConflict`. Neither case rechecks the base range or conflicts.
4. For a new request ID, check F<=base<=R, raising `SnapshotTooOld` or
   `FutureRevision` as appropriate.
5. Apply every conflict rule in F-07. A conflict raises `Conflict`.
6. Prepare one receipt and one journal frame. Persist and publish according to
   section 8; only then return the receipt.

For syntax errors involving several fields, which syntax error is reported is
not significant. Their priority over idempotency/base/conflict checks is.
Once step 3 applies, its priority over steps 4–5 is significant.

No ordinary rejection reserves a request ID, advances R/S, changes a pin, writes
a journal frame, publishes an event, or calls a crash hook.

### F-07 — Conflict rule, including absence and phantoms

Let B be the intent's base. Reject if **any** committed mutation at a revision
greater than B affected:

- a key in `reads`;
- a key in `writes`, including a blind write;
- any key starting with any declared prefix.

This is a change test, not a comparison of current values with old values.
Tombstones count. Same-value puts count. Insert-then-delete counts even if both
the old and current scan are empty. A change outside all declared prefixes and
point/write keys is irrelevant and must not cause rejection.

The rule is deliberately conservative about read-only transactions: they are
also validated. A stale read-only transaction cannot receive a fresh successful
receipt just because it writes nothing. Coarse conflict checks that reject all
transactions whenever R changes are nonconforming.

### F-08 — Receipt and retry

A receipt is exactly:

```json
{"request_id":"adjust_001","revision":4,"keys":["inventory/a","inventory/b"]}
```

Keys are the sorted final mutation keys. For an empty write set, keys is empty
and revision is R at acceptance. For a nonempty write set, revision is its new
revision. Even a read-only receipt is durable and consumes one journal sequence.

Receipts and their fingerprints are retained forever, including after
compaction. A retry returns the original receipt even when its revision or base
is below F, the affected keys were deleted, or the store was reopened. It
allocates no revision or sequence, appends no bytes, and publishes no event.

`lookup(request_id)` validates ID syntax and returns the original receipt or
`None`. It neither prepares nor commits anything. A changed base or dependency
set is a different intent even if the proposed final values are identical.

## 6. Change events and pagination

### F-09 — Events

Every mutation in an accepted write transaction generates exactly one event.
Order keys lexicographically within the commit and assign indices 0 through n-1.

```json
{
  "cursor":{"revision":4,"index":0},
  "key":"inventory/a",
  "deleted":false,
  "value":{"available":6}
}
```

Deletes use `deleted: true` and `value: null`. A put of null uses
`deleted: false`. No event is generated for a retry, rejection, read-only
commit, or compaction. Retained events are exactly those with revision >F.

`events(after=C, limit=N)` returns exactly:

```json
{"events":[...],"next":{"revision":4,"index":0}}
```

- Limit is an integer, excluding bool, from 1 through 256.
- A cursor is exactly `{"revision": r, "index": i}`, where r is a nonnegative
  integer and i is an integer >=-1, neither bool.
- A supplied cursor with r<F raises `SnapshotTooOld`; r>R raises
  `FutureRevision`. A cursor at r=F is valid, although events at F are gone.
- Validate all cursor and limit syntax before checking cursor revision range.
- The cursor is an exclusive lexicographic lower bound on `(revision,index)`.
  It need not identify an existing event. In particular `(r,-1)` precedes all
  events at r; an index beyond that revision's last event is also valid.
- `after=None` starts at the beginning of the currently retained event stream.
- Return at most N events in cursor order. On a nonempty page, next is the last
  returned cursor. On an empty page, next equals the supplied after, including
  None. Each page observes one committed store state atomically.
- Do not advance next to the end of a transaction when a page stops inside it.
  A page may include later commits made since the previous page. Compaction can
  invalidate a previously returned cursor; report that error rather than
  silently resetting the reader.

## 7. Durable representation

### F-10 — Files and generations

Use these exact managed file names, with generation g formatted as at least
eight decimal digits (Python `f"{g:08d}"`):

```text
CURRENT
CURRENT.tmp                       # only during a generation switch
checkpoint.00000000
journal.00000000
```

A new store selects generation 0. Each successful compaction that raises F
selects generation g+1. The selected checkpoint and journal define the store.
Do not use another durable database or hidden copy of the full history. A format
field is the integer 1, not bool or a string.

Fresh initialization must durably write the initial checkpoint, empty journal,
and CURRENT, including file and directory synchronization, before open returns.

All three durable file types use this frame encoding:

| Bytes | Field |
|---:|---|
| 4 | ASCII `SKV1` |
| 8 | unsigned big-endian payload byte length |
| L | canonical JSON payload bytes |
| 32 | raw SHA-256 digest of the payload bytes |
| 4 | ASCII `END!` |

Payload length L must be 1 through 67,108,864 inclusive. Evaluated valid
workloads fit that bound. A frame is exactly L+48 bytes. Reject duplicate JSON
object keys, noncanonical encodings, invalid value types, and schema violations.
`CURRENT` and a checkpoint contain exactly one complete frame and no trailing
bytes. A journal contains zero or more complete frames and possibly one torn
final frame.

`CURRENT` payload:

```json
{"format":1,"generation":0}
```

Checkpoint payload:

```json
{
  "format":1,"generation":0,"sequence":0,"revision":0,"floor":0,
  "versions":[],"receipts":[]
}
```

Each version entry has exactly `key`, `revision`, `deleted`, `value`. Version
revisions are positive. Entries are ordered by `(key,revision)` with no duplicate
pair. Deleted is a bool; deleted entries have value null. Receipt entries are
ordered by request ID and have exactly:

```json
{
  "request_id":"adjust_001",
  "fingerprint":"<64 lowercase hex characters>",
  "receipt":{"request_id":"adjust_001","revision":4,"keys":["inventory/a"]}
}
```

The two request IDs must match. The initial checkpoint has empty lists. A
checkpoint at floor F keeps exactly the versions specified in F-13. Events are
reconstructible from versions above F; do not add an event field to the schema.

Each journal frame has exactly:

```json
{
  "format":1,"sequence":1,
  "fingerprint":"<64 lowercase hex characters>",
  "receipt":{"request_id":"adjust_001","revision":1,"keys":["inventory/a"]},
  "mutations":[{"key":"inventory/a","deleted":false,"value":{"available":7}}]
}
```

Mutations are sorted by key, unique, and match receipt.keys exactly. A tombstone
has deleted true and value null. A read-only commit has no mutations. No other
state is appended for that commit. Thus append size depends on its mutations,
not on a serialized copy of all previous data.

### F-11 — Recovery validation

On open, validate the selected CURRENT, checkpoint, and complete journal prefix
before changing any file. Missing selected files, unsupported formats, bad
checksums, malformed schemas, inconsistent counters, or invalid ordering raise
`CorruptStore`. Leave all bytes unchanged in that failure case. Do not fall back
to an older generation or silently initialize a fresh store.

The required consistency checks include:

- Checkpoint generation matches CURRENT; 0<=floor<=revision; sequence equals
  the number of retained receipt entries; request IDs are unique.
- Nonempty checkpoint receipts have unique revisions covering 1 through R,
  where R is the checkpoint revision. Empty receipts have revisions in [0,R].
  Receipt keys are sorted and unique. Every nonempty receipt has at least one
  key; empty versus nonempty here refers to its keys list.
- Checkpoint version keys and values are valid. The receipt key lists identify
  every revision at which each key was mutated. That key's retained revisions
  must be exactly the greatest such revision <=F, if any, plus all such
  revisions >F. Every retained revision has exactly one version entry. Thus
  every previously mutated key has a retained version, including deleted keys,
  and an anchor cannot be omitted merely because a later version exists.
- Journal sequences are contiguous, starting at checkpoint.sequence+1. A new
  frame's request ID is absent from the replayed receipt registry. A frame with
  mutations has receipt.revision=current R+1; one without has revision=current
  R. All receipt and mutation types/orderings are valid.

Recovery does not reconstruct a missing intent from its fingerprint or rerun
conflict checks for a sealed journal frame. The checksum and these structural
checks establish its replay validity under the defined failure model.

Scan the journal from byte zero, strictly at frame boundaries:

1. EOF exactly at a boundary is clean.
2. An EOF-short header is a torn final frame only if its available magic bytes
   match the corresponding prefix of `SKV1`.
3. Once a complete header exists, an out-of-range length is corruption. If the
   valid declared frame extends beyond EOF, it is a torn final frame.
4. If all declared bytes exist, a bad digest, trailer, payload, or schema is
   corruption, even for the final frame. Invalid magic is corruption.
5. Do not search forward for another magic marker after a bad frame.

After **all** validation succeeds, truncate any torn tail to the last complete
boundary and fsync the journal before accepting new commits. Recover each
complete frame exactly once. A checksum-valid but structurally invalid complete
frame is still corruption. Checkpoints and CURRENT never get torn-tail recovery.

On a successful open, remove only recognized unselected generation files and
CURRENT.tmp, then fsync the directory. Unselected files, even corrupt ones,
are not alternative recovery sources. Unknown unrelated filenames are ignored
and must never be deleted. A missing CURRENT in a directory containing managed
generation files is corruption, not a fresh store.

## 8. Commit durability and fault injection

### F-12 — Journal seal is the commit boundary

Every instruction to call fault in this document is conditional on a configured
callback. With fault=None, omit the callback without changing operation order.

For a first accepted commit, use this order under the operation's exclusion:

1. Prepare and validate all state changes and the receipt in memory.
2. Call `fault("commit.before_append")` if fault is configured.
3. Append the frame except its final four-byte `END!`; flush userspace buffers
   and fsync the journal.
4. Call `fault("commit.after_body")`.
5. Append `END!`; flush and fsync the journal. The commit is now durable.
6. Call `fault("commit.after_seal")`.
7. Publish the complete in-memory state and receipt.
8. Call `fault("commit.after_publish")`, then return the receipt.

Each hook is called exactly once, in order, on the applicable successful path.
The fault callback normally returns None or raises `InjectedCrash`. On that
exception, let it escape and mark the original store unusable. Do not undo,
repair, truncate, compact, or clean its files while unwinding. Release file
handles/locks without new durable changes. All later calls other than close
raise `ClosedStore`. The evaluator reopens with a new Store instance.

| Injected crash point | Recovered result |
|---|---|
| commit.before_append | Exact previous logical state; no receipt for this attempt |
| commit.after_body | Exact previous logical state; incomplete frame removed on recovery |
| commit.after_seal | Entire commit and original receipt present |
| commit.after_publish | Entire commit and original receipt present |

This also applies to first read-only commits. Retrying an interrupted unsealed
attempt may commit; retrying a sealed attempt returns its existing receipt.
Any exception after a seal cannot be described as a guaranteed rollback.

The evaluator can also construct an interrupted append by placing a byte prefix
of one additional valid frame after a known sealed prefix. Test every region of
the additional frame. This models an unfinished, unacknowledged attempt; it does
not authorize removal of an earlier acknowledged commit. Under this benchmark's
simulated crash model, a complete valid sealed frame present in the reopened
files is durable; an EOF-incomplete last frame is not. Bit corruption and
inconsistent complete records are separately tested as corruption, not
interpreted as ordinary crashes. Arbitrary OS I/O errors between specified
points are not scored; describe how your code treats them without claiming a
guarantee you have not implemented.

## 9. History compaction and generation switching

### F-13 — Logical compaction

`compact(floor=N)` checks, in order: liveness, integer syntax, N<F
(`SnapshotTooOld`), N>R (`FutureRevision`), and any live builder with base<N
(`BusySnapshot`). Rejection changes no files, counters, receipts, or pins.

If N==F, it is a pure no-op returning status and calling no hooks. Otherwise:

- Keep, for each key, its greatest version <=N if one exists: its **anchor**.
  Keep that anchor even if it is a tombstone.
- Keep every version >N.
- Discard every other version. Do not retain a hidden full history.
- Retain every receipt and fingerprint, including read-only receipts.
- The event stream now consists only of revisions >N.
- R and S do not change. F becomes N atomically when the new generation is
  selected. Return the new status.

For example, revisions [1,3,7] of one key compacted at 5 become [3,7], not [7].
The retained version at 3 determines that key's value at snapshots 5 and 6.
The rule is applied independently to every key, not globally to the newest
transaction. Receipt storage is allowed to grow permanently; obsolete value
payloads are not.

### F-14 — Physical compaction and crash points

While excluding concurrent store operations for the switch:

1. Write checkpoint.g+1 with the compacted representation and current R/S;
   flush and fsync it. Call `fault("compact.after_checkpoint")`.
2. Create the empty journal.g+1; flush and fsync it. Fsync the directory so both
   new filenames are durable. Call `fault("compact.after_journal")`.
3. Write a complete framed CURRENT.tmp selecting g+1; flush and fsync it.
   Call `fault("compact.before_switch")`.
4. Atomically replace CURRENT with CURRENT.tmp and fsync the directory. Publish
   the new in-memory floor/generation. Call `fault("compact.after_switch")`.
5. Delete recognized unselected generation files and leftover CURRENT.tmp,
   fsync the directory, and call `fault("compact.after_cleanup")`.
6. Return status.

An injected crash has the same poison-and-propagate behavior as a commit crash.
Before the switch, reopen must select the complete **old** generation and old
floor. After the switch, it must select the complete **new** generation and new
floor. Both contain the same current values, receipts, R, and S. Recovery cleans
unselected artifacts only after validating the selected generation. A retry
may reuse an unselected g+1 filename safely; it may never overwrite a selected
file in place.

After successful compaction or reopen cleanup, only the selected checkpoint and
journal remain among generation files. The new journal is empty at the end of
compaction. Normal commits append to that selected journal. Never delete the
old selected generation before durable selection of the new one.

## 10. Threading and operation boundaries

### F-15 — Concurrency

All Store calls and all pin registration/release must be thread-safe. Transaction
builders are confined to one thread at a time; the evaluator does not call two
methods simultaneously on the same builder.

The commit's conflict check, request-ID decision, revision allocation, journal
append/seal, and publication belong to a single atomic ordering. A lock around
only the dictionary update is insufficient. Reads, scans, status, and event
pages never expose an unsealed or partially published transaction.

Two simultaneous commits of the same ID and same intent both return the same
receipt, with one physical frame. The same ID with different intents has one
winner and one `IdempotencyConflict`. Conflicting new IDs have one successful
commit and one `Conflict`. Disjoint nonconflicting intents both succeed even
when based on the same old revision.

Compaction racing a new pin must either observe and honor the pin or complete
first and make a too-old begin fail. It cannot invalidate a successfully
registered live snapshot. Tests must use barriers/events to establish relevant
interleavings; guessed sleep durations are not synchronization evidence.

## 11. Acceptance obligations

Each obligation is graded from executable behavior, including combined cases.

| ID | Obligation |
|---|---|
| AC-01 | Strict value/input validation, canonical identities, absent versus null, and full copy isolation. |
| AC-02 | Historical point/range reads, transaction overlays, export isolation, and explicit pin lifetime. |
| AC-03 | Write/write conflict, write skew prevention through declared reads, and success for truly disjoint writes. |
| AC-04 | Empty-range and absent-key guards; insert/delete and value ABA still conflict. |
| AC-05 | Correct error precedence; rejected calls leave logical state and managed bytes unchanged. |
| AC-06 | Immutable durable retry receipts across later writes, reopen, and compaction; read-only commits included. |
| AC-07 | Exact multi-key event order, pagination inside a revision, and cursor behavior at/under the floor. |
| AC-08 | All four commit crash points, retry after each outcome, and poisoning of old handles. |
| AC-09 | Torn tails versus complete corruption, contiguous replay, and no writes during failed recovery. |
| AC-10 | Compaction anchors, tombstone retention, snapshot pins, real history removal, and event retention. |
| AC-11 | All five compaction crash points followed by reopen, retry, new commits, and another compaction. |
| AC-12 | Real thread races for commits, request IDs, readers, and compaction versus pins. |
| AC-13 | Long deterministic traces against an independently structured reference model, with reproducible minimized failures. |
| AC-14 | Mutation testing that demonstrates the native suite rejects plausible but runnable semantic defects. |

## 12. Lower-authority material to reconcile

### Informal product notes

- **PN-1:** “Snapshot isolation should be enough. If a transaction only writes
  keys nobody else wrote, let it commit even when values it read have changed.”
- **PN-2:** “An empty prefix scan did not return records, so it adds no conflict
  dependencies. That will reduce unnecessary aborts.”
- **PN-3:** “On retry, return a receipt with the latest revision so the client
  knows it is current; the old result is no longer useful.”
- **PN-4:** “Compaction should forget request IDs below its floor and discard
  tombstones. Those represent operations or keys we no longer retain.”
- **PN-5:** “If commit raises instead of returning a receipt, its changes must
  be absent after restart. Retry can always allocate a new revision.”
- **PN-6:** “A coarse lock inside each operation is acceptable for this first
  version, provided transactions can be prepared concurrently.”
- **PN-7:** “An equal-value put is still a recorded write and must be visible
  in the revision history and event stream.”

### Illustrative examples

**EX-1:** Start with a=1 and b=1. Two builders start at the same revision, both
read a and b, then one writes a=0 and the other writes b=0. Whichever commits
second must fail, even though their write sets are disjoint.

**EX-2:** Revision 7 changes a, b, and c. A page of size 2 returns a and b.
“Set next to (7,2), since revision 7 has already been processed; the following
page can begin with revision 8.”

**EX-3:** A builder scans `jobs/` and sees nothing. Another transaction inserts
`jobs/a`; a later transaction deletes it. The first builder's scan would again
be empty, but its newly attempted commit still conflicts.

## 13. Engineering workflow and required artifacts

Work through PM → development → test authoring/execution → documentation/review.
Use additional repair rounds when warranted. These are roles/phases; spawning
multiple agents is not a requirement. Each phase can be a fresh session.

Route implementation defects to development, invalid test expectations to test
authoring, and actual unresolved authoritative ambiguities to PM. Preserve
evidence of failures and corrections. Do not manufacture a failure or a
requirements ambiguity to demonstrate a loop-back.

### PM outputs, before implementation

- `SPEC.md`: concise implementable specification, exact lifecycle/ordering
  decisions, and preservation of all formal requirements.
- `REQUIREMENTS_CORRECTIONS.md`: each conflicting lower-authority clause,
  controlling rule, corrected behavior, and separately any actual ambiguity.
- `DESIGN_REVIEW.md`: explain write skew, an absent-read phantom, the durable
  commit boundary, retry identity, the anchor rule, and generation-switch
  failure cases using concrete traces. Identify at least three interactions
  where two individually correct subsystems could combine incorrectly.

### Development outputs

- Importable `stratakv` implementation and `README.md` with API/run examples.
- `DEV_NOTES.md`: actual data structures, lock ownership, linearization points,
  byte-level persistence/recovery protocol, copy/validation strategy,
  complexity by operation, and limitations. Explain what survives each crash.
- Keep normal journal appends proportional to the current intent's mutations.
  Describe how key histories and conflict guards are found. Brute-force searches
  can be correct, but do not claim an index or complexity you did not implement.

### Test-authoring outputs

- Standard-library tests and `TEST_PLAN.md` mapping all obligations to concrete
  adversarial cases, expected results, and competing wrong implementations.
- An in-memory reference model with a materially different representation or
  algorithm. It must not import production validation, conflict, history, or
  replay helpers. An adapter may call the public production API. The model need
  not model physical bytes, but must cover logical state, intents, receipts,
  events, and compaction outcomes.
- Run at least 24 fixed seeds, each with at least 250 generated actions, against
  the model and the implementation. Actions must include overlapping builders,
  conflicting/disjoint intents, retries, invalid inputs, reopen, and compaction.
  Include injected-crash recovery in the overall test campaign. Compare errors
  as well as success values and resulting state.
- Provide `python3 tools/replay_trace.py TRACE.json` for saved differential
  traces, and `python3 tools/shrink_trace.py TRACE.json OUTPUT.json` to reduce a
  reproducible failing action sequence while retaining its failure. If a trace
  has no failure, report that fact and exit successfully without inventing one.
  Use a demonstrated runnable mutant to exercise shrinking when production has
  no discovered differential failure. Specify the trace format in TEST_PLAN.
- Exercise every named crash point, EOF truncation at multiple positions, and
  complete-frame corruption separately. Include at least one fresh subprocess
  reopening a store written by another process to rule out hidden memory state.
- Exercise deterministic real-thread schedules with barriers/events. Include
  write skew, disjoint commits, same-ID retries, conflicting same IDs, and a pin
  racing compaction. No sleeping as a substitute for an asserted interleaving.
- Implement and run at least six plausible, independently applied semantic
  mutations in isolated copies, with a restoration/reproduction mechanism.
  Syntax/import failures do not count as kills. `MUTATION_RESULTS.md` must name
  each mutation, the exact failing assertion/test, the command, and outcome.
  Surviving mutants must be disclosed.

### Test execution and final documentation

- `TEST_RESULTS.md`: commands, environment, seeds, counts, actual results, and
  each encountered failure's expected/observed behavior, classification,
  routing, and correction. Never report a planned run as executed.
- `TRACEABILITY.md`: map INV-1–INV-9, F-01–F-15, and AC-01–AC-14 to real source
  locations and executable tests. Identify gaps explicitly.
- `FINAL_REVIEW.md`: outcome, actual loop-backs, unresolved issues, corrected
  lower-authority defects, five highest remaining risks, and which guarantees
  have independent executable evidence. Distinguish implemented, tested, and
  merely proposed behavior.
- Preserve machine-readable failing traces and useful run logs under `evidence/`.
  There is no prose-length quota and no reward for inflated test counts.

## 14. Public grading contract

The evaluator uses the same frozen independent checks for every model, including
unpublished combinations of the rules above. No hidden feature or unstated input
standard is required. Native test counts do not determine production points.

| Dimension | Artifact points |
|---|---:|
| MVCC snapshots, overlays, and copy/value semantics | 9 |
| Declared-read, prefix, and write conflict correctness | 12 |
| Validation, error ordering, and durable receipts | 9 |
| Journal protocol, commit crashes, and recovery | 14 |
| Compaction, pins, and generation recovery | 11 |
| Event stream and cursor semantics | 5 |
| Thread safety and atomic observable behavior | 10 |
| Native verification: independent model, meaningful mutation kills, reproducible reduction | 15 |
| Architecture explanation, evidence accuracy, and maintainability | 10 |
| Requirements reconciliation and authority reasoning | 5 |
| **Total** | **100** |

The first seven rows total 70 independent functional points. A separate
**coding-agent capability** score is computed from the same evidence:

```text
20 × (requirements points / 5)
+ 35 × (functional points / 70)
+ 30 × (native verification points / 15)
+ 15 × (architecture/evidence points / 10)
```

Both scores use A=90–100, B=80–<90, C=70–<80, D=60–<70, F=<60.
Report one decimal place. These scores belong to this harder benchmark and are
not numerically interchangeable with scores from another task.

Correctness gates apply to both scores after weighted scoring:

- Maximum **69** if a supported trace loses a sealed/returned commit, publishes
  half a transaction, accepts complete corrupt selected storage, or accepts two
  different intents under one request ID.
- Maximum **79** if a supported trace accepts a commit forbidden by F-07,
  incorrectly replays a receipt, or invalidates a successfully pinned snapshot.
- Apply the lowest applicable cap once; report the raw score and the gate.

Infrastructure failures outside the contract are reported separately, not
silently counted as model defects. An evaluator must provide a minimized
reproducer for every applied correctness gate.

Final delivery is the working repository, its test/evidence artifacts, and a
short handoff listing known limitations honestly.
