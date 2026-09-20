# Result Declaration Algebra Specification

## Purpose

`Result(Model, Task, n_attempts=N)` is a declarative statement of
intended execution state.

``` python
result = Result(
    Model("xyz"),
    task,
    n_attempts=5,
)
```

It declares that at least `N` successful executions of the materialized
`(Model, TaskVersion)` must exist.

It does not declare a Run or an imperative submission operation.

## Declaration

Define:

``` text
Q = Result(M, T, N)
```

where:

``` text
M = Model
T = Task reference
N = required successful attempts
```

The Task reference is resolved before evaluating the Result:

``` text
resolve(Q)
=
(M, TaskVersion, N)
```

## Result Identity

The execution identity is:

``` text
ResultKey(Q)
=
(TaskVersion, Model)
```

`N` is a satisfaction requirement, not execution identity.

Therefore:

``` text
Result(M,T,1)
Result(M,T,5)
```

refer to the same population of Runs but impose different satisfaction
requirements.

A Task version change creates a different Result identity:

``` text
ResultKey(M,T1) != ResultKey(M,T2)
if
T1 != T2
```

## Execution History

Define:

``` text
H = observed execution history
```

Each Run has at least:

``` text
Run {
    id
    task_version
    model
    status
}
```

Statuses are partitioned into:

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

## Matching Runs

``` text
Runs(Q,H)
=
{
    r in H |
    r.task_version == Q.task_version
    AND
    r.model == Q.model
}
```

Partition matching Runs:

``` text
Success(Q,H)
=
{ r in Runs(Q,H) | status(r) == SUCCESS }

Active(Q,H)
=
{ r in Runs(Q,H) | status(r) == ACTIVE }

Failure(Q,H)
=
{ r in Runs(Q,H) | status(r) == FAILURE }
```

Define:

``` text
S = |Success(Q,H)|
A = |Active(Q,H)|
N = Q.n_attempts
```

## Desired-State Predicate

The declaration is satisfied iff at least `N` successful matching Runs
exist.

``` text
P(Q,H)
iff
S >= N
```

Equivalently:

``` text
P(Q,H)
iff
count(successful matching Runs) >= n_attempts
```

The original existential form is the special case:

``` text
N = 1
```

## Required Work

Active Runs reserve attempts that may satisfy the declaration.

``` text
needed(Q,H)
=
max(0, N - S - A)
```

Examples:

``` text
N=5, S=5, A=0 -> needed=0
N=5, S=3, A=2 -> needed=0
N=5, S=3, A=1 -> needed=1
N=5, S=0, A=0 -> needed=5
```

Failed Runs do not count toward satisfaction or reserve capacity.

## Evaluation

``` text
if S >= N:
    SATISFIED

elif S + A >= N:
    PROGRESSING

else:
    UNSATISFIED
```

Failure is evidence about previous attempts, not a terminal Result
state.

``` text
FAILED Run -> contributes 0
```

If failures leave the declaration short of `N`, apply may submit
replacement attempts.

## Apply Semantics

``` text
SATISFIED
    -> NOOP

PROGRESSING
    -> NOOP

UNSATISFIED
    -> SUBMIT needed(Q,H)
```

Therefore apply submits exactly enough work to make satisfaction
possible:

``` text
submit_count
=
max(0, N - S - A)
```

## Example

Declaration:

``` python
Result(model, task, n_attempts=5)
```

Observed history:

``` text
SUCCESS = 2
ACTIVE  = 1
FAILED  = 1
```

Then:

``` text
needed
=
5 - 2 - 1
=
2
```

Apply emits:

``` text
SUBMIT 2
```

After those Runs become active:

``` text
SUCCESS = 2
ACTIVE  = 3

-> PROGRESSING
-> NOOP
```

Eventually:

``` text
SUCCESS = 5

-> SATISFIED
-> NOOP
```

## Increasing n_attempts

Changing `n_attempts` expresses new desired evidence without changing
Result identity.

Initially:

``` python
Result(model, task, n_attempts=1)
```

After one success:

``` text
S = 1
N = 1

-> SATISFIED
```

If inspection suggests another observation is needed:

``` python
Result(model, task, n_attempts=2)
```

The same history is reevaluated:

``` text
S = 1
N = 2

-> UNSATISFIED
-> SUBMIT 1
```

Nothing about `(Model, TaskVersion)` changed. The satisfaction
requirement became stronger.

## State Transition

For fixed `N`:

``` text
                  SUBMIT missing attempts
UNSATISFIED --------------------------------> PROGRESSING
                                                  |
                                                  v
                                             SATISFIED
```

A failed Run may return the requirement to needing additional work:

``` text
PROGRESSING
    -- Run fails -->
UNSATISFIED
    -- SUBMIT replacement -->
PROGRESSING
```

## Fixed Point

The desired fixed point is:

``` text
S >= N
```

or:

``` text
P(Q,H) == true
```

At the fixed point:

``` text
apply(Q,H) = NOOP
```

## Apply Integration

Resources and Results remain separate declarative algebras:

``` text
Resources               Results
---------               -------
C, L, R                 Q, H

equality relation       cardinality predicate

reconcile R -> C        grow H until P(Q,H)

fixed point:            fixed point:
C == L == R             S >= N
```

where:

``` text
Q = Result(Model, TaskVersion, N)
P(Q,H) = |Success(Q,H)| >= N
```
