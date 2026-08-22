# Derived Collection Operations

## Status and intent

This document defines collection operations built from the primitive Provider
API. It specifies `plus` and `minus` for `ListProperty`, `SetProperty`, and
`MapProperty`, together with their compound forms.

It builds on:

- [Provider API Foundations](PROVIDER_API_FOUNDATIONS.md) for property
  selection, conventions, missing values, `map`, `zip`, and `orElse`;
- [Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md) for assigning an
  expression such as `P.set(P.plus(R))` back to `P`.

The term **derived** is intentional. These operations add collection algebra,
not another property or convention model.

The words **must**, **must not**, **should**, and **may** are normative.

## 1. Vocabulary

Derivation and mutation are different operations:

```text
P + R   == P.plus(R)        derive a Provider; do not change P
P - R   == P.minus(R)       derive a Provider; do not change P
P += R  == P.plusAssign(R)  rebind P from its previous version
P -= R  == P.minusAssign(R) rebind P from its previous version
```

Named forms should distinguish one element from many elements:

```text
plusElement(element)       minusElement(element)
plusAll(elements)          minusAll(elements)
```

This avoids ambiguous overloads when an element is itself a collection.
Collection operators belong only to types with a declared collection algebra;
they are not generic arithmetic for scalar `Property<T>`.

## 2. Collection algebra

The table defines ordinary `A + B` and `A - B` notation for each property type.

| Property | `A + B` | `A - B` |
|---|---|---|
| `ListProperty<E>` | Concatenate the elements of `B` after `A`; retain duplicates. | Remove every element of `A` equal to any element in `B`; preserve survivor order. |
| `SetProperty<E>` | Insertion-ordered union: retain positions from `A`, then append new values from `B`. | Set difference; preserve survivor order. |
| `MapProperty<K,V>` | Right-biased merge: values from `B` replace duplicate keys. | Remove the keys supplied by `B`. |

For a map merge, replacing an existing value does not move its key. New keys
follow in the iteration order of `B`. Map subtraction accepts keys, not
key-value pairs. List subtraction ignores duplicate counts in `B`; every
matching value is removed from `A`. Element and key comparisons use their
declared equality.

Examples:

```text
[a, a, b] + [b, c]             = [a, a, b, b, c]
[a, a, b] - [b, c]             = [a, a]

{a, b} + {b, c}                = {a, b, c}
{a, b} - {b, c}                = {a}

{a: 1, b: 2} + {b: 3, c: 4}   = {a: 1, b: 3, c: 4}
{a: 1, b: 2} - {b, c}          = {a: 1}
```

Each collection kind has an empty identity:

```text
A + empty = A        empty + A = A
A - empty = A        empty - A = empty
```

Subtraction is not an inverse of addition. In particular, an implementation
must not rewrite `(A + B) - B` to `A`. Such a rewrite loses list duplicates
and overwritten map values.

## 3. Definition from primitive operations

### 3.1 Algebra and normalization

Addition and subtraction may accept different operand types. For example, map
addition accepts entries while map subtraction accepts keys:

```kotlin
interface CollectionAlgebra<C : Any, A : Any, S : Any> {
    val emptyValue: C
    val emptyAddition: A
    val emptySubtraction: S

    fun add(left: C, right: A): C
    fun subtract(left: C, right: S): C
}

fun <T : Any> normalize(source: Provider<T>, empty: T): Provider<T> =
    source.orElse(providerOf(empty))
```

`C` is the property value, `A` is an addition operand, and `S` is a
subtraction operand. Each algebra implementation supplies the behavior from
the table in section 2.

Normalization locally interprets a missing collection contribution as the
appropriate empty identity. Missing does not become globally equivalent to
empty; raw `map`, `flatMap`, and `zip` remain missing-preserving.

### 3.2 `plus` and `minus`

These are the reference implementations. Public type-specific methods supply
their `CollectionAlgebra` implicitly.

A concrete right operand requires one dynamic input and uses `map`:

```kotlin
fun <C : Any, A : Any, S : Any> plus(
    source: Provider<C>,
    right: A,
    algebra: CollectionAlgebra<C, A, S>
): Provider<C> = normalize(source, algebra.emptyValue).map { left ->
    algebra.add(left, right)
}

fun <C : Any, A : Any, S : Any> minus(
    source: Provider<C>,
    right: S,
    algebra: CollectionAlgebra<C, A, S>
): Provider<C> = normalize(source, algebra.emptyValue).map { left ->
    algebra.subtract(left, right)
}
```

