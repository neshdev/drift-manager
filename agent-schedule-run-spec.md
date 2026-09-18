# Agent, Schedule, and Run Spec

## 1. Purpose

Agents and Runs model execution of Tasks.

An Agent:

``` text
is a server-side value
has a harness
has a model
has no owner/slug
has no version
is not managed by reconciliation
```

A ScheduleSpec:

``` text
describes execution intent
references one concrete Task version
references one Agent
```

A Run:

``` text
is created by scheduling
has a server-generated RunId
is immutable
executes asynchronously
has observable status
```

The core relationship is:

``` text
TaskVersionRef + Agent
    |
    v
ScheduleSpec
    |
    | schedule
    v
RunId
    |
    v
Run
```

------------------------------------------------------------------------

## 2. Agent

An Agent is a server-side value.

``` text
Agent {
    harness: string
    model: string
}
```

Example:

``` python
agent = Agent(
    harness="harbor",
    model="gemini-2.5-pro",
)
```

Agent identity is:

``` text
AgentKey =
    (harness, model)
```

Agents have no version.

Agents are not created, updated, or deleted by reconciliation.

------------------------------------------------------------------------

## 3. Agent Equality

Two Agents are equal iff:

``` text
A.harness == B.harness
AND
A.model == B.model
```

------------------------------------------------------------------------

## 4. Python Configuration

Example:

``` python
d1 = Dataset(
    owner="user1",
    slug="dataset1",
    path="./data/d1",
)

t1 = Task(
    owner="user1",
    slug="task1",
    dataset=d1,
)

a1 = Agent(
    harness="harbor",
    model="gemini-2.5-pro",
)

t1.schedule(a1)
```

The schedule references the logical Task in configuration.

The Task version is resolved during planning/apply.

------------------------------------------------------------------------

## 5. Schedule Specification

A ScheduleSpec describes one resolved execution specification.

``` text
ScheduleSpec {
    task: TaskVersionRef
    agent: Agent
}
```

Example:

``` text
ScheduleSpec {
    task = {
        owner = "user1"
        slug = "task1"
        version = 7
    }

    agent = {
        harness = "harbor"
        model = "gemini-2.5-pro"
    }
}
```

The ScheduleSpec contains no RunId.

RunId does not exist until the schedule operation succeeds.

------------------------------------------------------------------------

## 6. Schedule Resolution

Configuration:

``` python
t1.schedule(a1)
```

contains:

``` text
logical Task
+
Agent
```

After Task reconciliation:

``` text
t1
    ->
user1/task1/v7
```

the schedule resolves to:

``` text
ScheduleSpec {
    task = user1/task1/v7
    agent = (harbor, gemini-2.5-pro)
}
```

Define:

``` text
resolve_schedule:
    TaskVersionRef x Agent
    -> ScheduleSpec
```

------------------------------------------------------------------------

## 7. Schedule Identity

Schedule identity is determined by:

``` text
TaskVersionRef
+
Agent
```

Therefore:

``` text
ScheduleKey =
(
    task.owner,
    task.slug,
    task.version,
    agent.harness,
    agent.model
)
```

Changing the Task version changes the ScheduleSpec:

``` text
Schedule(task1/v7, agent1)
!=
Schedule(task1/v8, agent1)
```

Changing the Agent changes the ScheduleSpec.

------------------------------------------------------------------------

## 8. Canonical Schedule

The canonical representation is:

``` text
CanonicalSchedule {
    type: "schedule"

    task: {
        owner: string
        slug: string
        version: Version
    }

    agent: {
        harness: string
        model: string
    }
}
```

Example:

``` json
{
  "type": "schedule",
  "task": {
    "owner": "user1",
    "slug": "task1",
    "version": 7
  },
  "agent": {
    "harness": "harbor",
    "model": "gemini-2.5-pro"
  }
}
```

------------------------------------------------------------------------

## 9. Schedule Digest

CanonicalSchedule MUST support deterministic JSON serialization.

``` text
ScheduleDigest =
    SHA256(canonical_json(CanonicalSchedule))
```

The digest identifies an execution specification.

It does NOT identify an individual Run.

------------------------------------------------------------------------

