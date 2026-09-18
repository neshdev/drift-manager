# Run Artifact Download Spec

## 1. Purpose

Runs may produce downloadable artifacts:

``` text
RunArtifacts {
    logs
    output
}
```

Artifacts belong to a specific immutable `RunId`.

Artifacts are not managed resources and do not participate in
reconciliation.

## 2. Command

``` text
download:
    RunId -> LocalArtifacts
```

CLI:

``` bash
kaggle download <run-id>
```

Example:

``` bash
kaggle download abc123
```

The command downloads both:

``` text
logs
output
```

## 3. Local Layout

Default:

``` text
.runs/
    abc123/
        logs/
            ...
        output/
            ...
```

Invariant:

``` text
RunId -> exactly one local artifact root
```

## 4. Artifact Types

``` text
RunArtifacts {
    run_id: RunId
    logs: ArtifactTree
    output: ArtifactTree
}
```

Logs are execution diagnostics.

Output is the produced result of the Run.

## 5. Run State

``` text
QUEUED
    -> artifacts may be unavailable

RUNNING
    -> logs/output may be partial

ENDED
    -> artifacts complete

FAILED
    -> artifacts may exist

ERRORED
    -> artifacts may exist
```

`download` MUST NOT require a successful Run.

## 6. Semantics

``` text
download(run_id):
    fetch Run
    fetch available logs
    fetch available output
    write local artifact tree
    return
```

`download` does not wait for Run completion by default.

## 7. Separation from Apply

``` text
apply:
    reconcile resources
    schedule Runs
    persist RunIds

download:
    retrieve Run artifacts
```

Therefore:

``` text
apply != download
```

`apply` MUST NOT implicitly download artifacts.

## 8. Canonical State

The following MUST NOT affect canonical state or digests:

``` text
RunId
RunStatus
logs
output
local download path
```

Artifacts do not participate in:

``` text
C / L / R
```

## 9. Core Invariants

``` text
1. Logs and output belong to a RunId.

2. RunId is the artifact retrieval key.

3. download retrieves both logs and output.

4. download is independent of apply.

5. download does not mutate remote state.

6. download does not affect reconciliation.

7. download may retrieve partial artifacts for active Runs.

8. failed/errored Runs may still have downloadable artifacts.

9. repeated download may safely overwrite/refresh the local artifact cache.
```

------------------------------------------------------------------------

## State Independence

`download` depends only on `RunId`.

``` text
download:
    RunId -> LocalArtifacts
```

The Run may have been discovered by `init`, created by `apply`, or
supplied directly by the user.

Artifact retrieval does not depend on:

``` text
C
L
R
Materialization
ScheduleDigest
```

`init` and `apply` MUST NOT implicitly download logs or output.

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
