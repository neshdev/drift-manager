# Benchmark Canonicalization Spec

## 1. Purpose

Benchmark canonicalization converts Python configuration and remote
Kaggle state into the same deterministic representation.

A Benchmark:

``` text
has logical identity:
    owner/slug

has semantic metadata

depends on zero or more Tasks

is NOT versioned remotely
```

The important distinction is:

``` text
BenchmarkConfig
    -> references logical Tasks

CanonicalBenchmark
    -> references concrete Task versions
```

Pipeline:

``` text
BenchmarkConfig
    |
    | resolve Task dependencies
    v
CanonicalBenchmark

RemoteBenchmark
    |
    | read pinned Task versions
    v
CanonicalBenchmark
```

The reconciliation system compares `CanonicalBenchmark` values rather
than raw Python objects or API responses.

------------------------------------------------------------------------

## 2. Benchmark Identity

A Benchmark is logically identified by:

``` text
BenchmarkKey {
    owner: string
    slug: string
}
```

Identity:

``` text
key(benchmark) = (owner, slug)
```

Benchmark has no server version.

There is no:

``` text
user1/benchmark1/v2
```

There is only:

``` text
user1/benchmark1
```

Updates mutate the Benchmark in place.

------------------------------------------------------------------------

## 3. Python Configuration

A starting Python configuration:

``` python
d1 = Dataset(
    owner="user1",
    slug="dataset1",
    title="Dataset 1",
    description="Dataset used by benchmark tasks",
    path="./data/d1",
)

t1 = Task(
    owner="user1",
    slug="task1",
    title="Task 1",
    description="First benchmark task",
    dataset=d1,
)

t2 = Task(
    owner="user1",
    slug="task2",
    title="Task 2",
    description="Second benchmark task",
    dataset=d1,
)

b = Benchmark(
    owner="user1",
    slug="benchmark1",
    title="Benchmark 1",
    description="Example benchmark",
)

b.add(t1, t2)
```

The Python object references form the dependency graph:

``` text
        b
       / \
      v   v
     t1   t2
      \   /
       v v
        d1
```

Equivalent edges:

``` text
t1 -> d1
t2 -> d1

b -> t1
b -> t2
```

Conceptual configuration type:

``` text
BenchmarkConfig {
    owner: string
    slug: string

    title: string
    description: string

    tasks: Collection<TaskConfig>
}
```

The Task references in `BenchmarkConfig` are symbolic.

They do NOT contain server versions.

------------------------------------------------------------------------

## 4. Task Membership

A Benchmark may contain zero or more Tasks.

Configuration:

``` python
b.add(t1, t2)
```

means:

``` text
b -> t1
b -> t2
```

Adding a Task creates a direct dependency from the Benchmark to the
Task.

The Benchmark does NOT directly depend on the Task's Dataset.

For example:

``` text
b -> t1 -> d1
b -> t2 -> d1
```

The Benchmark stores only:

``` text
b -> t1
b -> t2
```

The Dataset dependency remains owned by each Task.

------------------------------------------------------------------------

## 5. Task Reference

The canonical Benchmark MUST NOT embed the entire canonical Task.

Instead, it stores a concrete Task reference:

``` text
TaskVersionRef {
    owner: string
    slug: string
    version: Version
}
```

Example:

``` text
TaskVersionRef {
    owner = "user1"
    slug = "task1"
    version = 3
}
```

This preserves the resource boundary:

``` text
CanonicalTask
    !=
part of CanonicalBenchmark
```

Instead:

``` text
CanonicalBenchmark
    -> TaskVersionRef
```

------------------------------------------------------------------------

## 6. Remote Benchmark

A remote Benchmark contains semantic metadata and concrete Task-version
memberships.

Conceptually:

``` text
RemoteBenchmark {
    owner: string
    slug: string

    title: string
    description: string

    tasks: Collection<TaskVersionRef>
}
```

Example:

``` text
RemoteBenchmark {
    owner = "user1"
    slug = "benchmark1"

    title = "Benchmark 1"
    description = "Example benchmark"

    tasks = {
        user1/task1/v3,
        user1/task2/v8
    }
}
```

There is no Benchmark version field.

The pinned Task versions ARE part of canonical Benchmark state.

------------------------------------------------------------------------

## 7. Canonical Benchmark

The canonical representation is:

``` text
CanonicalBenchmark {
    type: "benchmark"

    owner: string
    slug: string

    metadata: {
        title: string
        description: string
    }

    tasks: Collection<TaskVersionRef>
}
```

