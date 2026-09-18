# SQLite Durability Implementation Plan

## 1. Create Local State Database

``` text
.kaggle/
    state.db
    cache.db
    apply.lock
```

Initialize SQLite with:

``` sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA synchronous = FULL;
```

Use one database per project/workspace.

## 2. Start With Four Tables

``` text
resources
    resource_key
    resource_type
    acknowledged_json
    acknowledged_digest
    materialization

runs
    run_id
    schedule_digest
    task_key
    task_version
    harness
    model

ownership
    resource_key

metadata
    key
    value
```

Do NOT persist:

``` text
Plan
OperationDAG
READY/RUNNING operation state
R
RunStatus
```

Those are recomputed or observed.

## 3. State Backend Boundary

Design for multiple state backends, but implement only SQLite initially.

``` python
class StateBackend(Protocol):
    def get_resource(self, key): ...
    def commit_resource(self, key, canonical, materialization): ...

    def get_runs(self, schedule_digest): ...
    def add_run(self, run): ...


class SQLiteStateBackend(StateBackend):
    ...
```

Current architecture:

``` text
Reconciler
    |
    v
StateBackend
    |
    v
SQLiteStateBackend
    |
    v
.kaggle/state.db
```

Nothing outside the backend implementation should issue SQL or depend on
SQLite-specific behavior.

Future implementations MAY include:

``` text
StateBackend
    |
    +-- SQLiteStateBackend      # current
    +-- PostgresStateBackend    # future
    +-- GCSStateBackend         # future
    +-- S3StateBackend          # future
```

Do not implement future backends, distributed locking, or remote-state
coordination until required.

Expose semantic atomic operations such as:

``` text
commit_resource(
    key,
    acknowledged,
    materialization
)
```

rather than generic transaction or SQL primitives.

The contract is:

``` text
commit_resource(...)
    =
atomic {
    L[key] = acknowledged
    Materialization[key] = materialization
}
```

### State vs Cache

Keep durable state and disposable cache logically separate.

``` text
StateBackend                 Cache
     |                         |
     v                         v
state.db                  cache.db/files
```

Invariant:

``` text
State = required for correctness
Cache = disposable optimization
```

SQLite may physically store both initially, but the reconciler MUST NOT
depend on cached data for correctness.

## 4. Implement `init`

``` text
load Config
    |
    v
observe Remote
    |
    v
canonicalize
    |
    v
transaction {
    write L
    write Materialization
    discover/store relevant RunIds
}
```

`init` performs no remote mutations.

## 5. Implement `plan` as Read-Only

``` text
C = evaluate Config
L = read StateBackend
R = observe Remote
M = read Materialization

resolve dependencies
classify C/L/R
build OperationDAG
```

`plan` MUST NOT write reconciliation state.

## 6. Implement Resource Apply

For each operation:

``` text
execute remote mutation
    |
    v
observe remote
    |
    v
verify expected canonical state
    |
    v
StateBackend.commit_resource(...)
```

For SQLite this is implemented as a transaction that atomically updates:

``` text
L
Materialization
```

Only verified remote state enters durable state.

Never write:

``` text
L = desired state
```

Write:

``` text
L = verified observed state
```

## 7. Make Each Resource Commit Independent

Do not wrap the entire apply in one transaction.

Use:

``` text
operation 1
-> verify
-> COMMIT

operation 2
-> verify
-> COMMIT

operation 3
-> crash
```

After restart:

``` text
operations 1 + 2 remain durable
operation 3 is recomputed
```

## 8. Recovery by Replanning

Do not resume an old DAG.

On every invocation:

``` text
load StateBackend
evaluate C
observe R
recompute Plan
```

If a remote mutation succeeded before a crash:

``` text
C == R
C != L

-> ALREADY_RECONCILED
```

Then:

``` text
verify
-> commit L + Materialization
```

This is the primary crash-recovery mechanism.

## 9. Durable Run Submission

Run creation has a dangerous window:

``` text
schedule request
-> server creates Run
-> CRASH
-> RunId not stored
```

Use a deterministic request identity:

``` text
ScheduleDigest
+
Attempt
    ->
RunRequestId
```

Example:

``` text
RunRequestId =
SHA256(ScheduleDigest || Attempt)
```

Send it as the server idempotency key if supported.

``` text
schedule(S, idempotency_key=RunRequestId)
```

Retrying the same request MUST return the same RunId.

## 10. Persist Run Immediately

As soon as scheduling returns:

``` text
schedule
-> RunId
-> StateBackend.add_run(...)
-> durable commit
```

Do not wait for Run completion.

Run status remains remote observed state:

``` text
RunId -> get_status()
```

## 11. Retry Attempts

A network/process retry is NOT a new attempt:

``` text
same RunRequestId
-> same RunId
```

A retry after:

``` text
FAILED
ERRORED
```

IS a new attempt:

``` text
attempt = attempt + 1

new RunRequestId
-> new RunId
```

This distinction prevents accidental duplicate Runs.

## 12. Add Parallelism

Only after serial durability works.

Workers may perform remote work concurrently:

``` text
Worker 1 ---- remote API
Worker 2 ---- remote API
Worker 3 ---- remote API
```

SQLite commits remain short transactions:

``` text
BEGIN IMMEDIATE

write L
write Materialization

COMMIT
```

Do not hold SQLite transactions open while performing network calls.

Bad:

``` text
BEGIN
-> HTTP request
-> upload
-> observe
-> COMMIT
```

Good:

``` text
HTTP request
-> observe
-> verify

BEGIN
-> persist facts
-> COMMIT
```

