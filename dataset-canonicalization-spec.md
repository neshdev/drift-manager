# Dataset Canonicalization Spec

## 1. Purpose

Dataset canonicalization converts local configuration and remote Kaggle
state into the same deterministic representation.

``` text
DatasetConfig
    -> canonicalize
    -> CanonicalDataset

RemoteDataset
    -> canonicalize
    -> CanonicalDataset
```

The reconciliation system compares `CanonicalDataset` values rather than
raw config, filesystem, API, or cache objects.

------------------------------------------------------------------------

## 2. Dataset Identity

A dataset is logically identified by:

``` text
DatasetKey {
    owner: string
    slug: string
}
```

Identity:

``` text
key(dataset) = (owner, slug)
```

Server version is NOT part of logical identity.

``` text
user1/cats/v4
user1/cats/v5
```

both have:

``` text
DatasetKey {
    owner = "user1"
    slug = "cats"
}
```

------------------------------------------------------------------------

## 3. Dataset Configuration

Configuration may contain operational information needed to construct
the dataset.

``` text
DatasetConfig {
    owner: string
    slug: string

    title: string
    description: string

    path: LocalPath
}
```

Example:

``` text
DatasetConfig {
    owner = "user1"
    slug = "cats"

    title = "Cats"
    description = "Cat dataset"

    path = "./data/cats"
}
```

`path` is an input to canonicalization.

It is NOT part of canonical dataset state.

------------------------------------------------------------------------

## 4. Remote Dataset

A remote dataset contains server and cache information.

``` text
RemoteDataset {
    owner: string
    slug: string
    version: Version

    title: string
    description: string

    cache_path: LocalPath
}
```

Example:

``` text
RemoteDataset {
    owner = "user1"
    slug = "cats"
    version = 4

    title = "Cats"
    description = "Cat dataset"

    cache_path = ".cache/datasets/user1/cats/4"
}
```

The remote dataset files MUST be downloaded into the cache before
canonicalization.

`version` and `cache_path` are NOT part of canonical dataset state.

------------------------------------------------------------------------

## 5. Canonical File

Each file is represented by its normalized relative path and content
digest.

``` text
CanonicalFile {
    path: RelativePath
    digest: Digest
}
```

Example:

``` text
CanonicalFile {
    path = "train.csv"
    digest = "sha256:abc123..."
}
```

The digest is computed from file contents:

``` text
file_digest = SHA256(file_bytes)
```

Filesystem metadata MUST NOT affect the digest.

Examples of ignored filesystem state:

``` text
absolute path
mtime
ctime
permissions
cache location
local directory name
```

------------------------------------------------------------------------

## 6. Path Normalization

File paths MUST be relative to the dataset root.

Canonical paths MUST:

``` text
use "/"
contain no leading "./"
contain no absolute path
contain no "." path components
contain no ".." path components
```

Example:

``` text
./images/cat.jpg
    -> images/cat.jpg

images\cat.jpg
    -> images/cat.jpg
```

Canonicalization MUST reject paths that escape the dataset root.

------------------------------------------------------------------------

## 7. Canonical Dataset

The canonical representation is:

``` text
CanonicalDataset {
    type: "dataset"

    owner: string
    slug: string

    metadata: {
        title: string
        description: string
    }

    files: List<CanonicalFile>
}
```

Example:

``` json
{
  "type": "dataset",
  "owner": "user1",
  "slug": "cats",
  "metadata": {
    "title": "Cats",
    "description": "Cat dataset"
  },
  "files": [
    {
      "path": "images/1.jpg",
      "digest": "sha256:abc123..."
    },
    {
      "path": "train.csv",
      "digest": "sha256:def456..."
    }
  ]
}
```

------------------------------------------------------------------------

## 8. Excluded State

The following MUST NOT appear in `CanonicalDataset`:

``` text
local dataset path
remote cache path
server version
download timestamp
filesystem timestamps
absolute paths
API response metadata
temporary files created by the reconciler
```

These values describe how the dataset was obtained, not the semantic
dataset state.

------------------------------------------------------------------------

## 9. Local Canonicalization

Define:

``` text
canonicalize_config:
    DatasetConfig -> CanonicalDataset
```

Algorithm:

``` text
1. Construct DatasetKey(owner, slug).

2. Read semantic metadata.

3. Enumerate files under DatasetConfig.path.

4. Normalize every relative file path.

5. Compute SHA256(file_bytes) for every file.

6. Construct CanonicalFile(path, digest).

7. Sort files by canonical path.

8. Construct CanonicalDataset.
```

------------------------------------------------------------------------

## 10. Remote Materialization

Define:

``` text
fetch_remote:
    DatasetKey -> RemoteDataset
```

Remote materialization MUST:

``` text
1. Fetch dataset metadata.

2. Determine the current remote version.

3. Download the actual dataset files.

4. Store the files in the local cache.

5. Return a RemoteDataset describing the cached remote resource.
```

------------------------------------------------------------------------

## 11. Remote Canonicalization

Define:

