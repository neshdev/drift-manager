# Parallelism & Performance Recommendations

## Principle

Keep parallelism out of the resource model.

``` text
Dataset / Task / Benchmark / Run
    = semantics

Planner
    = determines dependencies

Executor
    = determines parallelism
```

The same plan should produce the same semantic result regardless of
parallelism.

## 1. Execute the DAG in Parallel

The dependency graph determines what can run concurrently.

``` text
        Dataset
        /     \
       v       v
    Task 1   Task 2
       \       /
        v     v
       Benchmark
```

Execute:

``` text
Dataset
   |
   v
Task 1 || Task 2
   |
   v
Benchmark
```

General rule:

``` text
run everything whose dependencies have completed successfully
```

Use bounded concurrency:

``` text
--parallelism 16
```

## 2. Parallelize Remote Reads

Independent remote reads SHOULD execute concurrently during:

``` text
init
plan
apply
```

Examples:

``` text
fetch datasets
fetch tasks
fetch benchmarks
fetch run statuses
```

Observation MUST NOT mutate acknowledged state.

## 3. Parallelize Dataset Hashing

Dataset files can be hashed independently.

``` text
hash(file1) ||
hash(file2) ||
hash(file3)
```

Sort results deterministically before constructing `CanonicalDataset`.

Parallel execution MUST NOT affect the resulting canonical state or
digest.

## 4. Cache File Hashes

Avoid repeatedly hashing unchanged files.

``` text
(path, size, mtime)
    ->
SHA256
```

``` text
unchanged file -> reuse digest
changed file   -> recompute digest
```

The cache is disposable and MUST NOT be required for correctness.

## 5. Use Resource Digests for Equality

``` text
CanonicalResource
    ->
CanonicalJSON
    ->
SHA256
```

Normal reconciliation can compare:

``` text
digest(C)
digest(L)
digest(R)
```

Compute structural differences only when detailed diff output is
required.

## 6. Submit Runs in Parallel

Once Task materializations are resolved:

``` text
schedule(task1, agent1) ||
schedule(task2, agent1) ||
schedule(task3, agent2)
```

Persist every returned `RunId` immediately.

`apply` MUST NOT wait for Runs to complete.

## 7. Persist Progress Incrementally

For every completed resource operation:

``` text
execute
-> observe
-> verify
-> atomic commit {
       L
       Materialization
   }
```

Do not wait for the complete apply before persisting successful work.

## 8. Keep Database Transactions Short

Remote work MAY execute concurrently.

Do not hold a SQLite transaction during remote operations.

``` text
remote mutation
-> observe
-> verify

BEGIN
-> persist durable facts
COMMIT
```

SQLite transactions should contain local state changes only.

## 9. Continue Independent Work After Failure

Given:

``` text
        d1          d2
        |           |
        v           v
       t1          t2
```

If `t1` fails:

``` text
descendants(t1) -> blocked
t2              -> continue
```

A failure SHOULD block only operations that depend on it.

## 10. Plans Remain Ephemeral

Do not persist worker progress or the Operation DAG for recovery.

``` text
Plan
OperationDAG
READY
RUNNING
worker state
```

are ephemeral.

After a crash:

``` text
load durable state
observe remote
recompute plan
continue
```

Durability comes from persisted facts, not workflow resumption.

## Recommended Implementation Order

``` text
1. serial reconciliation correctness
2. durable StateBackend
3. crash recovery through replanning
4. DAG executor
5. bounded parallel remote reads
6. bounded parallel resource mutations
7. parallel Run submission
8. file digest cache
9. parallel file hashing
10. profile before adding further optimization
```

## Core Invariant

``` text
apply(config, parallelism=1)
~
apply(config, parallelism=N)
```

where `~` means the same final semantic resource state and required
execution specifications.

Parallelism changes execution speed, not reconciliation meaning.

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
