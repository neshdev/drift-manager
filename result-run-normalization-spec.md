# Result / Run Normalization Specification

## Purpose

The Result algebra operates on normalized values.

``` text
Result declaration --------\
                            -> CanonicalResultKey -> matching
Remote Run ----------------/
```

Normalization isolates the algebra from local references, server
representations, aliases, and raw status names.

## Canonical Result Key

``` text
CanonicalResultKey {
    model: ModelRef
    task: TaskVersionRef
}

ModelRef {
    identifier: string
}

TaskVersionRef {
    owner: string
    slug: string
    version: Version
}
```

The key contains only fields that determine whether a Run satisfies a
Result declaration.

## Result Resolution

A Result declaration contains a symbolic Task reference:

``` python
Result(Model("xyz"), task)
```

Resolve it using Task materialization:

``` text
resolve_result(
    ResultConfig,
    Materialization
)
-> CanonicalResultKey
```

Example:

``` text
Result(Model("xyz"), task)

Task materialization:
    user1/classify-cats@7

->

CanonicalResultKey {
    model: "xyz"
    task: user1/classify-cats@7
}
```

Task version is semantic Result identity.

## Canonical Run

Remote Runs are normalized into:

``` text
CanonicalRun {
    id: RunId
    key: CanonicalResultKey
    status: CanonicalRunStatus
}
```

``` text
canonicalize_run(RemoteRun)
-> CanonicalRun
```

Fields that do not affect Result matching are excluded from the key:

``` text
timestamps
queue position
worker identity
logs
outputs
server metadata
```

They may remain available on the Run but do not participate in Result
identity.

## Status Normalization

``` text
CanonicalRunStatus =
    ACTIVE
    | SUCCESS
    | FAILURE
```

Example mapping:

``` text
QUEUED  -> ACTIVE
RUNNING -> ACTIVE

ENDED   -> SUCCESS

FAILED  -> FAILURE
ERRORED -> FAILURE
```

The Result algebra operates only on `CanonicalRunStatus`.

## Matching

After normalization:

``` text
matches(Q,r)
iff
Q.key == r.key
```

Equivalently:

``` text
matches(Q,r)
iff
Q.model == r.model
AND
Q.task_version == r.task_version
```

No other Run fields participate in matching.

## Successful Match

``` text
successful_match(Q,r)
iff
matches(Q,r)
AND
r.status == SUCCESS
```

Therefore:

``` text
P(Q,H)
iff

exists r in H:
    successful_match(Q,r)
```

## Identity Rules

Model identity must be stable:

``` text
same semantic Model
-> same ModelRef
```

Task identity includes materialized version:

``` text
TaskVersionRef(owner, slug, v1)
!=
TaskVersionRef(owner, slug, v2)
```

Therefore a Run against an old Task version cannot satisfy a Result
against a new Task version.

## Normalization Laws

### Determinism

``` text
normalize(x)
==
normalize(x)
```

for the same semantic input.

### Representation Invariance

If two representations describe the same semantic identity:

``` text
semantic(x) == semantic(y)
```

then:

``` text
normalize(x) == normalize(y)
```

### Matching Consistency

If a Result and Run represent the same `(Model, TaskVersion)`:

``` text
canonicalize_result(Q)
==
canonicalize_run(r).key
```

### Irrelevant-Field Invariance

Changing a non-semantic Run field must not change its Result key.

``` text
change(logs)
change(outputs)
change(timestamp)
change(queue_metadata)

-> same CanonicalResultKey
```

### Version Sensitivity

``` text
TaskV1 != TaskV2
=>
Key(Model,TaskV1) != Key(Model,TaskV2)
```

## Boundary

Normalization answers:

``` text
"Is this Run an execution of this Result?"
```

The Result algebra answers:

``` text
"Does execution history satisfy this Result?"
```

Keep these concerns separate:

``` text
Remote representation
        |
        v
   normalization
        |
        v
CanonicalRun
        |
        v
    Q,H algebra
```

## Relationship to Resource Canonicalization

``` text
RESOURCES

Config --------\
                -> CanonicalResource -> equality
Remote --------/


RESULTS

Result --------\
                -> CanonicalResultKey -> matching
Run -----------/
```

Resource canonicalization makes `C`, `L`, and `R` comparable.

Result/Run normalization makes `Q` and Runs in `H` comparable.
