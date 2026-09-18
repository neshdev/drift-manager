# State Backend Recommendations

## Recommendation

Design for multiple state backends, but implement only SQLite initially.

``` text
Reconciler
    |
    v
StateBackend
    |
    v
SQLiteStateBackend
```

The reconciler MUST NOT depend directly on SQLite.

## Interface

``` python
class StateBackend(Protocol):
    def get_resource(self, key): ...
    def commit_resource(self, key, canonical, materialization): ...

    def get_runs(self, schedule_digest): ...
    def add_run(self, run): ...
```

Expose semantic atomic operations rather than SQL or transaction
primitives.

``` text
commit_resource(...)
    =
atomic {
    L[key] = acknowledged
    Materialization[key] = materialization
}
```

## Current Implementation

``` text
StateBackend
    |
    +-- SQLiteStateBackend
            |
            v
        .kaggle/state.db
```

SQLite is the only backend that should be implemented initially.

Do not prematurely implement distributed locking, remote state
coordination, Postgres, GCS, or S3.

## Future Backends

The abstraction should permit future implementations:

``` text
StateBackend
    |
    +-- SQLiteStateBackend      # current
    +-- PostgresStateBackend    # future
    +-- GCSStateBackend         # future
    +-- S3StateBackend          # future
```

Backend-specific storage and locking behavior MUST remain behind
`StateBackend`.

## State vs Cache

Durable state and disposable cache are logically separate.

``` text
StateBackend                 Cache
     |                         |
     v                         v
durable state             optimization
```

``` text
State:
    L
    Materialization
    RunHistory
    schema metadata

Cache:
    file digests
    temporary metadata
    other recomputable data
```

Invariant:

``` text
State = required for correctness
Cache = disposable optimization
```

SQLite MAY physically store both initially, but correctness MUST NOT
depend on cached data.

## Core Rule

``` text
Reconciler knows:
    resource state semantics
    atomic state operations

Reconciler does NOT know:
    SQLite
    SQL
    Postgres
    S3
    GCS
    backend locking implementation
```

Implement the abstraction now.

Implement only SQLite now.

Add other backends only when required.

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
