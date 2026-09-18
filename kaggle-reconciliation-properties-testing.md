# Reconciliation Properties and Testing Spec

This document defines testable properties for:

``` text
classify(C, L, R) -> Classification
```

where:

``` text
C = desired configuration
L = last acknowledged remote state
R = current observed remote state
```

The classifier depends only on equality between `C`, `L`, and `R`.

------------------------------------------------------------------------

## 1. Classification

``` text
if C == L and R == L:
    NO_CHANGE

elif C != L and R == L:
    CONFIG_CHANGE

elif C == L and R != L:
    REMOTE_DRIFT

elif C == R:
    ALREADY_RECONCILED

else:
    DIVERGED
```

Equivalent equality partition:

``` text
C == L == R                  -> NO_CHANGE
L == R and C != L            -> CONFIG_CHANGE
L == C and R != L            -> REMOTE_DRIFT
C == R and L != C            -> ALREADY_RECONCILED
C != L and L != R and C != R -> DIVERGED
```

These five cases are exhaustive and mutually exclusive.

------------------------------------------------------------------------

## 2. Pure Function Model

Treat classification as a pure function:

``` text
classify : State x State x State -> Classification
```

Properties:

``` text
same inputs  -> same output
no mutation  -> C, L, and R are unchanged
no I/O       -> classification depends only on its arguments
```

Resource reconciliation should therefore separate:

``` text
canonicalize : Resource -> State

classify :
    State x State x State
    -> Classification

resolve :
    Classification x C x L x R
    -> DesiredState
```

Conceptually:

``` text
Resource
   |
   v
canonicalize
   |
   v
State
   |
   v
classify(C, L, R)
   |
   v
Classification
```

This keeps resource semantics separate from reconciliation semantics.

------------------------------------------------------------------------

## 3. Totality

For every possible `(C, L, R)`:

``` text
classify(C, L, R)
```

MUST return exactly one classification.

Property:

``` text
forall C, L, R:
    classify(C, L, R) is one of:

        NO_CHANGE
        CONFIG_CHANGE
        REMOTE_DRIFT
        ALREADY_RECONCILED
        DIVERGED
```

There is no undefined state.

------------------------------------------------------------------------

## 4. Mutual Exclusivity

A triple `(C, L, R)` MUST NOT satisfy two classifications.

Property:

``` text
forall C, L, R:
    exactly_one(
        is_no_change(C, L, R),
        is_config_change(C, L, R),
        is_remote_drift(C, L, R),
        is_already_reconciled(C, L, R),
        is_diverged(C, L, R)
    )
```

------------------------------------------------------------------------

## 5. Reflexivity

For every state `x`:

``` text
classify(x, x, x) == NO_CHANGE
```

Example property test:

``` python
@given(states())
def test_reflexivity(x):
    assert classify(x, x, x) == NO_CHANGE
```

------------------------------------------------------------------------

## 6. Config-Only Change

For any two distinct states `x` and `y`:

``` text
x != y

classify(
    C = y,
    L = x,
    R = x
) == CONFIG_CHANGE
```

Property:

``` text
forall x != y:
    classify(y, x, x) == CONFIG_CHANGE
```

------------------------------------------------------------------------

## 7. Remote-Only Change

For any two distinct states `x` and `y`:

``` text
x != y

classify(
    C = x,
    L = x,
    R = y
) == REMOTE_DRIFT
```

Property:

``` text
forall x != y:
    classify(x, x, y) == REMOTE_DRIFT
```

------------------------------------------------------------------------

## 8. Already Reconciled

For any two distinct states `x` and `y`:

``` text
x != y

classify(
    C = y,
    L = x,
    R = y
) == ALREADY_RECONCILED
```

Property:

``` text
forall x != y:
    classify(y, x, y) == ALREADY_RECONCILED
```

This is an important crash-recovery property.

Before apply:

``` text
C = B
L = A
R = A

classify(B, A, A)
    -> CONFIG_CHANGE
```

Remote mutation succeeds:

``` text
C = B
L = A
R = B
```

Process crashes before updating `L`.

After restart:

``` text
classify(B, A, B)
    -> ALREADY_RECONCILED
```

The reconciler MUST NOT perform the remote mutation again.

It can verify the remote state and then acknowledge it:

``` text
L <- R
```

------------------------------------------------------------------------

## 9. Divergence

For three pairwise-distinct states:

``` text
x != y
y != z
x != z
```

Property:

``` text
classify(
    C = y,
    L = x,
    R = z
) == DIVERGED
```

Or:

``` text
forall pairwise_distinct(x, y, z):
    classify(y, x, z) == DIVERGED
```

`DIVERGED` describes the relationship between states.

It does not itself define resolution policy.

If configuration is authoritative:

``` text
desired = C
```

even when classification is:

``` text
DIVERGED
```

------------------------------------------------------------------------

## 10. C/R Symmetry

Classification detection is symmetric between `C` and `R` around `L`.

Define:

``` text
swap(C, L, R) = (R, L, C)
```

Classification transforms as:

``` text
NO_CHANGE          <-> NO_CHANGE
CONFIG_CHANGE      <-> REMOTE_DRIFT
REMOTE_DRIFT       <-> CONFIG_CHANGE
ALREADY_RECONCILED <-> ALREADY_RECONCILED
DIVERGED           <-> DIVERGED
```

Define:

``` python
SWAP = {
    NO_CHANGE: NO_CHANGE,
    CONFIG_CHANGE: REMOTE_DRIFT,
    REMOTE_DRIFT: CONFIG_CHANGE,
    ALREADY_RECONCILED: ALREADY_RECONCILED,
    DIVERGED: DIVERGED,
}
```

Property:

``` python
classify(R, L, C) == SWAP[classify(C, L, R)]
```

Example property test:

``` python
@given(states(), states(), states())
def test_config_remote_symmetry(c, l, r):
    original = classify(c, l, r)
    swapped = classify(r, l, c)

    assert swapped == SWAP[original]
```

Resolution policy does NOT need to be symmetric.

For example:

``` text
detection:
    C and R are symmetric

policy:
    C is authoritative
```

------------------------------------------------------------------------

## 11. Equality-Pattern Invariance

The classifier MUST NOT depend on the contents of states.

It depends only on:

``` text
C == L
L == R
C == R
```

Therefore:

``` text
(A, A, A)
```

and:

``` text
(dataset_1, dataset_1, dataset_1)
```

have identical classification.

Likewise:

``` text
(B, A, C)
```

and:

``` text
(config_state, baseline_state, remote_state)
```

have identical classification whenever all three values are distinct.

Property:

``` text
if equality_pattern(a, b, c)
   == equality_pattern(x, y, z):

    classify(a, b, c)
    == classify(x, y, z)
```

This allows reconciliation classification to be tested independently of
Kaggle resource types.

------------------------------------------------------------------------

## 12. Renaming Invariance

For any injective pure function:

``` text
f : State -> State2
```

classification MUST be invariant under `f`.

Property:

``` text
classify(C, L, R)
==
classify(f(C), f(L), f(R))
```

provided:

``` text
x == y  iff  f(x) == f(y)
```

Example:

``` text
A -> 100
B -> 200
C -> 300
```

Then:

``` text
classify(B, A, C)
==
classify(200, 100, 300)
==
DIVERGED
```

------------------------------------------------------------------------

## 13. Exhaustive Finite Test

Because classification depends only on equality, three symbolic values
are sufficient:

``` text
S = {A, B, C}
```

Generate the Cartesian product:

``` text
S x S x S
```

There are:

``` text
3 * 3 * 3 = 27
```

triples.

Test every one:

``` python
from itertools import product

states = [A, B, C]

for c, l, r in product(states, repeat=3):
    result = classify(c, l, r)

    assert result in {
        NO_CHANGE,
        CONFIG_CHANGE,
        REMOTE_DRIFT,
        ALREADY_RECONCILED,
        DIVERGED,
    }
```

These 27 inputs cover all five possible equality structures:

``` text
[A A A] -> all equal

[B A A] -> C differs

[A A B] -> R differs

[B A B] -> C and R agree

[B A C] -> all differ
```

No larger state domain is necessary to exhaustively test the equality
classifier.

------------------------------------------------------------------------

## 14. Functional Decomposition

Prefer small pure functions.

