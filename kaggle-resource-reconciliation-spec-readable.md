# Declarative Kaggle Resource Reconciliation Spec

## 1. Resource Model

Resource types:

``` text
ResourceType = Dataset | Task | Benchmark
```

A logical resource is identified by:

``` text
Key = (ResourceType, owner, slug)
```

Version semantics:

  Resource    Versioned
  ----------- -----------
  Dataset     Yes
  Task        Yes
  Benchmark   No

A concrete versioned resource is:

``` text
VersionedResource = (Key, version)
```

`version` is allocated by the server.

------------------------------------------------------------------------

## 2. Dependency Model

Configuration defines a directed acyclic graph:

``` text
G = (Nodes, Edges)
```

Legal dependency edges:

``` text
Task      -> Dataset
Benchmark -> Task
```

Dependencies are version-pinned when materialized remotely:

``` text
Task/vT -> Dataset/vD

Benchmark -> {
    Task1/v1,
    Task2/v2,
    ...
}
```

Configuration references logical identities, not server versions.

Example:

``` text
C:

ds1   = Dataset("user1", "ds1")
task1 = Task("user1", "task1", dataset=ds1)
bench = Benchmark("user1", "bench1", tasks={task1})
```

Remote materialization:

``` text
R:

Dataset user1/ds1/v4

Task user1/task1/v7
    -> user1/ds1/v4

Benchmark user1/bench1
    -> user1/task1/v7
```

------------------------------------------------------------------------

## 3. Three States

For every managed resource `x`:

``` text
C[x] = desired configuration
L[x] = last acknowledged remote state
R[x] = current observed remote state
```

Represent the reconciliation input as:

``` text
State[x] = (C[x], L[x], R[x])
```

`C` is authoritative.

The desired convergence condition is:

``` text
R[x] satisfies C[x]
```

After successful reconciliation and verification:

``` text
L[x] <- R[x]
```

Observing `R` MUST NOT modify `L`.

------------------------------------------------------------------------

## 4. Representation of L

For a versioned resource:

``` text
L[x] = (Key, version)
```

Example:

``` text
dataset ds1 -> v4
task task1  -> v7
```

The historical state can be reconstructed from the immutable server
version.

For a non-versioned resource, `L` MUST preserve enough state to
reconstruct the last acknowledged remote value.

Example:

``` text
benchmark bench1:
    tasks = {
        task1/v7,
        task2/v3
    }
```

Unified meaning:

``` text
L[x] = last acknowledged remote state of x
```

The representation differs only because versioned resources have
server-side immutable history.

------------------------------------------------------------------------

## 5. State Classification

Classification is:

``` text
classify(C, L, R) -> Classification
```

Rules:

``` text
C == L == R
    -> NO_CHANGE

L == R and C != L
    -> CONFIG_CHANGE

L == C and R != L
    -> REMOTE_DRIFT

C == R and L != C
    -> ALREADY_RECONCILED

C != L and L != R and C != R
    -> DIVERGED
```

These five cases are exhaustive and mutually exclusive.

Because `C` is authoritative:

``` text
Desired = C
```

Therefore classifications such as:

``` text
REMOTE_DRIFT
DIVERGED
```

describe why the states differ; they do not make `R` authoritative.

The eventual target remains:

``` text
R -> C
```

for managed state.

------------------------------------------------------------------------

## 6. Resolved Desired State

Configuration may contain symbolic dependency references.

Example:

``` text
C(task1):
    dataset = ds1
```

The server requires a concrete version:

``` text
R(task1):
    dataset = ds1/v5
```

Define:

``` text
resolve(C[x], resolved dependencies)
    -> D[x]
```

where:

``` text
D[x] = concrete desired remote state
```

For a resource `x` with dependencies:

``` text
D[x] =
    resolve(
        C[x],
        { D[d] for d in dependencies(x) }
    )
```

Example:

``` text
C(task1):
    dataset = ds1

resolved dependency:
    ds1 -> ds1/v5

D(task1):
    dataset = ds1/v5
```

Planning compares the effective desired state `D`, not merely the
symbolic source representation `C`.

------------------------------------------------------------------------

## 7. Version Invalidation

For a versioned resource `x`, a new version is required when its
resolved desired state differs from its last acknowledged state:

``` text
if D[x] != state(L[x]):
    x requires a new version
```

A dependency version change can therefore invalidate its dependent.

Example:

``` text
ds1/v4 -> ds1/<new>
              |
              v
task1/v7 -> task1/<new>
              |
              v
benchmark reference updated
```

The propagation rule is:

``` text
dependency version changed
    ->
dependent resolved state changed
    ->
dependent may require reconciliation
```

For versioned dependents:

``` text
resolved state changed
    -> create new version
```

For non-versioned dependents:

``` text
resolved state changed
    -> update existing resource
```

Therefore:

``` text
Dataset change
    -> new Dataset version

Task dependency change
    -> new Task version

Benchmark dependency change
    -> update Benchmark
```

Benchmark itself receives no version.

------------------------------------------------------------------------

## 8. Plan

`plan` is a pure operation:

``` text
plan(C, L, R) -> Plan
```

It MUST NOT modify:

``` text
C
L
R
```

Future server-generated versions are symbolic.

Example:

``` text
CreateVersion(ds1)
    -> $ds1.version

CreateVersion(
    task1,
    dataset = $ds1.version
)
    -> $task1.version

UpdateBenchmark(
    bench1,
    tasks = {
        $task1.version,
        ...
    }
)
```

The plan therefore forms a dependency graph of operations.

Execution order MUST respect dependencies:

``` text
Dataset < Task < Benchmark
```

Example:

``` text
CreateVersion(ds1)
        |
        v
CreateVersion(task1)
        |
        v
UpdateBenchmark(bench1)
```

The planner MUST NOT predict server-generated version numbers.

------------------------------------------------------------------------

## 9. Apply

Given plan operations:

``` text
o1, o2, ..., on
```

execute them in topological order.

For each operation:

``` text
execute(operation)
    ->
observe remote state
    ->
verify remote state satisfies desired state
```

Only after verification:

``` text
L[x] <- observed R[x]
```

`L` MUST NOT advance merely because an API request was attempted or
reported success.

Conceptually:

``` text
execute(o)
    -> observe(R)
    -> verify(R satisfies D)
    -> acknowledge
    -> L <- R
```

------------------------------------------------------------------------

## 10. Recovery

After interruption or failure, do not depend on operation history.

Re-read the remote state:

``` text
R' = observe()
```

Then recompute:

``` text
P' = plan(C, L, R')
```

Example:

``` text
initial:

C = B
L = A
R = A
```

Apply successfully changes the server:

``` text
R = B
```

but the process crashes before updating `L`:

``` text
C = B
L = A
R = B
```

On restart:

``` text
classify(B, A, B)
    -> ALREADY_RECONCILED
```

No second remote mutation is necessary.

After verification:

``` text
L <- R
```

------------------------------------------------------------------------

## 11. Level-Based Reconciliation

Correctness depends on current state:

``` text
(C, L, R)
```

not on the sequence of operations that produced it.

Therefore:

``` text
reconcile(C, L, R)
```

should be restartable from any observable state.

The reconciler asks:

``` text
What exists now?
What does configuration require?
What was last acknowledged?
What operations move R toward C?
```

It does NOT require:

``` text
What exact sequence of API calls happened previously?
```

------------------------------------------------------------------------

## 12. Convergence

For every managed resource:

``` text
R[x] satisfies resolve(C[x])
```

After successful acknowledgement:

``` text
L[x] = R[x]
```

At the stable fixed point:

``` text
C == L == R
```

where equality means semantic equality after resolving symbolic
configuration.

At this point:

``` text
classify(C, L, R)
    -> NO_CHANGE
```

and another `plan` produces no mutation.

------------------------------------------------------------------------

## 13. Core Invariants

``` text
1. C is authoritative.

2. L is the last acknowledged remote state.

3. Reading R never modifies L.

4. plan(C, L, R) is pure.

5. Server-generated versions are unknown until apply.

6. Dependencies are resolved before dependents.

7. Versioned resources create new versions when their resolved desired state changes.

8. Non-versioned resources are updated in place.

9. L advances only after remote verification.

10. Reconciliation depends on current (C, L, R), not operation history.

11. Successful reconciliation converges to C == L == R.

12. Reconciliation at the fixed point is idempotent.
```

------------------------------------------------------------------------

## Shared State Model

All resource specs use these definitions:

``` text
ResourceState<T> =
    ABSENT
    | PRESENT<T>

AcknowledgedState:
    ResourceKey -> ResourceState<CanonicalResource>

Materialization:
    ResourceKey -> concrete server identity/version

ObservedState:
    ResourceKey -> ResourceState<CanonicalResource>

RunHistory:
    ScheduleDigest -> Set<RunId>
```

Interpretation:

``` text
C = desired semantic state
L = last acknowledged canonical remote state
R = current observed canonical remote state
```

`L` is semantic ancestry. `L` is NOT the server version/materialization
map.

Observing `R` MUST NOT mutate `L`.

`Materialization` is maintained separately and is used to resolve
symbolic dependencies to concrete server versions.

For versioned resources:

``` text
Materialization[DatasetKey] -> DatasetVersion
Materialization[TaskKey]    -> TaskVersion
```

Run history is separate from resource reconciliation:

``` text
RunHistory != L
RunStatus  != R
```

Logs and output are artifacts and participate in neither resource
reconciliation nor execution identity.

------------------------------------------------------------------------

## Init

`init` initializes local reconciliation state without mutating remote
state.

``` text
init:
    Config x Remote
    -> LocalState
```

Ownership is separate from `L`.

``` text
Ownership[x] =
    UNMANAGED
    | MANAGED
```

For a configured resource not yet owned by this project:

``` text
R = ABSENT
    -> eligible for CREATE

R = PRESENT
    -> ADOPTION_REQUIRED
```

`init` MUST NOT automatically adopt an existing remote resource.

Explicit import/adoption performs:

``` text
observe
-> verify ResourceKey
-> L = canonicalized observed state
-> record Materialization
-> mark MANAGED
```

`init` MAY discover existing Runs relevant to configured ScheduleSpecs
and populate RunHistory.

`init` MUST NOT mutate remote resources, schedule Runs, wait for Runs,
download artifacts, or silently establish ownership of existing remote
resources.

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
