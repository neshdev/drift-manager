# Declarative Kaggle Resource Reconciliation Spec

## 1. Resource Model

Let:

\[ T `\in`{=tex}{`\text{Dataset}`{=tex}, `\text{Task}`{=tex},
`\text{Benchmark}`{=tex}} \]

A logical resource is identified by:

\[ K = (T, owner, slug) \]

Version semantics:

  Resource    Versioned
  ----------- -----------
  Dataset     Yes
  Task        Yes
  Benchmark   No

A concrete versioned resource is:

\[ V = (K, v) \]

where `v` is allocated by the server.

------------------------------------------------------------------------

## 2. Dependency Model

Configuration defines a directed acyclic graph:

\[ G = (N,E) \]

Legal dependency edges are:

\[ Task `\rightarrow`{=tex}Dataset \]

\[ Benchmark `\rightarrow`{=tex}Task \]

Dependencies are **version-pinned when materialized remotely**:

\[ Task/v_t `\rightarrow`{=tex}Dataset/v_d \]

\[ Benchmark `\rightarrow`{=tex}{Task/v_1,`\ldots`{=tex},Task/v_n} \]

Configuration references logical identities, not versions.

------------------------------------------------------------------------

## 3. Three States

For every managed resource `x`:

\[ (C_x,L_x,R_x) \]

where:

\[ C_x = `\text{desired configuration}`{=tex} \]

\[ L_x = `\text{last acknowledged remote state}`{=tex} \]

\[ R_x = `\text{current observed remote state}`{=tex} \]

**C is authoritative.**

Desired convergence:

\[ R_x `\models`{=tex}C_x \]

After successful reconciliation:

\[ L_x `\leftarrow`{=tex}R_x \]

Observation of `R` MUST NOT modify `L`.

------------------------------------------------------------------------

## 4. Representation of L

For versioned resources:

\[ L_x = (K,v) \]

Historical state can be reconstructed from the immutable server version.

Example:

``` text
dataset ds1 -> v4
task task1  -> v7
```

For non-versioned resources, `L` MUST contain sufficient state to
reconstruct the previous remote value.

Example:

``` text
benchmark bench1:
    tasks = {
        task1/v7,
        task2/v3
    }
```

------------------------------------------------------------------------

## 5. State Classification

For comparable states `C`, `L`, and `R`:

\[ classify(C,L,R)=

```{=tex}
\begin{cases}
NO\_CHANGE & C=L=R \\
CONFIG\_CHANGE & L=R\ne C \\
REMOTE\_DRIFT & L=C\ne R \\
ALREADY\_RECONCILED & C=R\ne L \\
DIVERGED & L,C,R\text{ pairwise distinct}
\end{cases}
```
\]

These five cases form the complete partition of equality relations over:

\[ {L,C,R} \]

with:

\[ B_3 = 5 \]

Since `C` is authoritative:

\[ Desired = C \]

Therefore both `REMOTE_DRIFT` and `DIVERGED` may produce:

\[ R `\rightarrow`{=tex}C \]

------------------------------------------------------------------------

## 6. Resolved Desired State

Because `C` contains symbolic dependencies, define:

\[ D_x = resolve(C_x,{D_d : d `\in`{=tex}deps(x)}) \]

where `D_x` is the concrete desired remote state.

Example:

``` text
C(task1):
    dataset = ds1

D(task1):
    dataset = ds1/v5
```

Planning compares the effective desired state `D`, not merely the source
representation `C`.

------------------------------------------------------------------------

## 7. Version Invalidation

For a versioned resource `x`, create a new version iff:

\[ D_x `\ne`{=tex}state(L_x) \]

Therefore, a dependency version change can invalidate its dependent:

\[ v(d)\_{desired} `\ne`{=tex}v(d)\_L `\Rightarrow`{=tex} x
`\text{ may require a new version}`{=tex} \]

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

The benchmark itself receives **no version**.

------------------------------------------------------------------------

## 8. Plan

`plan` is pure:

\[ P = plan(C,L,R) \]

It MUST NOT modify `C`, `L`, or `R`.

Future server versions are represented symbolically:

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

Operations execute in dependency order:

\[ Dataset `\prec`{=tex}Task `\prec`{=tex}Benchmark \]

------------------------------------------------------------------------

## 9. Apply

For operations:

\[ o_1,`\ldots`{=tex},o_n \]

in topological order:

\[ execute(o_i) `\rightarrow`{=tex} observe(R_i) `\rightarrow`{=tex}
verify(R_i `\models`{=tex}D_i) \]

Only after verification:

\[ L_i `\leftarrow`{=tex}R_i \]

`L` MUST NOT advance based solely on attempted execution.

------------------------------------------------------------------------

## 10. Recovery and Idempotence

On restart, recompute:

\[ P' = plan(C,L,R\_{current}) \]

If:

\[ R_x `\models`{=tex}D_x \]

then no remote mutation is required and the observed state may be
acknowledged:

\[ L_x `\leftarrow`{=tex}R_x \]

Reconciliation is **level-based**:

\[
`\boxed{ \text{Correctness depends on } (C,L,R), \text{ not operation history} }`{=tex}
\]

The convergence target is:

\[ `\boxed{ \forall x \in Managed(C): R_x \models resolve(C_x) }`{=tex}
\]

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