``` text
canonicalize(resource)
    -> State

equality_pattern(C, L, R)
    -> EqualityPattern

classify(pattern)
    -> Classification

desired(C, classification)
    -> C

plan(C, L, R)
    -> Operations
```

One possible intermediate type:

``` python
EqualityPattern =
    ALL_EQUAL
    | CONFIG_ONLY
    | REMOTE_ONLY
    | CONFIG_REMOTE_EQUAL
    | ALL_DISTINCT
```

Mapping:

``` text
ALL_EQUAL
    -> NO_CHANGE

CONFIG_ONLY
    -> CONFIG_CHANGE

REMOTE_ONLY
    -> REMOTE_DRIFT

CONFIG_REMOTE_EQUAL
    -> ALREADY_RECONCILED

ALL_DISTINCT
    -> DIVERGED
```

This makes `classify` a total mapping over a closed set.

------------------------------------------------------------------------

## 15. Authoritative Configuration Law

If configuration is authoritative, resolution has a simple invariant:

``` text
desired(C, L, R) = C
```

Classification explains *why* `R != C`.

It does not change the desired state.

Therefore:

``` text
NO_CHANGE
    -> desired = C

CONFIG_CHANGE
    -> desired = C

REMOTE_DRIFT
    -> desired = C

ALREADY_RECONCILED
    -> desired = C

DIVERGED
    -> desired = C
```

This cleanly separates:

``` text
classify = describe state relationship

resolve = determine target state
```

------------------------------------------------------------------------

## 16. Convergence Property

A successful reconciliation should establish:

``` text
R == C
```

After acknowledgement:

``` text
L == R
```

Therefore the fixed point is:

``` text
C == L == R
```

and:

``` text
classify(C, L, R) == NO_CHANGE
```

Property:

``` text
reconcile(C, L, R)
    -> (C, L2, R2)

success implies:

    C == R2
    L2 == R2

therefore:

    classify(C, L2, R2)
        == NO_CHANGE
```

This is the primary end-to-end reconciliation invariant.

------------------------------------------------------------------------

## 17. Idempotence

Once the system reaches the fixed point:

``` text
C == L == R
```

running reconciliation again MUST produce no remote mutation.

Property:

``` text
reconcile(C, C, C)
    -> NO_OP
```

Repeated reconciliation remains:

``` text
NO_OP
NO_OP
NO_OP
...
```

Equivalently:

``` text
reconcile(reconcile(state))
==
reconcile(state)
```

for an already successfully reconciled state.

------------------------------------------------------------------------

## 18. Testing Layers

Keep three testing concerns separate:

``` text
Layer 1: Resource semantics

Resource
    -> canonicalize
    -> State
```

Test Dataset, Task, and Benchmark canonicalization independently.

``` text
Layer 2: Reconciliation mathematics

(C, L, R)
    -> classify
```

Test exhaustively and with property-based testing.

``` text
Layer 3: Effects

Classification + DesiredState
    -> plan
    -> apply
    -> observe
```

Test that effects converge to:

``` text
C == L == R
```

The core architecture is:

``` text
Resource
    |
    v
canonicalize
    |
    v
State
    |
    +------ C
    +------ L
    +------ R
              |
              v
           classify
              |
              v
             plan
              |
              v
            apply
              |
              v
           observe
              |
              v
         C == L == R
```

------------------------------------------------------------------------

## Additional State Properties

### Absence Closure

The classifier remains total when resource values include `ABSENT`.

``` text
ResourceState<T> =
    ABSENT
    | PRESENT<T>
```

Creation example:

``` text
C = PRESENT(A)
L = ABSENT
R = ABSENT

-> CONFIG_CHANGE
```

### Observation Purity

``` text
observe(R)
```

MUST NOT change `L`.

### Init Baseline

For every existing adopted resource `x` immediately after `init`:

``` text
L[x] == canonicalize_remote(R[x])
```

For every configured resource absent remotely:

``` text
L[x] == ABSENT
```

`init` performs zero remote mutations.

### State / Materialization Separation

Changing or reading:

``` text
Materialization[x]
```

MUST NOT implicitly change:

``` text
L[x]
```

Dependency resolution uses `Materialization`, not `L`.

### Run History Separation

Changes to:

``` text
RunHistory
RunStatus
logs
output
```

MUST NOT change resource canonical digests or `C/L/R` classification.

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
