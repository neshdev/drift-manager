# Resource Lifecycle and Adoption

## Recommendation

Keep ownership separate from reconciliation state.

``` text
Ownership[x] =
    UNMANAGED
    | MANAGED
```

For managed resources only:

``` text
C[x] =
    ABSENT
    | PRESENT<T>

L[x] =
    ABSENT
    | PRESENT<T>

R[x] =
    ABSENT
    | PRESENT<T>
```

This preserves the original five-case `C/L/R` reconciliation algebra.

## Resource Key

``` text
ResourceKey =
    (resource_type, owner, slug)
```

Examples:

``` text
dataset:user1/cats
task:user1/classify-cats
benchmark:user1/cat-benchmark
```

Use `ResourceKey` everywhere: state, materialization, dependencies,
operations, logs, and diagnostics.

## Ownership Invariant

``` text
Ownership[x] = MANAGED
    iff
the StateBackend contains explicit ownership for ResourceKey(x)
```

No ownership record means `UNMANAGED`; it is not a third value of `L`.

## New Resource

``` text
Config = PRESENT
Ownership = UNMANAGED
R = ABSENT

-> CREATE
```

Successful creation establishes ownership and atomically stores verified
`L` and `Materialization`.

## Existing Remote Resource

``` text
Config = PRESENT
Ownership = UNMANAGED
R = PRESENT

-> ADOPTION_REQUIRED
```

The remote resource MUST NOT be silently modified.

## Explicit Adoption

Use `import` as the CLI terminology.

``` bash
kaggle import dataset me/cats
```

``` text
observe R
-> verify ResourceKey
-> canonicalize R
-> atomically persist {
       Ownership = MANAGED
       L = R
       Materialization = observed identity/version
   }
```

Import MUST NOT mutate the remote resource.

## Init

`init` initializes state and discovers remote resources.

It MUST NOT automatically adopt existing resources.

``` text
init:
    initialize StateBackend
    discover remote resources
    report ADOPTION_REQUIRED where applicable
    perform zero remote mutations
```

## Missing Config Declaration

A resource that is managed in state but missing from config is
`ORPHANED`.

``` text
Ownership = MANAGED
Config declaration = missing

-> ORPHANED
-> NO REMOTE MUTATION
```

The user must explicitly choose:

``` text
kaggle state remove <resource>
    -> stop managing
    -> remote unchanged

kaggle destroy <resource>
    -> delete remote resource
```

Config omission MUST NOT destroy or forget a resource.

## Reconciliation Boundary

Ownership is checked before the normal classifier.

``` text
if Ownership[x] = UNMANAGED:
    handle CREATE or ADOPTION_REQUIRED
else:
    classify(C[x], L[x], R[x])
```

For managed resources:

``` text
C == L == R                  -> NO_CHANGE
L == R and C != L            -> CONFIG_CHANGE
L == C and R != L            -> REMOTE_DRIFT
C == R and L != C            -> ALREADY_RECONCILED
C != L and L != R and C != R -> DIVERGED
```

## Core Rules

``` text
1. Ownership is separate from C/L/R.
2. L remains ABSENT | PRESENT<T>.
3. Existing remote resources require explicit import.
4. Config omission produces ORPHANED, not DELETE.
5. state remove stops management without touching remote state.
6. destroy is explicit.
7. init never silently adopts resources.
8. ResourceKey = (resource_type, owner, slug).
```
