# Serial and Optimized Execution

## Recommendation

Implement serial execution first as the reference implementation.

Add optimized execution later without changing planning or
reconciliation semantics.

``` text
                 Planner
                    |
                    v
              OperationDAG
                    |
          +---------+---------+
          |                   |
          v                   v
   SerialExecutor      ConcurrentExecutor
   reference/simple    optimized
```

## Common Interface

Both executors consume the same plan.

``` python
class Executor(Protocol):
    def execute(self, plan: OperationDAG) -> ApplyResult:
        ...
```

The planner MUST NOT depend on which executor is selected.

## Serial Executor

The serial executor is the correctness reference.

``` text
topological_sort(DAG)

for op in order:
    execute
    observe
    verify
    commit L + Materialization
```

Keep it deliberately simple:

``` text
no worker pools
no concurrent mutations
no concurrent observation
no batching
no execution-specific caching
```

Durability and crash recovery SHOULD be proven against this
implementation first.

## Concurrent Executor

The optimized executor preserves the same semantics while exploiting
independent work.

``` text
while work remains:
    find ready operations
    execute independent operations concurrently
    observe concurrently
    verify
    commit each successful result
```

Concurrency MUST remain bounded.

Dependent operations MUST wait until their dependencies have been
verified and committed.

## Equivalence

The primary optimization property is:

``` text
SerialExecutor(P)
~
ConcurrentExecutor(P)
```

where `~` means equivalent observable semantics:

``` text
same final acknowledged state L
same Materialization
same required ScheduleSpecs
equivalent RunHistory
```

Server-generated identifiers MAY differ where identity is intentionally
server-generated.

Parallelism changes execution strategy, not reconciliation meaning.

## Testing

Use the serial executor as the oracle for optimized implementations.

For generated:

``` text
resource DAG
C
L
R
Materialization
RunHistory
```

execute both implementations and assert:

``` text
normalize(serial_result)
==
normalize(concurrent_result)
```

Also inject failures and crashes into both implementations.

After recovery:

``` text
recover(serial)
~
recover(concurrent)
```

## Optimization Path

Add optimizations incrementally.

``` text
Planner
    |
    v
SerialExecutor
    |
    v
durability + crash recovery
    |
    v
correctness/property tests
    |
    v
ConcurrentExecutor
    |
    +-> parallel remote reads
    +-> parallel resource mutations
    +-> parallel Run submission
    +-> file digest cache
    +-> parallel file hashing
    +-> batching where supported
```

Each optimization MUST preserve equivalence with the serial reference.

## Core Rule

``` text
SerialExecutor = reference semantics

ConcurrentExecutor = optimization
```

An optimization is valid iff it is observationally equivalent to serial
execution.

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