Example:

``` json
{
  "type": "benchmark",
  "owner": "user1",
  "slug": "benchmark1",
  "metadata": {
    "title": "Benchmark 1",
    "description": "Example benchmark"
  },
  "tasks": [
    {
      "owner": "user1",
      "slug": "task1",
      "version": 3
    },
    {
      "owner": "user1",
      "slug": "task2",
      "version": 8
    }
  ]
}
```

------------------------------------------------------------------------

## 8. Direct Dependencies Only

CanonicalBenchmark contains only direct Task dependencies.

Given:

``` text
t1 -> d1
t2 -> d1

b -> t1
b -> t2
```

Canonical state is separated by resource:

``` text
CanonicalTask(t1)
    -> d1/v5

CanonicalTask(t2)
    -> d1/v5

CanonicalBenchmark(b)
    -> t1/v3
    -> t2/v8
```

CanonicalBenchmark MUST NOT contain:

``` text
d1/v5
```

because Dataset membership is transitive state owned by the Tasks.

This produces:

``` text
CanonicalBenchmark
    |
    +-- t1/v3
    |     |
    |     +-- d1/v5
    |
    +-- t2/v8
          |
          +-- d1/v5
```

------------------------------------------------------------------------

## 9. Excluded State

The following MUST NOT appear in `CanonicalBenchmark`:

``` text
cache path
timestamps
API response metadata
Python object identity
Task metadata
Task Dataset references
Dataset contents
Dataset metadata
Dataset local paths
Dataset cache paths
transitive dependencies
```

A Task dependency is represented only by:

``` text
(owner, slug, version)
```

------------------------------------------------------------------------

## 10. Dependency Resolution

Configuration contains symbolic Task dependencies:

``` text
BenchmarkConfig {
    tasks = {
        Task("user1", "task1"),
        Task("user1", "task2")
    }
}
```

Canonical Benchmark state requires concrete Task versions:

``` text
CanonicalBenchmark {
    tasks = {
        user1/task1/v3,
        user1/task2/v8
    }
}
```

Define:

``` text
resolve_benchmark:
    BenchmarkConfig x ResolvedTasks
    -> CanonicalBenchmark
```

where:

``` text
ResolvedTasks =
    Collection<TaskVersionRef>
```

Resolution:

``` text
BenchmarkConfig
    |
    | tasks = logical Tasks
    v
resolve Tasks
    |
    | tasks = owner/slug/version
    v
CanonicalBenchmark
```

------------------------------------------------------------------------

## 11. Local Canonicalization

Benchmark configuration cannot be fully canonicalized until its Task
dependencies have been resolved.

Define:

``` text
canonicalize_config:
    BenchmarkConfig x Collection<TaskVersionRef>
    -> CanonicalBenchmark
```

Algorithm:

``` text
1. Construct BenchmarkKey(owner, slug).

2. Read semantic Benchmark metadata.

3. Resolve every Benchmark Task.

4. Obtain TaskVersionRef(owner, slug, version) for each Task.

5. Normalize Task membership.

6. Construct CanonicalBenchmark.
```

Example:

``` text
BenchmarkConfig:
    owner = user1
    slug = benchmark1
    tasks = {t1, t2}

Resolved Tasks:
    t1 = user1/task1/v3
    t2 = user1/task2/v8
```

produces:

``` text
CanonicalBenchmark:
    owner = user1
    slug = benchmark1

    tasks = {
        user1/task1/v3,
        user1/task2/v8
    }
```

------------------------------------------------------------------------

## 12. Task Membership Normalization

If Benchmark Task order has no semantic meaning, Task membership MUST be
treated as a set.

Canonical serialization MUST sort Task references by:

``` text
(owner, slug, version)
```

Therefore:

``` text
{t1/v3, t2/v8}
==
{t2/v8, t1/v3}
```

after canonicalization.

Duplicate Task references MUST collapse to one membership:

``` text
{t1/v3, t1/v3, t2/v8}
    ->
{t1/v3, t2/v8}
```

If Task ordering later becomes semantically meaningful, this invariant
MUST be changed and list order MUST be preserved.

------------------------------------------------------------------------

## 13. Remote Canonicalization

Define:

``` text
canonicalize_remote:
    RemoteBenchmark -> CanonicalBenchmark
```

Algorithm:

``` text
1. Construct BenchmarkKey(owner, slug).

2. Read semantic Benchmark metadata.

3. Read all Task memberships.

4. Construct TaskVersionRef for every Task.

5. Normalize Task membership.

6. Construct CanonicalBenchmark.
```