## 10. Schedule to Runs

One ScheduleSpec may produce multiple Runs.

``` text
ScheduleSpec
    |
    +-- Run abc123
    +-- Run def456
    +-- Run ghi789
```

Therefore:

``` text
ScheduleSpec 1 -> N Runs
```

A ScheduleDigest MUST NOT map exclusively to one RunId.

------------------------------------------------------------------------

## 11. Scheduling Operation

Scheduling is an effectful server operation.

``` text
schedule:
    ScheduleSpec -> RunId
```

RunId is allocated by the server.

The planner MUST NOT predict RunId.

------------------------------------------------------------------------

## 12. Run

A Run is the immutable materialization of a scheduling operation.

``` text
Run {
    id: RunId
    task: TaskVersionRef
    agent: Agent
}
```

Once created:

``` text
Run.id
Run.task
Run.agent
```

MUST NOT change.

If execution must happen again, create another Run.

------------------------------------------------------------------------

## 13. Run Status

Run execution is asynchronous.

``` text
RunStatus =
    QUEUED
    | RUNNING
    | ENDED
    | FAILED
    | ERRORED
```

Status is observed from the server.

Status is NOT part of immutable Run identity.

------------------------------------------------------------------------

## 14. Run State Categories

``` text
ACTIVE:
    QUEUED
    RUNNING

SUCCESS:
    ENDED

FAILURE:
    FAILED
    ERRORED
```

------------------------------------------------------------------------

## 15. Run Lifecycle

Typical successful lifecycle:

``` text
QUEUED
    |
    v
RUNNING
    |
    v
ENDED
```

Failure lifecycle:

``` text
QUEUED
    |
    v
RUNNING
    |
    +----> FAILED
    |
    +----> ERRORED
```

Scheduling logic depends primarily on:

``` text
ACTIVE
SUCCESS
FAILURE
```

------------------------------------------------------------------------

## 16. Asynchronous Apply

`apply` MUST NOT wait for a Run to complete.

Scheduling is considered successfully submitted when the server returns
a valid RunId.

``` text
apply
    |
    v
Schedule(...)
    |
    v
server accepts
    |
    v
RunId
    |
    v
persist RunId
    |
    v
apply continues
```

Therefore:

``` text
apply guarantees:
    required Run was submitted

apply does NOT guarantee:
    Run completed

apply does NOT guarantee:
    Run succeeded
```

------------------------------------------------------------------------

## 17. Persisting Run Identity

After the server returns a RunId, it MUST be persisted.

Conceptually:

``` text
RunRecord {
    schedule_digest: Digest
    run_id: RunId
}
```

Persisting the RunId allows later `apply` operations to observe the
existing Run rather than blindly scheduling another one.

------------------------------------------------------------------------

## 18. Run History

Run history is associated with ScheduleSpec.

``` text
Schedule(task1/v7, agent1)
    |
    +-- run123 -> FAILED
    +-- run456 -> FAILED
    +-- run789 -> ENDED
```

Conceptually:

``` text
RunHistory:
    ScheduleDigest -> List<RunId>
```

Run statuses are fetched from the server when needed.

------------------------------------------------------------------------

## 19. Default Scheduling Policy

The default scheduling policy is:

``` text
no previous Run
    -> SCHEDULE

active Run exists
    -> DO_NOT_SCHEDULE

successful Run exists
    -> DO_NOT_SCHEDULE

only failed/errored Runs exist
    -> SCHEDULE
```

Equivalent:

``` text
should_schedule(S, H)
    =
NOT (
    active_run_exists(S, H)
    OR
    successful_run_exists(S, H)
)
```

------------------------------------------------------------------------

## 20. Active Run

If an existing Run is:

``` text
QUEUED
```

or:

``` text
RUNNING
```

then:

``` text
should_schedule
    -> false
```

Repeated apply MUST NOT create duplicate concurrent Runs for the same
ScheduleSpec.

------------------------------------------------------------------------

## 21. Successful Run

If an existing Run is:

``` text
ENDED
```

then:

``` text
should_schedule
    -> false
```

Therefore:

``` text
exists successful Run(S)
    ->
repeated apply does not schedule S again
```