A provider right operand requires two dynamic inputs and uses `zip` after both
have been normalized:

```kotlin
fun <C : Any, A : Any, S : Any> plus(
    source: Provider<C>,
    right: Provider<A>,
    algebra: CollectionAlgebra<C, A, S>
): Provider<C> =
    normalize(source, algebra.emptyValue).zip(
        normalize(right, algebra.emptyAddition)
    ) { left, resolvedRight ->
        algebra.add(left, resolvedRight)
    }

fun <C : Any, A : Any, S : Any> minus(
    source: Provider<C>,
    right: Provider<S>,
    algebra: CollectionAlgebra<C, A, S>
): Provider<C> =
    normalize(source, algebra.emptyValue).zip(
        normalize(right, algebra.emptySubtraction)
    ) { left, resolvedRight ->
        algebra.subtract(left, resolvedRight)
    }
```

Constructing any of these results remains lazy and does not mutate `source`.
The provider overloads retain dependencies and provenance from both original
inputs.

### 3.3 Missing results

The reference implementations produce these results:

| Left result | Right result | `+` | `-` |
|---|---|---|---|
| `A` | `B` | `A + B` | `A - B` |
| `A` | missing | `A` | `A` |
| missing | `B` | `B` | `empty` |
| missing | missing | `empty` | `empty` |

An input currently resolved through its empty fallback remains an input. If it
later becomes present before finalization, its value participates normally.

### 3.4 Element providers

A provider of one element is lifted to a provider of zero or one elements:

```kotlin
element.map(::listOf).orElse(providerOf(emptyList()))
```

A missing element provider therefore contributes no element. It does not need
a fictitious empty value of type `E`.

## 4. Property selection and convention

Collection operations do not have a separate convention rule. The property
first selects its explicit or convention plan according to the foundational
specification; arithmetic then normalizes the selected result.

For a concrete operand `R`:

```text
Configured(V), convention C       + R = V + R
Configured(V), convention C       - R = V - R

Unconfigured, convention C        + R = C + R
Unconfigured, convention C        - R = C - R

Unconfigured, missing convention  + R = R
Unconfigured, missing convention  - R = empty

Configured(Missing), convention C + R = R
Configured(Missing), convention C - R = empty
```

The last case does not fall through to `C`. The explicit missing plan remains
selected; this operation then normalizes its result to empty.

An explicit empty collection is present and shadows the convention. A
deprecated `set(null)` overload has the same semantics as `setMissing()` and
therefore follows the configured-missing case.

Null is not a valid arithmetic operand, element, map key, or map value.

## 5. Compound assignment

Compound assignment is structural previous-version assignment:

```text
P += R == P.set(P.plus(R))
P -= R == P.set(P.minus(R))
```

The canonical implementations delegate to the non-mutating operations:

```kotlin
fun <C : Any, A : Any, S : Any> plusAssign(
    target: Property<C>,
    right: A,
    algebra: CollectionAlgebra<C, A, S>
) {
    target.set(plus(target, right, algebra))
}

fun <C : Any, A : Any, S : Any> plusAssign(
    target: Property<C>,
    right: Provider<A>,
    algebra: CollectionAlgebra<C, A, S>
) {
    target.set(plus(target, right, algebra))
}

fun <C : Any, A : Any, S : Any> minusAssign(
    target: Property<C>,
    right: S,
    algebra: CollectionAlgebra<C, A, S>
) {
    target.set(minus(target, right, algebra))
}

fun <C : Any, A : Any, S : Any> minusAssign(
    target: Property<C>,
    right: Provider<S>,
    algebra: CollectionAlgebra<C, A, S>
) {
    target.set(minus(target, right, algebra))
}
```

The setter replaces each structural `PropertyRead(P)` on the right-hand side
with the previous version of `P`. It does not realize a value during mutation.
Direct `plusAssign` and `minusAssign` methods may avoid constructing a temporary
expression but must have identical semantics.

If `P` is already configured, the operation uses its previous explicit plan.
A previous missing plan is normalized to empty, and the convention remains
shadowed.

If `P` is unconfigured, the previous version is a live `ConventionRead(P)`:

