# Derived Collection Operations

## Status and intent

This document defines collection operations built from the primitive Provider
API. It specifies `plus` and `minus` for `ListProperty`, `SetProperty`, and
`MapProperty`, together with their compound forms.

This version of the specification makes no concurrency guarantees for
`Property`. Its requirements apply when property access is not concurrent.
Thread safety, atomicity, visibility, and ordering between concurrent reads or
mutations remain unspecified until the `Property` concurrency contract is
defined.

It builds on:

- [Provider API Foundations](PROVIDER_API_FOUNDATIONS.md) for property
  selection, conventions, missing values, `map`, and `zip`;
- [Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md) for assigning an
  expression such as `P.set(P.plus(R))` back to `P`.

The term **derived** is intentional. These operations add collection algebra,
not another property or convention model.

The words **must**, **must not**, **should**, and **may** are normative.

## Contents

- [1. Vocabulary](#1-vocabulary)
- [2. Collection algebra](#2-collection-algebra)
- [3. Definition from primitive operations](#3-definition-from-primitive-operations)
- [4. Property selection and convention](#4-property-selection-and-convention)
- [5. Compound assignment](#5-compound-assignment)
- [6. Configurable file collections](#6-configurable-file-collections)
- [7. Lifecycle and implementation requirements](#7-lifecycle-and-implementation-requirements)
- [8. Minimum conformance examples](#8-minimum-conformance-examples)
- [9. Summary](#9-summary)

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

These laws apply to present empty collections. Missing is not an empty value.

Subtraction is not an inverse of addition. In particular, an implementation
must not rewrite `(A + B) - B` to `A`. Such a rewrite loses list duplicates
and overwritten map values.

## 3. Definition from primitive operations

### 3.1 Collection algebra interface

The base addition combines two collection values. Element and bulk operands
are converted to that value type before addition. Subtraction retains a
separate removal type because, for example, map subtraction accepts keys:

```kotlin
interface CollectionAlgebra<C : Any, E : Any, S : Any> {
    fun collectionOf(element: E): C
    fun collectionFrom(elements: Iterable<E>): C

    fun plus(left: C, right: C): C
    fun minus(left: C, removals: S): C
}
```

`C` is the property value, `E` is one addable element, and `S` is a
subtraction operand. For maps, `E` is an entry and `S` supplies keys. Each
implementation supplies the behavior from the table in section 2.

`collectionOf(e)` produces `[e]` for a list, `{e}` for a set, and a
single-entry map for a map entry. `collectionFrom` performs the corresponding
bulk conversion.

### 3.2 `plus` and `minus`

These are the reference implementations. Public type-specific methods supply
their `CollectionAlgebra` implicitly.

A concrete collection is first wrapped as a provider:

```kotlin
fun <C : Any, E : Any, S : Any> plus(
    source: Provider<C>,
    right: C,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> = plus(source, providerOf(right), algebra)
```

A provider-valued addition combines two present collections. Like `zip`, it is
missing when either input is missing:

```kotlin
fun <C : Any, E : Any, S : Any> plus(
    source: Provider<C>,
    right: Provider<C>,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> =
    source.zip(right, algebra::plus)
```

A concrete subtraction is likewise wrapped as a present provider:

```kotlin
fun <C : Any, E : Any, S : Any> minus(
    source: Provider<C>,
    right: S,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> = minus(source, providerOf(right), algebra)
```

A provider-valued subtraction combines two present inputs. Like `zip`, it is
missing when either input is missing:

```kotlin
fun <C : Any, E : Any, S : Any> minus(
    source: Provider<C>,
    right: Provider<S>,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> = source.zip(right, algebra::minus)
```

Constructing any of these results remains lazy and does not mutate `source`.
The provider overloads retain dependencies and provenance from both original
inputs.

### 3.3 Missing results

The reference implementations produce these results:

| Left result | Right result | `+` | `-` |
|---|---|---|---|
| `A` | `B` | `A + B` | `A - B` |
| `A` | missing | missing | missing |
| missing | `B` | missing | missing |
| missing | missing | missing | missing |

Both operations follow `zip` semantics and therefore require both inputs to be
present. Missing is not treated as an empty collection or as an instruction to
ignore that operand. All original providers remain inputs, so a provider which
becomes present before finalization participates normally.

### 3.4 Element and bulk operands

Element and bulk providers are converted before invoking the base `plus`:

```kotlin
fun <C : Any, E : Any, S : Any> plusElement(
    source: Provider<C>,
    element: Provider<E>,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> = plus(
    source,
    element.map(algebra::collectionOf),
    algebra
)

fun <C : Any, E : Any, S : Any> plusAll(
    source: Provider<C>,
    elements: Provider<Iterable<E>>,
    algebra: CollectionAlgebra<C, E, S>
): Provider<C> = plus(
    source,
    elements.map(algebra::collectionFrom),
    algebra
)
```

Concrete operands use the same definitions after `providerOf`. A missing
element or bulk provider remains missing through `map`, so the resulting
addition is missing regardless of whether the left collection is present.

## 4. Property selection and convention

Collection operations do not have a separate convention rule. The property
first selects its explicit or convention plan according to the foundational
specification; arithmetic then applies the missing rules from section 3.3.

For a concrete operand `R`:

```text
Configured(V), convention C       + R = V + R
Configured(V), convention C       - R = V - R

Unconfigured, convention C        + R = C + R
Unconfigured, convention C        - R = C - R

Unconfigured, missing convention  + R = missing
Unconfigured, missing convention  - R = missing

Configured(Missing), convention C + R = missing
Configured(Missing), convention C - R = missing
```

The last case does not fall through to `C`. The explicit missing plan remains
selected, and both operations remain missing because their left input is
missing.

An explicit empty collection is present and shadows the convention. A
deprecated `set(null)` overload has the same semantics as
`set(Provider.missing())` and therefore follows the configured-missing case.

Null is not a valid arithmetic operand, element, map key, or map value.

## 5. Compound assignment

Compound assignment is structural previous-version assignment:

```text
P += R == P.set(P.plus(R))
P -= R == P.set(P.minus(R))
```

The canonical implementations delegate to the non-mutating operations:

```kotlin
fun <C : Any, E : Any, S : Any> plusAssign(
    target: Property<C>,
    right: C,
    algebra: CollectionAlgebra<C, E, S>
) {
    target.set(plus(target, right, algebra))
}

fun <C : Any, E : Any, S : Any> plusAssign(
    target: Property<C>,
    right: Provider<C>,
    algebra: CollectionAlgebra<C, E, S>
) {
    target.set(plus(target, right, algebra))
}

fun <C : Any, E : Any, S : Any> minusAssign(
    target: Property<C>,
    right: S,
    algebra: CollectionAlgebra<C, E, S>
) {
    target.set(minus(target, right, algebra))
}

fun <C : Any, E : Any, S : Any> minusAssign(
    target: Property<C>,
    right: Provider<S>,
    algebra: CollectionAlgebra<C, E, S>
) {
    target.set(minus(target, right, algebra))
}
```

Element and bulk compound overloads use the same pattern with `plusElement`
and `plusAll` inside `target.set(...)`.

The setter replaces each structural `PropertyRead(P)` on the right-hand side
with the previous version of `P`. It does not realize a value during mutation.
Direct `plusAssign` and `minusAssign` methods may avoid constructing a temporary
expression but must have identical semantics.

If `P` is already configured, the operation uses its previous explicit plan.
A previous missing plan plus or minus any contribution remains missing. The
convention remains shadowed in all cases.

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
as `P.set(Q.map(f))` or `P.set(Provider.missing())` cuts the previous chain
without revealing the convention.

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
- apply the same missing rules as other collections: both operands must be
  present for addition or subtraction.

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

// Provider-valued operations follow zip and require both inputs.
assertMissing(providerOf(listOf("a")) + Provider.missing())
assertMissing(Provider.missing<List<String>>() + listOf("b"))
assertMissing(
    Provider.missing<List<String>>() + Provider.missing<List<String>>()
)

assertMissing(providerOf(listOf("a")) - Provider.missing())
assertMissing(Provider.missing<List<String>>() - listOf("b"))
assertMissing(
    Provider.missing<List<String>>() - Provider.missing<List<String>>()
)

// Explicit missing shadows convention and propagates through both operations.
val p = listProperty<String>()
p.convention(listOf("default"))
p.set(Provider.missing())
assertMissing(p)
assertMissing(p + listOf("user"))
assertMissing(p - listOf("user"))

// An unconfigured compound assignment retains a live convention root.
val q = listProperty<String>()
q += listOf("user")
q.convention(listOf("default"))
assertValue(q, listOf("default", "user"))

// Non-self replacement cuts that chain; the convention does not reappear.
q.set(Provider.missing())
assertMissing(q)
```

File collection conformance must additionally verify lazy notation expansion,
stable identity ordering, dependency propagation, and no implicit directory
traversal.

## 9. Summary

```text
provider-valued plus/minus are derived directly from zip
concrete operands are wrapped as present providers
provider operands use zip without converting missing to empty
element and bulk addition operands are lifted with map before zip
ListProperty concatenates and removes matching values
SetProperty uses insertion-ordered union and difference
MapProperty uses right-biased merge and key subtraction
addition and subtraction require both operands to be present
plus/minus do not mutate their source
plusAssign/minusAssign use structural previous-version assignment
an unconfigured compound assignment retains a live convention root
non-self set cuts the chain without revealing the convention
```
