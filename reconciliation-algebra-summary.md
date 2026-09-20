# Reconciliation Algebra Summary

``` text
Resources               Results
---------               -------
C, L, R                 Q, H

equality relation       satisfaction predicate

reconcile R -> C        grow H until P(Q,H)

fixed point:            fixed point:
C == L == R             P(Q,H) == true
```

Where:

``` text
C = desired resource state
L = last acknowledged resource state
R = observed resource state

Q = declarative requirement
H = observed facts/history
P = satisfaction predicate
```

Both are declarative:

``` text
Resources declare       Results declare
what state must exist   what fact must eventually be true
```
