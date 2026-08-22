# Self-Referencing Property Bindings

## Status and intent

This document specifies self-referential assignment for a new lazy Property
implementation. It builds on [Provider API Foundations](PROVIDER_API_FOUNDATIONS.md)
but can be implemented separately from
[Derived Collection Operations](DERIVED_COLLECTION_OPERATIONS.md).

The feature allows assignment-shaped expressions such as:

```kotlin
p.set(p.map(f))
p.set(p.zip(q, f))
p.set(p + additions)
```

without creating a provider cycle. The right-hand `p` means the version of `p`
that existed immediately before the assignment. No value is realized by the
assignment.

The words **must**, **must not**, **should**, and **may** are normative.

## 1. Goals

Self-referential assignment must:

1. have the familiar meaning of `x = f(x)`, where the right-hand `x` is old;
2. remain lazy and preserve provider dependencies and missing values;
3. be atomic with respect to other mutations of the target property;
4. compose with a convention installed before or after the assignment;
5. retain only history that the current provider graph can still observe;
6. diagnose cycles that are not broken by the assignment's old-value rule.

It must not eagerly read the property, guess a fixed point, or depend on stack
overflow to detect a cycle.

## 2. Model and terminology

A public property is a mutable cell whose bindings point to immutable provider
plans:

```text
PropertyCell<T>:
    explicitHead:   unset | Plan<T>
    conventionSlot: unset | Plan<T>
    lifecycle:      mutable | changesDisallowed | finalized
    revision:       monotonically increasing identifier
```

Relevant immutable plan nodes are:

```text
Missing
Constant(value)
PropertyRead(propertyId)
ConventionRead(propertyId)
Map(source, transform, origin)
FlatMap(source, transform, origin)
Zip(left, right, transform, origin)
OrElse(primary, fallback, origin)
Fixed(result)
```

`PropertyRead(P)` reads the effective current plan of `P`.
`ConventionRead(P)` reads only `P`'s convention slot and never selects its
explicit head. These nodes are structurally visible to the implementation.
Property reads hidden inside an opaque callback are not structural reads.

A **version** is an immutable plan selected as the meaning of a property at a
particular binding operation. A version is not a realized value.

## 3. Assignment semantics

### 3.1 Ordinary assignment

When the right-hand plan contains no structural read of the target property,
`set` is ordinary replacement:

```text
P.set(Q), where Q cannot read P:
    P.explicitHead = Q
```

The previous explicit plan is not retained by `P`. It may be reclaimed when no
other provider references it.

### 3.2 Self-referential assignment

When the right-hand plan structurally reads `P`, `set` atomically captures the
plan to use for right-hand reads and substitutes it into a path-copied
right-hand plan:

```text
previous(P) =
    P.explicitHead, when the explicit slot is set
    ConventionRead(P), otherwise

P.set(Q), where Q structurally reads P:
    P.explicitHead = substitute(Q, PropertyRead(P), previous(P))
```

This is static single-assignment notation for the same rule:

```text
P1 = f(P0)
current(P) = P1
```

For example:

```text
P.set(P.map(f))

before substitution: Map(PropertyRead(P), f)
after substitution:  Map(P_previous, f)
```

All structural reads of the target in one right-hand side must be replaced by
the same previous version. Reads of other properties remain live. The original
right-hand provider must not be mutated; `set` installs a path-copied plan and
preserves shared subgraphs.

Reads hidden inside an opaque callback are not substituted:

```kotlin
p.set(provider { p.get() }) // unsupported opaque self-reference
```

This assignment is installed as an ordinary replacement because its dependency
is not structurally visible. Evaluating it forms a normal provider cycle and
must fail with a cycle diagnostic. An implementation may diagnose it earlier
when it has trustworthy metadata, but it must not make it a supported
self-reference by retaining and dynamically scoping the old value.

### 3.3 Multiple self reads

Every occurrence uses the same previous version:

```kotlin
p.set(p.zip(p, combine))
```

is equivalent to:

```text
old = previous(P)
P.explicitHead = Zip(old, old, combine)
```

The old plan may be evaluated once per evaluation session and shared by both
inputs when normal provider memoization permits it. This is sharing, not
finalization; a later query may evaluate live upstream providers again.

### 3.4 Sequential assignments

Sequential self-assignments form an immutable chain:

```kotlin
p.set(p.map(f))
p.set(p.map(g))
```

```text
P1 = Map(P0, f)
P2 = Map(P1, g)
current(P) = P2
```

Ordering is observable. An implementation may fuse or flatten the chain only
when it preserves values, missingness, failures, dependencies, and diagnostic
origins.

### 3.5 Relation to `update`

For an inspectable provider graph:

```text
P.update(f) == P.set(P.map(f))
```

`update` is the direct spelling and should avoid constructing and then
searching a temporary self-referential graph. Collection compound assignment
is similarly direct:

```text
P += R == P.set(P.plus(R))
P -= R == P.set(P.minus(R))
```

The DSL forms should call the same internal previous-version substitution
primitive.

## 4. Convention interaction

When `P` has no explicit head, a structural self-reference is substituted with
`ConventionRead(P)` rather than a snapshot of the convention slot's current
plan.
This preserves the ordering independence for which conventions exist:

```kotlin
// User action happens first.
p.set(p.map(f))

// A plugin's lazy configuration action happens later.
p.convention(c)
```

must have the same result as:

```kotlin
p.convention(c)
p.set(p.map(f))
```

Both evaluate as `f(c)`.

A later convention replacement remains observable through a self-derived
explicit chain rooted at `ConventionRead(P)`:

```kotlin
p.convention(c)
p.set(p.map(f))
p.convention(d)

// Result: f(d)
```

This does not mean that every explicit binding falls through to a convention.
An ordinary explicit provider remains authoritative even when it evaluates to
missing:

```kotlin
p.convention(c)
p.set(Provider.missing())

// P is missing, not c.
```

If a self-assignment follows that explicit missing provider, its previous
version is the explicit missing provider, not the convention.

`unset()` discards the complete explicit head, including any self-derived
chain, and exposes the current convention directly. `unsetConvention()` clears
the convention slot. A self-derived chain rooted at `ConventionRead(P)` then
becomes missing unless one of its operations explicitly supplies an `orElse`
value.

## 5. Binding algorithm

Conceptually, `set` uses a compare-and-set loop or an equivalent property-local
lock. A cached structural dependency summary allows the common non-self case to
replace the head without traversing or retaining the old plan:

```kotlin
fun set(rhs: Plan<T>) {
    while (true) {
        val oldState = state.get()
        oldState.requireMutable()

        val newHead = when {
            !rhs.structurallyReads(propertyId) -> rhs
            else -> {
                val previous = oldState.explicitHead
                    ?: ConventionRead(propertyId)
                substitute(
                    rhs,
                    PropertyRead(propertyId),
                    previous
                )
            }
        }

        val newState = oldState.copy(
            explicitHead = newHead,
            revision = oldState.revision + 1
        )
        if (state.compareAndSet(oldState, newState)) return
    }
}
```

The old plan is captured only after the mutation has acquired a coherent old
state. A failed compare-and-set attempt must rebuild the substitution against
the newly observed state.

`substitute` traverses only structurally inspectable plan nodes. It must use an
identity memo so that a shared input is copied at most once and remains shared
in the result. It may return an existing node unchanged when that node's
dependency summary proves it cannot contain `PropertyRead(P)`.

Evaluation requires no dynamic rebinding scope. A substituted plan contains
direct references to the previous immutable plan. Consequently, an opaque
provider that calls `P.get()` reads the current cell normally and is rejected
as a cycle.

## 6. Structural dependency classification

Each inspectable plan must expose the property reads represented by its
structure:

```text
PropertyRead(P)  structurally reads P
Map(S, f)        has the structural reads of S
Zip(L, R, f)     has the union of the structural reads of L and R
OrElse(A, B)     has the union of the structural reads of A and B
Opaque(callable) exposes no structural reads from inside callable
```

Structural traversal does not enter the mutable binding of another
`PropertyRead(Q)` node. That node is a live read of `Q`, not an inline copy of
Q's current plan. Consequently, `P.set(Q.map(f))` is a non-self replacement and
cuts P's old chain. If `Q` currently or later depends on `P`, ordinary cycle
detection rejects that cross-property cycle; assignment does not snapshot or
rewrite `Q`.

An opaque or dynamically selected computation may expose declared external task
dependencies without exposing hidden property reads for substitution. A hidden
read of the assignment target remains unsupported and fails as a runtime cycle.
The diagnostic should recommend `map`, `zip`, `update`, or another inspectable
operation.