Local and remote canonicalization MUST produce the same type:

``` text
canonicalize_config(C, resolved_tasks)
    -> CanonicalBenchmark

canonicalize_remote(R)
    -> CanonicalBenchmark
```

------------------------------------------------------------------------

## 14. Canonical Equality

Two Benchmarks are semantically equal iff their canonical
representations are equal.

``` text
benchmark_equal(A, B)
    =
canonicalize(A) == canonicalize(B)
```

Structural equality requires:

``` text
A.type        == B.type
A.owner       == B.owner
A.slug        == B.slug
A.metadata    == B.metadata
A.tasks       == B.tasks
```

Task reference equality requires:

``` text
A.task.owner   == B.task.owner
A.task.slug    == B.task.slug
A.task.version == B.task.version
```

If Task membership is a set, source ordering does not affect equality.

------------------------------------------------------------------------

## 15. Dependency Invalidation

A Task version change changes the canonical Benchmark even if the
Benchmark's Python configuration is unchanged.

Before:

``` text
CanonicalBenchmark {
    tasks = {
        user1/task1/v3,
        user1/task2/v8
    }
}
```

After Task reconciliation:

``` text
CanonicalBenchmark {
    tasks = {
        user1/task1/v4,
        user1/task2/v9
    }
}
```

Therefore:

``` text
old CanonicalBenchmark
    !=
new CanonicalBenchmark
```

and the Benchmark requires reconciliation.

Because Benchmark is not versioned:

``` text
Benchmark canonical state changed
    ->
UPDATE Benchmark in place
```

There is no:

``` text
CREATE Benchmark version
```

------------------------------------------------------------------------

## 16. Transitive Dependency Propagation

Consider:

``` text
t1 -> d1
t2 -> d1

b -> t1
b -> t2
```

Initial resolved graph:

``` text
        b
       / \
      v   v
   t1/v3 t2/v8
      \   /
       v v
       d1/v5
```

If Dataset changes:

``` text
d1/v5
    ->
d1/v6
```

then both Tasks resolve differently:

``` text
t1:
    d1/v5 -> d1/v6

t2:
    d1/v5 -> d1/v6
```

Their canonical states change:

``` text
CanonicalTask(t1, d1/v5)
    !=
CanonicalTask(t1, d1/v6)

CanonicalTask(t2, d1/v5)
    !=
CanonicalTask(t2, d1/v6)
```

Because Tasks are versioned:

``` text
t1/v3 -> t1/v4
t2/v8 -> t2/v9
```

The Benchmark then resolves to:

``` text
before:
    b -> {t1/v3, t2/v8}

after:
    b -> {t1/v4, t2/v9}
```

Therefore:

``` text
CanonicalBenchmark_before
    !=
CanonicalBenchmark_after
```

and:

``` text
UpdateBenchmark(
    b,
    tasks = {
        t1/v4,
        t2/v9
    }
)
```

No special transitive invalidation operation is required.

Dependency propagation follows naturally from canonical dependency
resolution.

------------------------------------------------------------------------

## 17. Canonical JSON

`CanonicalBenchmark` MUST support deterministic JSON serialization.

Define:

``` text
serialize:
    CanonicalBenchmark -> bytes
```

Serialization rules:

``` text
encoding       = UTF-8
object keys    = sorted
Task entries   = sorted by (owner, slug, version)
whitespace     = deterministic
Unicode        = normalized consistently
null handling  = deterministic
```

The same canonical Benchmark MUST always produce identical serialized
bytes.

Property:

``` text
A == B
    ->
serialize(A) == serialize(B)
```

------------------------------------------------------------------------

## 18. Benchmark Digest

Define:

``` text
benchmark_digest:
    CanonicalBenchmark -> Digest
```

as:

``` text
benchmark_digest(B)
    =
SHA256(serialize(B))
```

Pipeline:

``` text
BenchmarkConfig
    |
    v
resolve Tasks
    |
    v
CanonicalBenchmark
    |
    v
deterministic JSON
    |
    v
SHA256
    |
    v
BenchmarkDigest
```

Changing any semantic Benchmark state changes the canonical input to the
digest:

``` text
title changes
    -> BenchmarkDigest changes

description changes
    -> BenchmarkDigest changes

Task added
    -> BenchmarkDigest changes

Task removed
    -> BenchmarkDigest changes

Task identity changes
    -> BenchmarkDigest changes

Task version changes
    -> BenchmarkDigest changes
```

------------------------------------------------------------------------

