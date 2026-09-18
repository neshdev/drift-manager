# Architecture Decisions

## Identity

``` text
ResourceKey =
    (resource_type, owner, slug)
```

## Ownership

``` text
Ownership =
    UNMANAGED
    | MANAGED
```

Ownership is separate from reconciliation.

For managed resources:

``` text
C, L, R =
    ABSENT
    | PRESENT<T>
```

``` text
unmanaged + remote absent  -> CREATE
unmanaged + remote present -> ADOPTION_REQUIRED
managed + config missing   -> ORPHANED
```

Explicit intent:

``` text
import       -> adopt existing remote resource
state remove -> stop managing; remote unchanged
destroy      -> delete remote resource
```

## Planning

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

## Acknowledgement

``` text
execute
-> observe
-> verify
-> atomic commit(L, Materialization)
```

`L` is the last remote state successfully observed, verified, and
acknowledged.

## Local Storage

``` text
.kaggle/
    state.db
    cache.db
    apply.lock
```

`state.db` is correctness-critical. `cache.db` is disposable.

## Cache

`(path, size, mtime)` is only a heuristic cache fingerprint. Strict mode
hashes contents.

## Execution

``` text
Planner
    -> OperationDAG
        -> SerialExecutor
        -> ConcurrentExecutor
```

Serial execution is the reference semantics. Optimized execution must be
observationally equivalent.

## Durability

Persist facts, not workflow progress.

``` text
durable:
    Ownership
    L
    Materialization
    RunHistory

ephemeral:
    Plan
    OperationDAG
    R
    RunStatus
    worker progress
```

Recovery always reloads durable state, observes remote state, replans,
and continues.