```kotlin
p.plusAssign(r)
p.convention(c)
```

has the same result as:

```kotlin
p.convention(c)
p.plusAssign(r)
```

Both produce `c + r`. A later convention replacement remains observable
through this convention-rooted explicit plan. This is an explicit dependency
inside the plan, not fallback from a configured missing result.

The first compound assignment permanently changes `P` from `Unconfigured` to
`Configured`. There is no `unset` operation. A later non-self assignment such
as `P.set(Q.map(f))` or `P.setMissing()` cuts the previous chain without
revealing the convention.

Sequential compound operations retain order:

```text
P += A
P -= B

P1 = plus(previous(P), A)
P2 = minus(P1, B)
current(P) = P2
```

An implementation may compact this chain only when it preserves operation
order, dependencies, missingness, failures, and diagnostics.

## 6. Configurable file collections

A configurable file collection may use the same algebra over an
insertion-ordered set of normalized file identities:

```text
files + notation = union after lazy notation expansion
files - notation = difference after lazy notation expansion
```

File operations must:

- expand file notation lazily;
- preserve dependencies from both operands;
- avoid checking existence while constructing the provider graph;
- normalize paths using the operand's declared base directory;
- not resolve symlinks merely to establish identity;
- preserve left order and append new right identities in right order;
- subtract only after both operands are expanded;
- treat a missing contribution as an empty file set.

Directory notation denotes the directory itself. Recursive membership is
introduced only by an explicit file-tree operand; subtraction must not silently
walk a directory.

If file collections adopt the foundational binding model, their first
`setFrom`, `from`, or `remove` operation permanently configures them. `from`
and `remove` use previous-version semantics; `setFrom` is non-self replacement.

## 7. Lifecycle and implementation requirements

Derived operations obey the foundational lifecycle rules:

- constructing `plus` or `minus` does not mutate or finalize a source;
- compound assignment fails when changes are disallowed or the property is
  finalized;
- finalization fixes the selected derived plan according to the lifecycle
  contract;
- failures identify the property, operation, and failing operand when
  available.

Collection values must be immutable in effect. Implementations should copy
mutable inputs at API boundaries and preserve provider-DAG sharing.

Repeated list concatenation must not cause quadratic copying. An evaluator may
use a builder, collector, rope, or ordered update node and construct the final
collection once. Compaction must preserve ordering and provenance.

## 8. Minimum conformance examples

```kotlin
// Collection algebra.
assertValue(
    providerOf(listOf("a", "a", "b")) - listOf("b"),
    listOf("a", "a")
)
assertValue(
    providerOf(linkedSetOf("a", "b")) + linkedSetOf("b", "c"),
    linkedSetOf("a", "b", "c")
)
assertValue(
    providerOf(linkedMapOf("a" to 1, "b" to 2)) +
        linkedMapOf("b" to 3, "c" to 4),
    linkedMapOf("a" to 1, "b" to 3, "c" to 4)
)

// Missing contributions use the empty identity.
assertValue(providerOf(listOf("a")) + Provider.missing(), listOf("a"))
assertValue(Provider.missing<List<String>>() + listOf("b"), listOf("b"))

// Explicit missing shadows convention before arithmetic normalizes it.
val p = listProperty<String>()
p.convention(listOf("default"))
p.setMissing()
assertMissing(p)
assertValue(p + listOf("user"), listOf("user"))

// An unconfigured compound assignment retains a live convention root.
val q = listProperty<String>()
q += listOf("user")
q.convention(listOf("default"))
assertValue(q, listOf("default", "user"))

// Non-self replacement cuts that chain; the convention does not reappear.
q.setMissing()
assertMissing(q)
```

File collection conformance must additionally verify lazy notation expansion,
stable identity ordering, dependency propagation, and no implicit directory
traversal.

## 9. Summary

```text
plus/minus are derived from orElse + map/zip
concrete operands use map
provider operands normalize both sides and use zip
ListProperty concatenates and removes matching values
SetProperty uses insertion-ordered union and difference
MapProperty uses right-biased merge and key subtraction
missing becomes empty only inside collection arithmetic
plus/minus do not mutate their source
plusAssign/minusAssign use structural previous-version assignment
an unconfigured compound assignment retains a live convention root
non-self set cuts the chain without revealing the convention
```