## 19. Digest Equality

If two canonical Benchmarks are equal:

``` text
A == B
```

then:

``` text
digest(A) == digest(B)
```

For reconciliation, cryptographic digest equality may be treated as
semantic equality subject to the standard cryptographic hash collision
assumption.

The canonical object MUST still be retained.

Use:

``` text
BenchmarkDigest
    -> fast equality

CanonicalBenchmark
    -> structural diff
```

------------------------------------------------------------------------

## 20. Last Acknowledged State

Benchmark is not versioned.

Therefore historical server versions cannot reconstruct the last
acknowledged state.

For Dataset and Task:

``` text
L[dataset] = dataset/version
L[task]    = task/version
```

may be sufficient because immutable historical versions can reconstruct
canonical state.

For Benchmark:

``` text
L[benchmark]
```

MUST preserve enough state to reconstruct the last acknowledged
`CanonicalBenchmark`.

For example:

``` json
{
  "type": "benchmark",
  "owner": "user1",
  "slug": "benchmark1",
  "metadata": {
    "title": "Benchmark 1",
    "description": "Example benchmark"
  },
  "tasks": [
    {
      "owner": "user1",
      "slug": "task1",
      "version": 3
    },
    {
      "owner": "user1",
      "slug": "task2",
      "version": 8
    }
  ]
}
```

This state means:

``` text
L[benchmark]
    =
last acknowledged remote Benchmark state
```

Observing the current remote Benchmark MUST NOT mutate `L`.

`L` advances only after deliberate successful acknowledgement.

------------------------------------------------------------------------

## 21. Reconciliation

Benchmark canonicalization occurs before three-state classification.

For Benchmark `x`:

``` text
C' = resolve_and_canonicalize(C[x])
L' = canonicalize_baseline(L[x])
R' = canonicalize_remote(R[x])
```

Then:

``` text
classify(C', L', R')
```

For fast equality:

``` text
C_digest = digest(C')
L_digest = digest(L')
R_digest = digest(R')
```

Classification:

``` text
C_digest == L_digest and R_digest == L_digest
    -> NO_CHANGE

C_digest != L_digest and R_digest == L_digest
    -> CONFIG_CHANGE

C_digest == L_digest and R_digest != L_digest
    -> REMOTE_DRIFT

C_digest == R_digest
    -> ALREADY_RECONCILED

otherwise
    -> DIVERGED
```

------------------------------------------------------------------------

## 22. Benchmark Mutation

Benchmark is updated in place.

Conceptually:

``` text
desired = resolve(BenchmarkConfig, desired Task versions)

if desired != remote:
    UpdateBenchmark(desired)
```

Unlike Dataset and Task:

``` text
Dataset changed
    -> CREATE_VERSION

Task changed
    -> CREATE_VERSION

Benchmark changed
    -> UPDATE_IN_PLACE
```

There is no server-generated future Benchmark version.

------------------------------------------------------------------------

## 23. Apply Ordering

Dependencies MUST be reconciled before dependents.

For:

``` text
t1 -> d1
t2 -> d1

b -> t1
b -> t2
```

apply order is:

``` text
1. d1
2. t1, t2
3. b
```

`t1` and `t2` may execute independently after `d1` has resolved.

Conceptually:

``` text
            d1
           /  \
          v    v
         t1    t2
          \    /
           v  v
            b
```

Example:

``` text
CreateVersion(d1)
    -> d1/v6

CreateVersion(t1, dataset=d1/v6)
    -> t1/v4

CreateVersion(t2, dataset=d1/v6)
    -> t2/v9

UpdateBenchmark(
    b,
    tasks={
        t1/v4,
        t2/v9
    }
)
```

------------------------------------------------------------------------

## 24. Diff

Digests determine whether Benchmark state changed.

Canonical objects determine what changed.

Example:

``` text
~ benchmark user1/benchmark1

  tasks:
      user1/task1/v3
      ->
      user1/task1/v4

      user1/task2/v8
      ->
      user1/task2/v9
```

Membership changes may also be expressed as:

``` text
tasks:
    + user1/task3/v2
    - user1/task2/v8
```

Therefore:

``` text
digest
    -> equality

canonical object
    -> explanation / diff
```

------------------------------------------------------------------------

## 25. Functional Model

Prefer a functional decomposition:

``` text
resolve_tasks:
    Collection<TaskConfig>
    -> Collection<TaskVersionRef>

canonicalize_config:
    BenchmarkConfig x Collection<TaskVersionRef>
    -> CanonicalBenchmark

canonicalize_remote:
    RemoteBenchmark
    -> CanonicalBenchmark

serialize:
    CanonicalBenchmark
    -> bytes

digest:
    bytes
    -> Digest

classify:
    Digest x Digest x Digest
    -> Classification
```

Effectful operations remain at the boundary:

``` text
remote API
Benchmark creation
Benchmark update
Task resolution
state reads
```

The reconciliation core operates on immutable values:

``` text
CanonicalBenchmark
TaskVersionRef
Digest
Classification
Plan
```

------------------------------------------------------------------------

## 26. Core Properties

### Determinism

``` text
canonicalize(x, dependencies)
==
canonicalize(x, dependencies)
```

### Task Order Independence

If Task membership is unordered:

``` text
canonicalize({t1, t2})
==
canonicalize({t2, t1})
```

### Duplicate Membership Idempotence

If Task membership is a set:

``` text
add(t1)
add(t1)

==
add(t1)
```

### Dependency Determinism

Given the same Benchmark configuration and the same resolved Task
versions:

``` text
resolve(b, {t1/v3, t2/v8})
==
resolve(b, {t1/v3, t2/v8})
```

### Dependency Sensitivity

For the same Benchmark configuration:

``` text
resolve(b, {t1/v3, t2/v8})
!=
resolve(b, {t1/v4, t2/v8})
```

### Transitive Dependency Sensitivity

If:

``` text
d1 changes
```

and that causes:

``` text
t1 version changes
```

then any Benchmark containing `t1` changes canonically:

``` text
TaskVersion(t1) changes
    ->
CanonicalBenchmark changes
```

### Local/Remote Equivalence

If local resolved configuration and remote Benchmark represent the same
semantic state:

``` text
canonicalize_config(C, dependencies)
==
canonicalize_remote(R)
```

### Digest Consistency

``` text
A == B
    ->
digest(A) == digest(B)
```

### Reconciliation Stability

If:

``` text
C == L == R
```

then:

``` text
classify(C, L, R)
    -> NO_CHANGE
```

and repeated `apply` MUST NOT perform another Benchmark update.

------------------------------------------------------------------------

## 27. Core Invariants

``` text
1. owner/slug defines logical Benchmark identity.

2. Benchmark is not server-versioned.

3. Benchmark changes are applied in place.

4. A Benchmark contains zero or more Tasks.

5. BenchmarkConfig references Tasks symbolically.

6. CanonicalBenchmark references concrete Task versions.

7. Task reference is (owner, slug, version).

8. The entire Task is not embedded in CanonicalBenchmark.

9. Task Dataset dependencies are not embedded in CanonicalBenchmark.

10. CanonicalBenchmark contains direct dependencies only.

11. Task version IS semantic Benchmark state.

12. Changing a resolved Task version changes CanonicalBenchmark.

13. Dataset changes propagate through Tasks into Benchmarks.

14. Dependency propagation follows the DAG.

15. No special transitive invalidation operation is required.

16. If Task order is not semantic, Task membership is canonicalized as a set.

17. Duplicate Task memberships collapse to one membership.

18. CanonicalBenchmark is deterministically JSON serializable.

19. BenchmarkDigest is SHA256(canonical JSON).

20. Digest is used for fast equality.

21. CanonicalBenchmark is retained for structural diff.

22. Local and remote representations canonicalize to the same type.

23. L for Benchmark must preserve the last acknowledged canonical state.

24. Observing remote Benchmark state MUST NOT mutate L.

25. Dataset operations execute before dependent Task operations.

26. Task operations execute before dependent Benchmark operations.

27. Unchanged semantic Benchmark state must always produce the same digest.

28. Repeated apply at the fixed point MUST NOT update the Benchmark again.
```

------------------------------------------------------------------------

## State and Dependency Resolution

Benchmark acknowledged state is:

``` text
L[b] =
    ABSENT
    | PRESENT<CanonicalBenchmark>
```

Because Benchmark is unversioned, its full canonical acknowledged state
MUST be persisted.

Benchmark Task references MUST be resolved from Task materializations:

``` text
Materialization[t1] = v7
Materialization[t2] = v3

->

CanonicalBenchmark(
    tasks = {
        t1/v7,
        t2/v3
    }
)
```

Benchmark resolution MUST NOT derive Task versions from `L[t]`.

After `init`:

``` text
L[b] = PRESENT(canonicalize_remote(b))
```

After successful in-place update:

``` text
update
-> observe
-> verify
-> L[b] = PRESENT(observed canonical state)
```

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