## 13. File Hash Cache

Use a disposable cache for Dataset file hashes:

``` text
(path, size, mtime)
    ->
digest
```

Lookup before hashing.

``` text
cache hit
    -> reuse digest

cache miss
    -> hash
    -> update cache
```

Treat this as optimization state.

Corrupt/lost cache data must affect performance only, never correctness.

## 14. Downloads

Download into temporary paths:

``` text
.runs/<run-id>/
    logs.tmp/
    output.tmp/
```

When complete:

``` text
logs.tmp   -> logs
output.tmp -> output
```

Use atomic rename.

Partial downloads never appear complete.

## 15. Schema Migrations

Store:

``` text
metadata.schema_version
```

Every release performs:

``` text
current_version
    |
    v
migration
    |
    v
new_version
```

Migrations execute transactionally.

Never make application code depend on manually deleting `state.db`.

## 16. Test With Crash Injection

Add explicit crash points:

``` text
before remote mutation
after remote mutation
after observe
after verify
before SQLite commit
after SQLite commit

after Run creation
before RunId commit
after RunId commit
```

For every crash point:

``` text
crash
-> restart
-> apply
-> converge
```

Property:

``` text
apply_with_arbitrary_crashes(...)
    eventually
~
uninterrupted_apply(...)
```

## Implementation Order

``` text
1. StateBackend interface

2. SQLite schema + migrations

3. SQLiteStateBackend

4. init

5. read-only plan

6. serial resource apply

7. verify + atomic L/Materialization commits

8. crash recovery through replanning

9. RunHistory persistence

10. idempotent Run submission

11. crash-injection tests

12. parallel DAG executor

13. file digest cache

14. durable artifact downloads
```

## Core Rule

``` text
StateBackend stores durable facts.

SQLite is the first implementation.

Remote APIs perform effects.

Plan is always recomputed.

Cache is never required for correctness.
```

If the process disappears at any instruction, the next invocation should
need only:

``` text
durable StateBackend
+
Config
+
current Remote
```

to determine what to do next.

------------------------------------------------------------------------

## Cross-Cutting Architecture Clarifications

These rules apply across the reconciliation design.

### Ownership Is Separate From Reconciliation

``` text
Ownership[x] =
    UNMANAGED
    | MANAGED
```

`L` is not extended with `UNKNOWN`.

For managed resources:

``` text
L[x] =
    ABSENT
    | PRESENT<CanonicalResource>
```

An unmanaged resource with an existing remote object requires explicit
import/adoption. A managed resource missing from config is `ORPHANED`;
config omission does not destroy or forget it.

### Global Resource Key

``` text
ResourceKey =
    (resource_type, owner, slug)
```

Use this key consistently in state, materialization, dependencies,
operations, logs, and diagnostics.

### Resource vs Execution Planning

Keep resource mutation planning separate from execution scheduling.

``` text
ResourcePlan:
    CREATE
    CREATE_VERSION
    UPDATE
    ACKNOWLEDGE
    NOOP
    DESTROY

ExecutionPlan:
    SCHEDULE
    RETRY
    NOOP
```

Runs are execution attempts, not reconciled resources.

### Acknowledgement Semantics

After:

``` text
execute
-> observe
-> verify
-> commit
```

`L` means the last remote state successfully observed, verified, and
acknowledged by this project.

It does not claim that the remote remained unchanged between
verification and the local commit. A later mutation is detected as
remote drift on the next observation.

### Cache Correctness

File fingerprint caching is a heuristic optimization.

``` text
(path, size, mtime)
    -> cached digest
```

A fingerprint match MAY reuse the digest in optimized mode. Strict mode
hashes file contents. Cache loss or corruption must never change
reconciliation semantics.

### Physical State / Cache Separation

Preferred local layout:

``` text
.kaggle/
    state.db       # correctness-critical durable facts
    cache.db       # disposable optimization data
    apply.lock
```

Deleting `cache.db` must always be safe.

------------------------------------------------------------------------

## Cross-Cutting Architecture Clarifications

### Ownership Is Separate From Reconciliation

``` text
Ownership[x] =
    UNMANAGED
    | MANAGED
```

`L` is not extended with `UNKNOWN`.

For managed resources:

``` text
L[x] =
    ABSENT
    | PRESENT<CanonicalResource>
```

An unmanaged resource with an existing remote object requires explicit
import/adoption. A managed resource missing from config is `ORPHANED`;
config omission does not destroy or forget it.

### Global Resource Key

``` text
ResourceKey =
    (resource_type, owner, slug)
```

Use this key consistently in state, materialization, dependencies,
operations, logs, and diagnostics.

### Resource vs Execution Planning

``` text
ResourcePlan:
    CREATE
    CREATE_VERSION
    UPDATE
    ACKNOWLEDGE
    NOOP
    DESTROY

ExecutionPlan:
    SCHEDULE
    RETRY
    NOOP
```

Runs are execution attempts, not reconciled resources.

### Acknowledgement Semantics

After:

``` text
execute
-> observe
-> verify
-> commit
```

`L` means the last remote state successfully observed, verified, and
acknowledged by this project. It does not claim that the remote remained
unchanged between verification and local commit.

### Cache Correctness

``` text
(path, size, mtime)
    -> cached digest
```

is a heuristic optimization, not a content identity. Strict mode hashes
file contents. Cache loss or corruption must never change reconciliation
semantics.

### Physical State / Cache Separation

``` text
.kaggle/
    state.db
    cache.db
    apply.lock
```

`state.db` contains correctness-critical facts. `cache.db` is
disposable. Deleting `cache.db` must always be safe.
