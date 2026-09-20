# Result Declaration Algebra Properties

## Definitions

Let:

``` text
Q = Result(Model, TaskVersion)
H = observed execution history

P(Q,H)
iff
exists r in H:
    matches(r,Q)
    AND successful(r)
```

## 1. Total Classification

``` text
E(Q,H)
in
{
    SATISFIED,
    PROGRESSING,
    FAILED,
    UNSATISFIED
}
```

Exactly one state is returned using:

``` text
SUCCESS > ACTIVE > FAILED > NONE
```

## 2. Existential Satisfaction

One successful matching Run is sufficient.

``` text
exists SUCCESS
    ->
SATISFIED
```

The identity of the satisfying Run is irrelevant.

## 3. Success Dominance

``` text
SUCCESS + FAILED
    -> SATISFIED

SUCCESS + ACTIVE
    -> SATISFIED

SUCCESS + ACTIVE + FAILED
    -> SATISFIED
```

Other Runs cannot cancel an existing success.

## 4. Monotonicity

For append-only history:

``` text
H0 subset H1
```

then:

``` text
P(Q,H0)
    =>
P(Q,H1)
```

Once satisfied, adding executions cannot make the Result unsatisfied.

## 5. Stable Fixed Point

Once:

``` text
P(Q,H) == true
```

normal apply yields:

``` text
NOOP
```

For unchanged `Q`:

``` text
SATISFIED
-> apply
-> SATISFIED
```

## 6. Apply Idempotence

Once submission is observable:

``` text
UNSATISFIED
-> SUBMIT
-> PROGRESSING
```

Repeated apply yields:

``` text
PROGRESSING
-> NOOP
-> NOOP
```

Operationally:

``` text
A(A(Q,H)) ~ A(Q,H)
```

provided submission is discoverable or idempotent.

## 7. Failure Stability

Under explicit retry semantics:

``` text
FAILED
-> apply
-> FAILED
```

Normal apply does not create unbounded retries.

## 8. Identity Isolation

``` text
ResultKey
=
(TaskVersion, Model)
```

For:

``` text
ResultKey(Q1) != ResultKey(Q2)
```

a Run matching only `Q1` cannot satisfy `Q2`.

## 9. Dependency-Version Sensitivity

``` text
TaskV1 != TaskV2
```

implies:

``` text
Result(M,TaskV1)
!=
Result(M,TaskV2)
```

A success for an old Task version does not satisfy the new requirement.

## 10. Order Invariance

History ordering has no semantic effect.

``` text
E(Q,[r1,r2,r3])
=
E(Q,[r3,r1,r2])
```

## 11. Duplicate Invariance

Duplicate observations of the same Run do not affect evaluation.

``` text
E(Q,H)
=
E(Q,H union H)
```

History may therefore be reasoned about as a set of Run identities.

## 12. Independence

For unrelated ResultKeys, evaluation is independent.

Runs for `Q1` do not affect `Q2`.

This permits safe parallel evaluation and submission.

## 13. Commutative Merge

``` text
H1 union H2
=
H2 union H1
```

## 14. Associative Merge

``` text
(H1 union H2) union H3
=
H1 union (H2 union H3)
```

## 15. Idempotent Merge

``` text
H union H
=
H
```

## 16. Join-Semilattice

Execution history under set union forms a join-semilattice:

``` text
(H, union)
```

with:

``` text
commutativity
associativity
idempotence
```

The partial order is:

``` text
H0 <= H1
iff
H0 subset H1
```

## 17. Monotone Predicate

The satisfaction predicate is monotone over the history lattice.

``` text
H0 <= H1
AND
P(Q,H0)

=>

P(Q,H1)
```

## 18. Convergence

If a successful matching Run is eventually produced and observed:

``` text
SUBMIT
-> ACTIVE
-> SUCCESS
```

then eventually:

``` text
P(Q,H) == true
```

and by monotonicity:

``` text
eventually SATISFIED
-> permanently SATISFIED
```

for immutable `Q`.

## 19. Projection

Define:

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

Unrelated Runs have no semantic effect.

## 20. Resource / Result Duality

``` text
Resources               Results
---------               -------
C, L, R                 Q, H

equality relation       existential predicate

reconcile R -> C        grow H until P(Q,H)

fixed point:            fixed point:
C == L == R             P(Q,H) == true
```

Result-specific properties:

``` text
monotonicity
existential satisfaction
success dominance
order invariance
duplicate invariance
commutative merge
associative merge
idempotent merge
join-semilattice structure
```
