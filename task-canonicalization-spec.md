# Task Canonicalization Spec

## 1. Purpose

Task canonicalization converts Python configuration and remote Kaggle
state into the same deterministic representation.

A Task:

``` text
has logical identity:
    owner/slug

has semantic metadata

depends on exactly one Dataset

is versioned remotely
```

The important distinction is:

``` text
TaskConfig
    -> references a logical Dataset

CanonicalTask
    -> references a concrete Dataset version
```

Pipeline:

``` text
TaskConfig
    |
    | resolve Dataset dependency
    v
CanonicalTask

RemoteTask
    |
    | read pinned Dataset version
    v
CanonicalTask
```

The reconciliation system compares `CanonicalTask` values rather than
raw Python objects or API responses.

------------------------------------------------------------------------

## 2. Task Identity

A Task is logically identified by:

``` text
TaskKey {
    owner: string
    slug: string
}
```

Identity:

``` text
key(task) = (owner, slug)
```

Server version is NOT part of logical identity.

For example:

``` text
user1/classify-cats/v3
user1/classify-cats/v4
```

both have:

``` text
TaskKey {
    owner = "user1"
    slug = "classify-cats"
}
```

------------------------------------------------------------------------

## 3. Python Configuration

A starting Python configuration:

``` python
dataset = Dataset(
    owner="user1",
    slug="cats",
    title="Cats Dataset",
    description="Training data for cat classification",
    path="./data/cats",
)

task = Task(
    owner="user1",
    slug="classify-cats",
    title="Classify Cats",
    description="Classify images of cats",
    dataset=dataset,
)
```

The Python object reference forms the dependency graph:

``` text
task
 |
 v
dataset
```

Conceptual configuration type:

``` text
TaskConfig {
    owner: string
    slug: string

    title: string
    description: string

    dataset: DatasetConfig
}
```

The Dataset reference in `TaskConfig` is symbolic.

It does NOT contain a server version.

------------------------------------------------------------------------

## 4. Dependency Semantics

A Task depends on exactly one Dataset.

Configuration dependency:

``` text
TaskConfig
    -> DatasetConfig
```

Remote dependency:

``` text
Task/version
    -> Dataset/version
```

Example:

``` text
Task user1/classify-cats/v3
    -> Dataset user1/cats/v5
```

The Dataset dependency is version-pinned when materialized remotely.

------------------------------------------------------------------------

## 5. Dataset Reference

The canonical Task MUST NOT embed the entire canonical Dataset.

Instead, it stores a concrete Dataset reference:

``` text
DatasetVersionRef {
    owner: string
    slug: string
    version: Version
}
```

Example:

``` text
DatasetVersionRef {
    owner = "user1"
    slug = "cats"
    version = 5
}
```

This preserves the resource boundary:

``` text
CanonicalDataset
    !=
part of CanonicalTask
```

Instead:

``` text
CanonicalTask
    -> DatasetVersionRef
```

------------------------------------------------------------------------

## 6. Remote Task

A remote Task contains server information and a pinned Dataset
dependency.

Conceptually:

``` text
RemoteTask {
    owner: string
    slug: string
    version: Version

    title: string
    description: string

    dataset: DatasetVersionRef
}
```

Example:

``` text
RemoteTask {
    owner = "user1"
    slug = "classify-cats"
    version = 3

    title = "Classify Cats"
    description = "Classify images of cats"

    dataset = {
        owner = "user1"
        slug = "cats"
        version = 5
    }
}
```

The Task's own server version is NOT part of canonical Task state.

The pinned Dataset version IS part of canonical Task state.

------------------------------------------------------------------------

## 7. Canonical Task

The canonical representation is:

``` text
CanonicalTask {
    type: "task"

    owner: string
    slug: string

    metadata: {
        title: string
        description: string
    }

    dataset: DatasetVersionRef
}
```

Example:

``` json
{
  "type": "task",
  "owner": "user1",
  "slug": "classify-cats",
  "metadata": {
    "title": "Classify Cats",
    "description": "Classify images of cats"
  },
  "dataset": {
    "owner": "user1",
    "slug": "cats",
    "version": 5
  }
}
```

------------------------------------------------------------------------

## 8. Excluded State

The following MUST NOT appear in `CanonicalTask`:

``` text
Task server version
cache path
timestamps
API response metadata
Python object identity
Dataset local path
Dataset cache path
Dataset contents
Dataset metadata
```

The Dataset is represented only by:

``` text
(owner, slug, version)
```

------------------------------------------------------------------------

## 9. Dependency Resolution

Configuration contains a symbolic Dataset dependency:

``` text
TaskConfig {
    dataset = Dataset("user1", "cats")
}
```

Canonical Task state requires a concrete Dataset version:

``` text
CanonicalTask {
    dataset = user1/cats/v5
}
```

Define:

``` text
resolve_task:
    TaskConfig x ResolvedDataset
    -> CanonicalTask
```

where:

``` text
ResolvedDataset {
    owner
    slug
    version
}
```

Resolution:

``` text
TaskConfig
    |
    | dataset = logical Dataset
    v
resolve Dataset
    |
    | dataset = owner/slug/version
    v
CanonicalTask
```

------------------------------------------------------------------------

## 10. Local Canonicalization

Task configuration cannot be fully canonicalized until its Dataset
dependency has been resolved.

Define:

``` text
canonicalize_config:
    TaskConfig x DatasetVersionRef
    -> CanonicalTask
```

Algorithm:

``` text
1. Construct TaskKey(owner, slug).

2. Read semantic Task metadata.

3. Resolve TaskConfig.dataset.

4. Obtain DatasetVersionRef(owner, slug, version).

5. Construct CanonicalTask.
```

Example:

``` text
TaskConfig:
    owner = user1
    slug = classify-cats
    dataset = Dataset(user1, cats)

Resolved Dataset:
    user1/cats/v5
```

produces:

``` text
CanonicalTask:
    owner = user1
    slug = classify-cats
    dataset = user1/cats/v5
```

------------------------------------------------------------------------

## 11. Remote Canonicalization

Define:

``` text
canonicalize_remote:
    RemoteTask -> CanonicalTask
```

Algorithm:

``` text
1. Construct TaskKey(owner, slug).

2. Read semantic Task metadata.

3. Read the pinned Dataset reference.

4. Construct DatasetVersionRef.

5. Construct CanonicalTask.

6. Discard the Task's own server version.
```

Local and remote canonicalization MUST produce the same type:

``` text
canonicalize_config(C, resolved_dataset)
    -> CanonicalTask

canonicalize_remote(R)
    -> CanonicalTask
```

------------------------------------------------------------------------

## 12. Canonical Equality

Two Tasks are semantically equal iff their canonical representations are
equal.

``` text
task_equal(A, B)
    =
canonicalize(A) == canonicalize(B)
```

Structural equality requires:

``` text
A.type        == B.type
A.owner       == B.owner
A.slug        == B.slug
A.metadata    == B.metadata
A.dataset     == B.dataset
```

Dataset reference equality requires:

``` text
A.dataset.owner   == B.dataset.owner
A.dataset.slug    == B.dataset.slug
A.dataset.version == B.dataset.version
```

------------------------------------------------------------------------

## 13. Dependency Invalidation

A Dataset version change changes the canonical Task even if the Task's
Python configuration is unchanged.

Before:

``` text
CanonicalTask {
    dataset = user1/cats/v5
}
```

After Dataset reconciliation:

``` text
CanonicalTask {
    dataset = user1/cats/v6
}
```

Therefore:

``` text
old CanonicalTask != new CanonicalTask
```

and the Task requires reconciliation.

For a versioned Task:

``` text
Task canonical state changed
    ->
create new Task version
```

Example:

``` text
Dataset:
    user1/cats/v5
        ->
    user1/cats/v6

Task:
    user1/classify-cats/v3
        ->
    user1/classify-cats/v4

New dependency:
    task/v4 -> dataset/v6
```

No special "invalidate Task" operation is required.

The ordinary canonical equality comparison detects the dependency
change.

------------------------------------------------------------------------

## 14. Canonical JSON

`CanonicalTask` MUST support deterministic JSON serialization.

Define:

``` text
serialize:
    CanonicalTask -> bytes
```

Serialization rules:

``` text
encoding       = UTF-8
object keys    = sorted
whitespace     = deterministic
Unicode        = normalized consistently
null handling  = deterministic
```

The same canonical Task MUST always produce identical serialized bytes.

Property:

``` text
A == B
    ->
serialize(A) == serialize(B)
```

------------------------------------------------------------------------

## 15. Task Digest

Define:

``` text
task_digest:
    CanonicalTask -> Digest
```

as:

``` text
task_digest(T)
    =
SHA256(serialize(T))
```

Pipeline:

``` text
TaskConfig
    |
    v
resolve Dataset
    |
    v
CanonicalTask
    |
    v
deterministic JSON
    |
    v
SHA256
    |
    v
TaskDigest
```

Changing any semantic Task state changes the canonical input to the
digest:

``` text
title changes
    -> TaskDigest changes

description changes
    -> TaskDigest changes

Dataset identity changes
    -> TaskDigest changes

Dataset version changes
    -> TaskDigest changes
```

------------------------------------------------------------------------

## 16. Digest Equality

If two canonical Tasks are equal:

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
TaskDigest
    -> fast equality

CanonicalTask
    -> structural diff
