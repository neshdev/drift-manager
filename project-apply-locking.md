# Project Apply Locking

## Recommendation

Use a single project-level OS file lock.

V1 permits at most one state-mutating process per project.

``` text
at most one local state-mutating command
per project
at any time
```

## Layout

``` text
.kaggle/
    state.db
    apply.lock
```

`apply.lock` is used as the target of an OS-level file lock.

Do not use file existence alone as the locking mechanism.

## Mutating Commands

Commands that mutate durable state or remote state MUST acquire the
project lock.

Examples:

``` text
init
apply
import
destroy
```

Read-only commands such as `plan` MAY execute without the lock.

## Execution

``` text
acquire project lock
    |
    +-- unavailable
    |       ->
    |   fail immediately
    |
    +-- acquired
            |
            v
        execute command
            |
            v
        release lock
```

Conceptually:

``` python
with project_lock():
    load_state()
    observe_remote()
    plan()
    execute()
```

## Crash Behavior

Use an OS-level lock so ownership is tied to the process.

``` text
process exits normally
    -> lock released

process crashes
    -> OS releases lock

kill -9
    -> OS releases lock
```

A stale lock file on disk MUST NOT itself imply that the project is
locked.

## SQLite

The project lock and SQLite transactions solve different problems.

``` text
project lock
    prevents multiple mutating commands

SQLite transaction
    makes individual durable state commits atomic
```

Do not rely on a long-running SQLite transaction as the project lock.

Remote API calls MUST occur outside SQLite transactions.

## Future Backends

Distributed locking is out of scope for V1.

Future shared `StateBackend` implementations MAY provide
backend-specific locking or leases.

``` text
SQLiteStateBackend
    -> local OS project lock

SharedStateBackend
    -> future distributed lock/lease
```

The reconciler should depend on locking semantics, not a specific
distributed locking implementation.

## Core Rules

\`\`\`text 1. One mutating command per local project.

2.  Use an OS-level file lock.

3.  File existence is not lock ownership.

4.  Crashes automatically release the lock.

5.  Keep SQLite transactions short.

6.  plan may remain read-only and unlocked.

7.  Distributed locking is deferred until shared backends exist.

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