Dependency metadata should be structural or persistent. Storing a copied
transitive set of every property dependency on every node can turn a linear
chain into quadratic memory consumption.

## 7. Cycle rules

Substitution breaks only cycles that pass through a structural read of the
assignment target in that assignment's right-hand plan.

Valid:

```kotlin
p.set(p.map(f))
p.set(p.zip(q, f))
```

Unsupported opaque or dynamically introduced self-reference:

```kotlin
p.set(provider { p.get() })
p.set(q.flatMap { if (it.enabled) p else defaults })
```

These expressions are not rebound. If their dynamic path reads `p`, evaluation
must report a dependency cycle.

Still invalid:

- a cycle between other properties that does not pass through a substituted
  target read;
- a convention plan that recursively requires its own effective property;
- a transform or opaque callback that reads the new property;
- mutation of the same property from inside its evaluating transform.

Cycle detection must track property identity and plan identity. A diagnostic
should show the new binding and explain that opaque self-reference is not
eligible for previous-version substitution.

## 8. Lifecycle and concurrency

- Creating a right-hand provider does not mutate or finalize its sources.
- Substitution and installation of the new explicit head are one atomic
  mutation.
- One evaluation observes one coherent property revision.
- `disallowChanges()` rejects the assignment before installing a plan.
- `finalizeValue()` evaluates the selected plan according to the lifecycle
  contract and may replace the complete chain with one `Fixed(result)` node.
- After finalization, previous plans, transforms, and provider operands may be
  reclaimed when no external provider references them.

## 9. Performance and memory

### 9.1 Time complexity

With cached structural dependency summaries and memoized path copying:

| Operation | Expected construction cost |
|---|---:|
| Non-self `set` | `O(1)` |
| Direct `P.set(P.map(f))` | `O(1)` |
| General structural self-reference | `O(nodes on RHS paths to P)` |
| `unset` or convention replacement | `O(1)` |
| Evaluation | `O(live plan nodes + value-operation cost)` |

The CAS loop may retry under concurrent mutation, but each attempt performs no
provider realization.

An evaluator must not use one JVM or native call-stack frame per chained plan
node. It should interpret long chains iteratively, flatten compatible map
chains, or use balanced nodes. Otherwise a valid configuration containing many
self-assignments can fail with stack overflow even when sufficient heap exists.

For repeated collection additions, naive immutable concatenation can make
evaluation quadratic in the number of elements. An implementation should use a
collector, builder, rope, or ordered update node so the chain is traversed once
and the final collection is built once. Subtraction and right-biased map merge
must preserve operation order.

### 9.2 What must be retained

The implementation does **not** store a log of every `set()` call.

```kotlin
p.set(a)
p.set(b)
p.set(c)
```

After the last call, `P` retains only `c`. Plans for `a` and `b` are reclaimable
unless another provider references them.

A self-referential assignment retains the previous plan because the new value
semantically depends on it:

```kotlin
p.set(a)
p.set(p.map(f)) // retains a
p.set(p.map(g)) // retains the a -> f -> g chain
```

A later non-self replacement cuts the chain:

```kotlin
p.set(c) // the a -> f -> g chain is reclaimable
```

The same is true when the replacement is a plan derived from another property:

```kotlin
p.set(a)
p.set(p.map(f)) // retains a
p.set(q.map(g)) // does not read p; the a -> f chain is reclaimable
```

`q.map(g)` remains live with respect to `q`, but it has no reason to retain an
old version of `p`.

`unset()` also cuts the explicit chain. `finalizeValue()` may collapse it to a
fixed result. External providers can independently keep old plans alive; that
is normal provider-DAG reachability rather than property history.

An opaque provider never causes conservative retention. If it hides a read of
the target property, evaluation rejects that read as a cycle.

### 9.3 Memory bounds

Let:

```text
N = number of live self-referential assignments since the last chain-cutting
    set, unset, or finalization
R = number of distinct right-hand provider nodes retained by those assignments
C = memory retained by closures, constant values, origins, and provider inputs
```

The required live structural memory is:

```text
O(N + R) nodes, plus C
```

It must not be `O(total set calls)` and must not become `O(N^2)` merely from
copied dependency or provenance metadata.

