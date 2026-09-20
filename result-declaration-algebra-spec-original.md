# Result Declaration Algebra Specification

## Purpose

`Result(Model, Task)` is a declarative statement of intended execution
state.

``` python
result = Result(Model("xyz"), task)
```

It does not declare a Run, Schedule, or imperative submission operation.

It declares that a successful execution satisfying the Result
declaration must exist.

## Declaration

Define:

``` text
Q = Result(M, T)
H = observed execution history
```

After resolving the Task:

``` text
resolve(Q)
=
(M, TaskVersion)
```

## Result Identity

``` text
ResultKey
=
(TaskVersion, Model)
```

Changing the Task version creates a new requirement:

``` text
Result(M,T1) != Result(M,T2)
if
TaskVersion(T1) != TaskVersion(T2)
```

A success for one Task version does not satisfy another.

## Matching Runs

``` text
Runs(Q,H)
=
{ r in H |
    r.model == Q.model
    AND
    r.task == Q.task_version
}
```

Partition matching Runs:

``` text
Success(Q,H)
=
{ r in Runs(Q,H) |
    r.status == SUCCESS }

Active(Q,H)
=
{ r in Runs(Q,H) |
    r.status in {QUEUED, RUNNING} }

Failed(Q,H)
=
{ r in Runs(Q,H) |
    r.status in {FAILED, ERRORED} }
```

## Desired-State Predicate

``` text
P(Q,H)
iff
Success(Q,H) != empty
```

Equivalently:

``` text
P(Q,H)
iff

exists r in H:
    matches(r,Q)
    AND successful(r)
```

This is an existential desired-state declaration.

## Classification

``` text
if Success(Q,H) != empty:
    SATISFIED

elif Active(Q,H) != empty:
    PROGRESSING

elif Failed(Q,H) != empty:
    FAILED

else:
    UNSATISFIED
```

Priority:

``` text
SUCCESS > ACTIVE > FAILED > NONE
```

## Apply

``` text
SATISFIED
    -> NOOP

PROGRESSING
    -> NOOP

FAILED
    -> NOOP

UNSATISFIED
    -> SUBMIT
```

Normal apply therefore submits only when no matching execution exists.

## Retry

Failure does not automatically create an unbounded sequence of attempts.

``` text
FAILED
    -> NOOP
```

A retry is explicit:

``` text
FAILED
    -- retry -->
UNSATISFIED
    -- apply -->
SUBMIT
```

## State Transitions

``` text
                   SUBMIT
UNSATISFIED --------------------> PROGRESSING
                                      |
                         +------------+------------+
                         |                         |
                         v                         v
                    SATISFIED                   FAILED
```

Normal apply performs:

``` text
UNSATISFIED -> PROGRESSING
```

The environment performs:

``` text
PROGRESSING -> SATISFIED
PROGRESSING -> FAILED
```

## Submission Idempotency

Submission should be discoverable or idempotent.

``` text
AttemptKey
=
(ResultKey, AttemptNumber)
```

Then:

``` text
submit(AttemptKey)
submit(AttemptKey)

-> same logical Run
```

A new explicit retry increments `AttemptNumber`.

## Fixed Point

The desired fixed point is:

``` text
P(Q,H) == true
```

At the fixed point:

``` text
apply(Q,H) = NOOP
```

## Apply Integration

``` text
apply
  |
  +-- reconcile persistent resources -> C,L,R
  |
  +-- satisfy Result declarations     -> Q,H
```

Resources and Results remain separate declarative algebras:

``` text
Resources               Results
---------               -------
C, L, R                 Q, H

equality relation       existential predicate

reconcile R -> C        grow H until P(Q,H)

fixed point:            fixed point:
C == L == R             P(Q,H) == true
```