```

------------------------------------------------------------------------

## 17. Reconciliation

Task canonicalization occurs before three-state classification.

For Task `x`:

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

## 18. New Task Version

A new Task version is required when the authoritative resolved desired
state differs from the current remote Task state.

Conceptually:

``` text
desired = resolve(TaskConfig, desired Dataset version)

if desired != remote:
    create new Task version
```

The server allocates the new Task version.

The planner MUST NOT predict it.

Example plan:

``` text
Dataset user1/cats:
    v5 -> <new>

Task user1/classify-cats:
    v3 -> <new>

    dataset:
        user1/cats/v5
        ->
        user1/cats/<new>
```

Apply resolves versions in dependency order:

``` text
create Dataset
    |
    v
server returns Dataset/v6
    |
    v
create Task(dataset=Dataset/v6)
    |
    v
server returns Task/v4
```

------------------------------------------------------------------------

## 19. Diff

Digests determine whether Task state changed.

Canonical objects determine what changed.

Example:

``` text
~ task user1/classify-cats

  metadata.title:
      "Classify Cats"
      ->
      "Cat Classification"

  dataset:
      user1/cats/v5
      ->
      user1/cats/v6
```

Therefore:

``` text
digest
    -> equality

canonical object
    -> explanation / diff
```

------------------------------------------------------------------------

## 20. Functional Model

Prefer a functional decomposition:

``` text
resolve_dataset:
    DatasetConfig
    -> DatasetVersionRef

canonicalize_config:
    TaskConfig x DatasetVersionRef
    -> CanonicalTask

canonicalize_remote:
    RemoteTask
    -> CanonicalTask

serialize:
    CanonicalTask
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
Task creation
Dataset resolution
state/cache reads
```

The reconciliation core operates on immutable values:

``` text
CanonicalTask
DatasetVersionRef
Digest
Classification
Plan
```

------------------------------------------------------------------------

## 21. Core Properties

### Determinism

``` text
canonicalize(x, dependency)
==
canonicalize(x, dependency)
```

### Dependency Determinism

Given the same Task configuration and the same Dataset version:

``` text
resolve(task, dataset/v5)
==
resolve(task, dataset/v5)
```

### Dependency Sensitivity

For the same Task configuration:

``` text
resolve(task, dataset/v5)
!=
resolve(task, dataset/v6)
```

### Task-Version Independence

Changing only the Task's own server version MUST NOT change its
canonical state.

``` text
Task/v3 -> canonical X
Task/v4 -> canonical X
```

provided metadata and Dataset reference are identical.

### Local/Remote Equivalence

If local resolved configuration and remote Task represent the same
semantic state:

``` text
canonicalize_config(C, dependency)
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

and repeated `apply` MUST NOT create a new Task version.

------------------------------------------------------------------------

## 22. Core Invariants

``` text
1. owner/slug defines logical Task identity.

2. Task server version is not part of canonical Task state.

3. A Task depends on exactly one Dataset.

4. TaskConfig references the Dataset symbolically.

5. CanonicalTask references a concrete Dataset version.

6. Dataset reference is (owner, slug, version).

7. The entire Dataset is not embedded in CanonicalTask.

8. Dataset contents are not embedded in CanonicalTask.

9. Dataset version IS semantic Task state.

10. Changing the resolved Dataset version changes CanonicalTask.

11. Changing CanonicalTask requires a new remote Task version.

12. CanonicalTask is deterministically JSON serializable.

13. TaskDigest is SHA256(canonical JSON).

14. Digest is used for fast equality.

15. CanonicalTask is retained for structural diff.

16. Local and remote representations canonicalize to the same type.

17. Server-generated Task versions are unknown until apply.

18. Dataset operations execute before dependent Task operations.

19. Unchanged semantic Task state must always produce the same digest.

20. Repeated apply at the fixed point MUST NOT create a new Task version.
```

------------------------------------------------------------------------

## State, Materialization, and Resolution

Task state and concrete server version are separate:

``` text
L[t] =
    ABSENT
    | PRESENT<CanonicalTask>

Materialization[t] =
    TaskVersion
```

Task dependency resolution MUST use the Dataset materialization map:

``` text
resolve:
    TaskConfig x Materialization
    -> CanonicalTask
```

Example:

``` text
TaskConfig(dataset=d1)
+
Materialization[d1] = v6

->

CanonicalTask(dataset=d1/v6)
```

It MUST NOT derive the Dataset version from `L[d1]`.

After `init` for an existing Task:

``` text
L[t] = PRESENT(canonicalize_remote(t))
Materialization[t] = observed TaskVersion
```

After successful Task version creation:

``` text
create version
-> observe
-> verify
-> L[t] = PRESENT(observed canonical state)
-> Materialization[t] = observed TaskVersion
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
