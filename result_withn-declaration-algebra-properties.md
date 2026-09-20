# Result Declaration Algebra Properties

## Definitions

Let:

``` text
Q = Result(Model, TaskVersion, N)
H = observed execution history

S(Q,H) = successful matching Runs
A(Q,H) = active matching Runs
```

and:

``` text
P(Q,H)
iff
|S(Q,H)| >= N
```

Required work is:

``` text
needed(Q,H)
=
max(0, N - |S| - |A|)
```

## 1. Total Classification

Every Result is exactly one of:

``` text
SATISFIED
PROGRESSING
UNSATISFIED
```

with:

``` text
|S| >= N
    -> SATISFIED

|S| < N <= |S| + |A|
    -> PROGRESSING

|S| + |A| < N
    -> UNSATISFIED
```

The classification is exhaustive and mutually exclusive.

## 2. Cardinality Satisfaction

``` text
P(Q,H)
iff
|S(Q,H)| >= N
```

The original existential predicate is:

``` text
N = 1
```

Thus existential satisfaction is a special case of cardinality
satisfaction.

## 3. Exact Deficit

Apply derives the minimum additional work required:

``` text
needed
=
max(0, N - |S| - |A|)
```

After submitting exactly `needed` attempts:

``` text
|S| + |A| = N
```

assuming submissions become observable as active Runs.

## 4. Active Reservation

Active Runs reserve future satisfaction capacity.

``` text
|S| + |A| >= N
    =>
no additional submission
```

This prevents repeated apply from overscheduling while existing attempts
are queued or running.

## 5. Failure Replacement

Failed Runs contribute neither success nor active capacity.

``` text
FAILED contributes 0
```

Therefore failure naturally exposes a deficit:

``` text
ACTIVE -> FAILED

may cause:

|S| + |A| < N
```

and then:

``` text
needed > 0
```

No separate retry rule is required to satisfy `N`.

## 6. Monotone Satisfaction

For fixed `Q`, if successful history only grows:

``` text
S0 subset S1
```

then:

``` text
P(Q,H0)
    =>
P(Q,H1)
```

Once at least `N` successes exist, additional Runs cannot make the
Result unsatisfied.

## 7. Stable Fixed Point

Once:

``` text
|S| >= N
```

normal apply yields:

``` text
NOOP
```

and remains a no-op for unchanged `Q`.

``` text
SATISFIED
-> apply
-> SATISFIED
```

## 8. Apply Idempotence

Once submitted Runs are observable:

``` text
UNSATISFIED
-> SUBMIT needed
-> PROGRESSING
```

Repeated apply emits:

``` text
PROGRESSING -> NOOP
```

Therefore apply does not create duplicate attempts merely because
existing attempts have not completed.

## 9. Stronger Intent by Increasing N

For the same ResultKey:

``` text
N1 <= N2
```

implies:

``` text
P(Q[N2],H)
    =>
P(Q[N1],H)
```

A larger `N` is a stronger satisfaction requirement.

Conversely:

``` text
P(Q[N1],H)
```

does not necessarily imply:

``` text
P(Q[N2],H)
```

This allows intent to evolve declaratively:

``` text
n_attempts=1
-> inspect result
-> n_attempts=2
```

## 10. Result Identity Isolation

``` text
ResultKey
=
(TaskVersion, Model)
```

Runs matching one ResultKey do not contribute to another.

``` text
key(Q1) != key(Q2)
=>
Runs(Q1) do not satisfy Q2
```

`N` does not alter ResultKey.

## 11. Dependency-Version Sensitivity

``` text
TaskV1 != TaskV2
=>
ResultKey(M,TaskV1) != ResultKey(M,TaskV2)
```

Therefore successes for an old Task version do not satisfy a Result for
a new Task version.

## 12. Order Invariance

Run ordering has no semantic effect.

``` text
P(Q,[r1,r2,r3])
=
P(Q,[r3,r1,r2])
```

Only matching Run identities and statuses matter.

## 13. Duplicate Invariance

Duplicate observations of the same Run do not increase cardinality.

``` text
H union H = H
```

Runs must therefore be counted by unique Run identity.

## 14. Independent Evaluation

For:

``` text
ResultKey(Q1) != ResultKey(Q2)
```

evaluation is independent.

This permits parallel evaluation and submission for unrelated Results.

## 15. Commutative History Merge

For histories represented by unique Run identity:

``` text
H1 union H2
=
H2 union H1
```

## 16. Associative History Merge

``` text
(H1 union H2) union H3
=
H1 union (H2 union H3)
```

## 17. Idempotent History Merge

``` text
H union H
=
H
```

## 18. Join-Semilattice

Execution history forms a join-semilattice under union:

``` text
(H, union)
```

with:

``` text
commutativity
associativity
idempotence
```

The satisfaction predicate is monotone over this history as successful
Runs accumulate.

## 19. Convergence

Assume:

``` text
1. Q remains fixed
2. apply can submit missing attempts
3. submitted Runs become observable
4. failed attempts can be replaced
5. at least N attempts eventually succeed
```

Then eventually:

``` text
|S| >= N
```

and therefore:

``` text
P(Q,H) == true
```

Once reached, the fixed point is stable.

## 20. Submission Crash Safety

Each individual submission should be recoverable/idempotent.

For an intended attempt identity:

``` text
submit(AttemptKey)
submit(AttemptKey)

-> same logical Run
```

This prevents a crash between remote Run creation and local persistence
from accidentally increasing the attempt count.

A new intended attempt receives a new `AttemptKey`.

## 21. Projection

Only Runs matching the ResultKey affect evaluation.

``` text
project(Q,H)
=
Runs(Q,H)
```

Then:

``` text
E(Q,H)
=
E(Q,project(Q,H))
```

Unrelated Runs are irrelevant.

## 22. Resource / Result Duality

``` text
Resources               Results
---------               -------
C, L, R                 Q, H

equality relation       cardinality predicate

reconcile R -> C        grow H until P(Q,H)

fixed point:            fixed point:
C == L == R             |S(Q,H)| >= N
```

The Result algebra preserves the same high-level goal:

``` text
declare what would satisfy the user
-> observe reality
-> perform only the missing work
-> converge to a stable fixed point
```