On a typical 64-bit JVM with compressed references, a small immutable plan node
is often roughly a few dozen bytes before referenced closures and metadata.
As an engineering estimate, a self-assignment commonly retains approximately
64 to 160 bytes of graph structure and small metadata, excluding captured user
objects. Ten thousand live self-assignments would therefore typically consume
on the order of 0.6 to 1.6 MB of structural memory. These numbers are not API
guarantees: object layout, call-site representation, transform objects, and
captured values can dominate the total.

The most important memory risks are:

- transforms capturing large project or model objects;
- storing a full stack trace or duplicated path string on every node;
- copying transitive dependency sets at every link;
- caching realized collection values at every intermediate node;
- never cutting or finalizing very long update chains.

Implementations should store compact or interned origin identifiers, preserve
DAG sharing, avoid caching intermediate values beyond an evaluation session,
and expose diagnostics or metrics for unusually deep live chains.

### 9.4 Optional compaction

An implementation may compact a live chain without realization by:

- representing ordered transforms in a persistent `UpdateChain` node;
- fusing maps while retaining a compact list of diagnostic origins;
- balancing long concatenation or zip structures;
- interning repeated constants and call-site metadata;
- replacing the chain with `Fixed(result)` during finalization.

Compaction must not discard task dependencies, missing-value reasons, source
locations, or observable operation ordering.

## 10. Supported implementation boundary

The supported self-reference boundary is the structurally inspectable provider
graph:

1. represent `PropertyRead`, `Map`, `Zip`, and `OrElse` as inspectable nodes;
2. index target reads nested through inspectable immutable nodes;
3. path-copy and substitute those reads with the previous plan;
4. treat opaque and dynamically introduced target reads as ordinary cycles and
   reject them with a message recommending `update` or another inspectable
   operation;
5. use an iterative evaluator or an `UpdateChain` for long sequences.

This supports `P.set(P.map(f))`, arithmetic derived from `P`, multiple target
reads, and nested reads through inspectable nodes. It has exact retention: only
a proven structural self-dependency captures the previous plan.

## 11. Minimum conformance examples

```kotlin
// Direct self-reference uses the previous version.
val p = property<String>()
p.set("a")
p.set(p.map { it + "b" })
assertValue(p, "ab")

// Repeated self-reference is ordered.
p.set(p.map { it + "c" })
assertValue(p, "abc")

// A normal set cuts the retained chain.
p.set("z")
assertValue(p, "z")

// A plan derived only from another property also cuts the chain.
val source = property<String>().value("q")
p.set(p.map { it + "-old" })
p.set(source.map { it + "-new" })
assertValue(p, "q-new")

// Convention ordering does not matter.
val q1 = property<String>()
q1.set(q1.map { it + "!" })
q1.convention("hello")
assertValue(q1, "hello!")

val q2 = property<String>()
q2.convention("hello")
q2.set(q2.map { it + "!" })
assertValue(q2, "hello!")

// A convention replacement remains live through a convention-rooted chain.
q2.convention("goodbye")
assertValue(q2, "goodbye!")

// An explicit missing provider still shadows the convention.
val r = property<String>()
r.convention("default")
r.set(Provider.missing())
assertMissing(r)

// Multiple reads use one previous version.
val n = property<Int>()
n.set(2)
n.set(n.zip(n) { left, right -> left + right })
assertValue(n, 4)

// update is equivalent to inspectable self-reference.
val a = property<Int>().value(1)
val b = property<Int>().value(1)
a.update { it + 1 }
b.set(b.map { it + 1 })
assertValue(a, b.get())

// An opaque self-reference is not substituted.
val opaque = property<String>().value("old")
opaque.set(provider { opaque.get() })
assertCycle { opaque.get() }
```

## 12. Design summary

```text
ordinary set:
    replace the explicit head and retain no property history

self-referential set:
    bind target reads to the previous immutable plan and install a new head

no prior explicit head:
    bind target reads to the live convention slot

structurally non-self RHS:
    no substitution and no old-plan retention

structurally self-referential RHS:
    retain exactly the previous plans reachable from the substituted RHS

opaque self-reference:
    do not retain the old plan; report a normal cycle when evaluated

later non-self set, unset, or finalization:
    cut or collapse the retained chain
```

The provider DAG is the history. No separate unbounded assignment log is
required.