``` text
canonicalize_remote:
    RemoteDataset -> CanonicalDataset
```

Local and remote canonicalization MUST produce the same type:

``` text
canonicalize_config(C)
    -> CanonicalDataset

canonicalize_remote(R)
    -> CanonicalDataset
```

------------------------------------------------------------------------

## 12. Canonical Equality

Two datasets are semantically equal iff their canonical representations
are equal.

``` text
dataset_equal(A, B)
    =
canonicalize(A) == canonicalize(B)
```

Structural equality requires:

``` text
A.type        == B.type
A.owner       == B.owner
A.slug        == B.slug
A.metadata    == B.metadata
A.files       == B.files
```

------------------------------------------------------------------------

## 13. Canonical JSON

`CanonicalDataset` MUST support deterministic JSON serialization.

Define:

``` text
serialize:
    CanonicalDataset -> bytes
```

Serialization rules:

``` text
encoding       = UTF-8
object keys    = sorted
file list      = sorted by canonical path
path separator = "/"
whitespace     = deterministic
Unicode        = normalized consistently
null handling  = deterministic
```

Property:

``` text
A == B
    ->
serialize(A) == serialize(B)
```

------------------------------------------------------------------------

## 14. Dataset Digest

Define:

``` text
dataset_digest:
    CanonicalDataset -> Digest
```

as:

``` text
dataset_digest(D)
    =
SHA256(serialize(D))
```

Pipeline:

``` text
file bytes
    |
    v
SHA256
    |
    v
CanonicalFile
    |
    v
CanonicalDataset
    |
    v
deterministic JSON
    |
    v
SHA256
    |
    v
DatasetDigest
```

------------------------------------------------------------------------

## 15. Digest Equality

If two canonical datasets are equal:

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

The canonical object MUST still be retained for structural diff.

------------------------------------------------------------------------

## 16. Reconciliation

Canonicalization occurs before three-state classification.

``` text
C' = canonicalize_config(C)
L' = canonicalize_baseline(L)
R' = canonicalize_remote(R)
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

## 17. Diff

Digests determine whether state changed.

Canonical objects determine what changed.

``` text
digest
    -> equality

canonical object
    -> explanation / diff
```

------------------------------------------------------------------------

## 18. Functional Model

``` text
read_local:
    DatasetConfig -> MaterializedDataset

fetch_remote:
    DatasetKey -> RemoteDataset

canonicalize:
    MaterializedDataset -> CanonicalDataset

serialize:
    CanonicalDataset -> bytes

digest:
    bytes -> Digest

classify:
    Digest x Digest x Digest
    -> Classification
```

Effectful operations occur at the boundary:

``` text
filesystem read
remote API
remote download
cache write
```

The reconciliation core operates on immutable values:

``` text
CanonicalDataset
Digest
Classification
Plan
```

------------------------------------------------------------------------

## 19. Core Properties

### Determinism

``` text
canonicalize(x) == canonicalize(x)
```

### Serialization Determinism

``` text
serialize(x) == serialize(x)
```

### Digest Determinism

``` text
digest(x) == digest(x)
```

### Location Independence

``` text
canonicalize("./foo")
==
canonicalize("/tmp/bar")
```

provided semantic metadata and file contents are identical.

### File Ordering Independence

``` text
[a.csv, b.csv]
==
[b.csv, a.csv]
```

after canonicalization.

### Local/Remote Equivalence

``` text
canonicalize_config(C)
==
canonicalize_remote(R)
```

when both represent the same semantic dataset.

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

and repeated `apply` MUST NOT create a new dataset version.

------------------------------------------------------------------------

## 20. Core Invariants

``` text
1. owner/slug defines logical dataset identity.

2. Server version is not part of canonical dataset state.

3. Local path is not part of canonical dataset state.

4. Cache path is not part of canonical dataset state.

5. File identity is (normalized relative path, content digest).

6. File content digest is SHA256(file_bytes).

7. File ordering is deterministic.

8. CanonicalDataset is deterministically JSON serializable.

9. DatasetDigest is SHA256(canonical JSON).

10. Digest is used for fast equality.

11. CanonicalDataset is retained for structural diff.

12. Local and remote representations canonicalize to the same type.

13. Remote files are materialized into cache before remote canonicalization.

14. Canonicalization must not depend on operational filesystem state.

15. Unchanged semantic state must always produce the same digest.
```

------------------------------------------------------------------------

## State and Materialization

Dataset reconciliation stores semantic state and server version
separately.

``` text
L[d] =
    ABSENT
    | PRESENT<CanonicalDataset>

Materialization[d] =
    DatasetVersion
```

After `init` for an existing Dataset:

``` text
L[d] = PRESENT(canonicalize_remote(d))
Materialization[d] = observed DatasetVersion
```

After successful creation/update:

``` text
create version
-> observe
-> verify
-> L[d] = PRESENT(observed canonical state)
-> Materialization[d] = observed DatasetVersion
```

Observing remote state alone MUST NOT advance `L`.

`Materialization[d]`, not `L[d]`, is used when resolving dependent
Tasks.

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