This is the execution fixed point.

------------------------------------------------------------------------

## 22. Failed Run

If all existing Runs are:

``` text
FAILED
```

or:

``` text
ERRORED
```

and no active or successful Run exists:

``` text
should_schedule
    -> true
```

Retries create new immutable Runs.

------------------------------------------------------------------------

## 23. Retry Semantics

Example:

``` text
run123
    FAILED

apply

run456
    FAILED

apply

run789
    ENDED
```

History:

``` text
Schedule(task1/v7, agent1)
    |
    +-- run123 FAILED
    +-- run456 FAILED
    +-- run789 ENDED
```

After `run789` succeeds:

``` text
next apply
    -> DO_NOT_SCHEDULE
```

------------------------------------------------------------------------

## 24. Task Version Changes

Schedule identity includes Task version.

Suppose:

``` text
task1/v7 + agent1
    -> run123
    -> ENDED
```

Then the Task changes:

``` text
task1/v7
    ->
task1/v8
```

The new ScheduleSpec is:

``` text
task1/v8 + agent1
```

and:

``` text
Schedule(task1/v7, agent1)
!=
Schedule(task1/v8, agent1)
```

Therefore the previous successful Run does NOT satisfy the new
ScheduleSpec.

The next apply schedules a new Run.

------------------------------------------------------------------------

## 25. Dependency Propagation

Given:

``` text
t1 -> d1

schedule:
    t1 + agent1
```

initial materialization:

``` text
d1/v5
    |
    v
t1/v7
    |
    v
Schedule(t1/v7, agent1)
    |
    v
run123
```

If Dataset changes:

``` text
d1/v5 -> d1/v6
```

then:

``` text
d1/v6
    |
    v
t1/v8
    |
    v
Schedule(t1/v8, agent1)
    |
    v
new Run
```

No special execution invalidation mechanism is required.

Schedule identity changes naturally because TaskVersionRef changes.

------------------------------------------------------------------------

## 26. Apply Ordering

Scheduling occurs only after the Task version is resolved.

``` text
1. reconcile Dataset

2. obtain concrete Dataset version

3. reconcile Task

4. obtain concrete Task version

5. resolve ScheduleSpec

6. inspect Run history/status

7. schedule if required
```

Apply returns after the Run has been accepted.

------------------------------------------------------------------------

## 27. Run Monitoring

Run completion is monitored separately from `apply`.

``` text
get_run_status:
    RunId -> RunStatus
```

Example:

``` text
get_run_status("abc123")
    -> RUNNING
```

Later:

``` text
get_run_status("abc123")
    -> ENDED
```

Monitoring MUST NOT require holding the original `apply` process open.

------------------------------------------------------------------------

## 28. Apply vs Monitor

Resource reconciliation and execution monitoring are separate concerns.

``` text
apply:
    reconcile resources
    resolve schedules
    submit necessary Runs
    persist RunIds
    return
```

Monitoring:

``` text
monitor:
    fetch RunId
    query server
    report RunStatus
```

Therefore:

``` text
apply != wait
```

------------------------------------------------------------------------

## 29. C/L/R Separation

Runs SHOULD NOT be forced into the normal resource `C/L/R`
reconciliation model.

`C/L/R` is used for resource state:

``` text
Dataset
Task
Benchmark
```

Run execution uses:

``` text
ScheduleSpec
RunHistory
RunStatus
RunPolicy
```

These are separate state machines.

------------------------------------------------------------------------

## 30. Functional Model

``` text
resolve_schedule:
    TaskVersionRef x Agent
    -> ScheduleSpec

canonicalize_schedule:
    ScheduleSpec
    -> CanonicalSchedule

serialize:
    CanonicalSchedule
    -> bytes

digest:
    bytes
    -> ScheduleDigest

get_history:
    ScheduleDigest
    -> RunHistory

get_status:
    RunId
    -> RunStatus

should_schedule:
    ScheduleSpec x RunHistory
    -> bool

schedule:
    ScheduleSpec
    -> RunId
```

Pure logic:

``` text
ScheduleSpec construction
canonicalization
serialization
digest
status classification
should_schedule
```

Effects:

``` text
server status query
server schedule request
RunId persistence
Run history persistence
```

------------------------------------------------------------------------

## 31. Scheduling Decision

The default scheduling decision:

``` text
should_schedule(S, H):

    if successful_run_exists(S, H):
        false

    if active_run_exists(S, H):
        false

    true
```

Where:

``` text
successful_run_exists(S, H)
    =
exists r in H(S):
    status(r) == ENDED
```

and:

``` text
active_run_exists(S, H)
    =
exists r in H(S):
    status(r) == QUEUED
    OR
    status(r) == RUNNING
```

------------------------------------------------------------------------

## 32. Core Properties

### Schedule Determinism

``` text
resolve(task/v7, agent1)
==
resolve(task/v7, agent1)
```

### Schedule Sensitivity

``` text
resolve(task/v7, agent1)
!=
resolve(task/v8, agent1)
```

### Agent Sensitivity

``` text
resolve(task/v7, agent1)
!=
resolve(task/v7, agent2)
```

### Active Run Idempotence

``` text
exists active Run(S)
    ->
apply(S) creates no new Run
```

### Successful Run Idempotence

``` text
exists successful Run(S)
    ->
apply(S) creates no new Run
```

### Failure Retry

``` text
no active Run(S)
AND
no successful Run(S)
    ->
apply(S) may create a new Run
```

### Run Immutability

For Run `r`:

``` text
r.id
r.task
r.agent
```

never change.

### Asynchronous Apply

``` text
Run accepted by server
    ->
apply may return
```

Run completion is not required.

### Dependency Sensitivity

``` text
TaskVersion changes
    ->
ScheduleSpec changes
```

Therefore:

``` text
TaskVersion changes
    ->
previous successful Run does not satisfy new ScheduleSpec
```

------------------------------------------------------------------------

## 33. Core Invariants

``` text
1. Agent is identified by (harness, model).

2. Agent has no version.

3. Agent is a server-side value, not a managed resource.

4. ScheduleSpec is identified by (TaskVersionRef, Agent).

5. Task version IS part of ScheduleSpec identity.

6. RunId is generated by the server.

7. RunId is immutable.

8. Run task and agent are immutable.

9. One ScheduleSpec may have multiple Runs.

10. ScheduleDigest identifies a ScheduleSpec, not a Run.

11. Run status is observed separately from immutable Run identity.

12. QUEUED and RUNNING are active states.

13. ENDED is a successful terminal state.

14. FAILED and ERRORED are failure terminal states.

15. apply MUST NOT wait for Run completion.

16. apply returns after required Runs have been accepted and RunIds persisted.

17. Active Runs prevent duplicate scheduling.

18. Successful Runs prevent duplicate scheduling.

19. Failed and errored Runs may be retried by creating new Runs.

20. Retries never mutate existing Runs.

21. Task version changes produce new ScheduleSpecs.

22. Successful Runs for old Task versions do not satisfy new Task versions.

23. Scheduling happens after Task version resolution.

24. Run monitoring is separate from resource reconciliation.

25. Run history is not C/L/R resource state.

26. Repeated apply with an active Run MUST NOT create another Run.

27. Repeated apply after successful completion MUST NOT create another Run.

28. A changed Task version may create a new Run on the next apply.
```

------------------------------------------------------------------------

## Local Execution State

Execution history is persisted separately from resource `C/L/R` state.

``` text
RunHistory:
    ScheduleDigest -> Set<RunId>
```

Schedule resolution uses the Task materialization map:

``` text
Materialization[task] -> TaskVersion
```

Example:

``` text
Materialization[t1] = v8
Agent = (harbor, model1)

->

ScheduleSpec(t1/v8, Agent)
```

`init` MAY discover existing Runs for configured ScheduleSpecs and
populate `RunHistory`.

This prevents the first `apply` after initialization from duplicating an
active or successful execution.

After scheduling:

``` text
schedule(S)
-> RunId

persist:
RunHistory[digest(S)] += RunId
```

The following remain distinct:

``` text
L
    acknowledged resource state

Materialization
    concrete resource versions

RunHistory
    immutable execution identities

RunStatus
    current observed execution state
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
