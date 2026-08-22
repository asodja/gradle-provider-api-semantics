# Derived Collection Operations

## Status and intent

This document specifies collection operations derived from the primitive
Provider API. It defines `plus` and `minus` for `ListProperty`, `SetProperty`,
and `MapProperty`, together with corresponding compound assignments.

The document depends on:

- [Provider API Foundations](PROVIDER_API_FOUNDATIONS.md) for selection,
  conventions, missing values, `map`, `zip`, and `orElse`;
- [Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md) for rebinding an
  expression such as `P.set(P.plus(R))` to the previous version of `P`.

It is a design for a new implementation, not a description of Gradle's current
behavior.

The term **derived** is intentional: these operations introduce collection
algebra, but they do not introduce another property-selection or convention
model.

The words **must**, **must not**, **should**, and **may** are normative.

## 1. Operation vocabulary

The collection API distinguishes derivation from mutation:

```text
P + R   == P.plus(R)        derive a provider; do not change P
P - R   == P.minus(R)       derive a provider; do not change P
P += R  == P.plusAssign(R)  atomically rebind P from its previous version
P -= R  == P.minusAssign(R) atomically rebind P from its previous version
```

Named overloads should make element and collection operands unambiguous:

```text
plusElement(element)
plusAll(elements)
minusElement(element)
minusAll(elements)
```

An element that is itself a collection must not be misinterpreted as a bulk
operand merely because of overload resolution.

Collection operators belong only to types with a declared collection algebra.
A scalar `Property<T>` does not acquire generic numeric or string arithmetic.
Domain-specific scalar operations can be written directly with `map` or `zip`.

## 2. Notation and missing-value policy

This document uses:

| Symbol | Meaning |
|---|---|
| `P` | A collection property or provider. |
| `R` | The right-hand operand appropriate to the operation. |
| `A ⊕ B` | Addition for the collection kind. |
| `A ⊖ B` | Subtraction for the collection kind. |
| `empty` | The present empty value for the collection kind. |
| missing | A provider produced no value. |

Collection arithmetic locally interprets a missing collection contribution as
the collection's empty identity:

```text
normalizeCollection(Missing)    = Present(empty)
normalizeCollection(Present(A)) = Present(A)
```

This does not globally equate missing with empty. Raw `map`, `flatMap`, and
`zip` remain missing-preserving. Collection arithmetic applies this explicit
normalization because an absent additive or subtractive contribution should
not erase contributions made elsewhere.

The resulting laws are:

```text
A ⊕ empty = A
empty ⊕ A = A
A ⊖ empty = A
empty ⊖ A = empty

A ⊕ missing = A
missing ⊕ A = A
A ⊖ missing = A
missing ⊖ A = empty
```

## 3. Collection algebra

### 3.1 Summary

| Property kind | Value model | Right operand for `+` | `A ⊕ B` | Right operand for `-` | `A ⊖ B` |
|---|---|---|---|---|---|
| `ListProperty<E>` | ordered sequence; duplicates retained | elements | concatenate `B` after `A` | elements | remove every value in `A` equal to any value in `B`; preserve survivor order |
| `SetProperty<E>` | insertion-ordered unique elements | elements | union; existing positions from `A` remain, new values from `B` follow | elements | set difference; preserve survivor order |
| `MapProperty<K,V>` | insertion-ordered unique keys | entries | right-biased merge by key | keys | remove keys supplied by `B` |

Equality is the declared element or key equality. Values and operands are
non-null.

### 3.2 `ListProperty`

List addition concatenates and retains duplicates:

```text
[a, a, b] + [b, c] = [a, a, b, b, c]
```

List subtraction treats the right operand as a set of values to remove and
removes every matching occurrence:

```text
[a, a, b, c] - [b, c] = [a, a]
```

The result preserves the order of surviving left-hand elements. The right-hand
operand's duplicate count does not affect subtraction.

Singular forms lift the element to a singleton list:

```text
P.plusElement(e)  == P.plusAll([e])
P.minusElement(e) == P.minusAll([e])
```

### 3.3 `SetProperty`

Set addition is insertion-ordered union:

```text
{a, b} + {b, c} = {a, b, c}
```

Values already present in the left set retain their position. New values follow
in right-operand iteration order.

Set subtraction is insertion-ordered difference:

```text
{a, b} - {b, c} = {a}
```

Set addition is idempotent but not specified as commutative with respect to
iteration order.

### 3.4 `MapProperty`

Map addition is a right-biased merge:

```text
{a: 1, b: 2} + {b: 3, c: 4} = {a: 1, b: 3, c: 4}
```

Replacing the value of an existing key does not move that key. New keys follow
in right-operand iteration order.

Map subtraction accepts keys:

```text
{a: 1, b: 2} - {b, c} = {a: 1}
```

Subtraction by entries would imply value-sensitive semantics and should use a
differently named filtering operation.

### 3.5 Algebraic cautions

Subtraction is not a general inverse of addition. An implementation must not
rewrite:

```text
(A ⊕ B) ⊖ B
```

to `A`. This is invalid for duplicate lists, overwritten map values, and other
ordered cases.

List concatenation and right-biased map merge are not commutative. Set union is
commutative by membership but not necessarily by iteration order.

## 4. Definition from primitive operations

Addition and subtraction may consume different operand types, so the algebra
declares both:

```kotlin
interface CollectionArithmetic<C : Any, A : Any, S : Any> {
    val emptyValue: C
    val emptyAddition: A
    val emptySubtraction: S

    fun plus(left: C, addition: A): C
    fun minus(left: C, subtraction: S): C
}
```

### 4.1 Concrete right operand uses `map`

```kotlin
fun <C : Any, A : Any, S : Any> plus(
    source: Provider<C>,
    right: A,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> = source.orElse(arithmetic.emptyValue).map { left ->
    arithmetic.plus(left, right)
}

fun <C : Any, A : Any, S : Any> minus(
    source: Provider<C>,
    right: S,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> = source.orElse(arithmetic.emptyValue).map { left ->
    arithmetic.minus(left, right)
}
```

Consequences:

1. `P + R` and `P - R` do not mutate `P`.
2. The property selects explicit or convention before arithmetic evaluates it.
3. A selected provider that evaluates missing is normalized to `empty`.
4. The result remains lazy and retains the source's dependencies.

### 4.2 Provider right operand uses normalized `zip`

When the right operand is also a provider, both inputs are normalized before a
strict `zip`:

```kotlin
fun <C : Any, A : Any, S : Any> plus(
    source: Provider<C>,
    right: Provider<A>,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> =
    source.orElse(arithmetic.emptyValue).zip(
        right.orElse(arithmetic.emptyAddition)
    ) { left, addition ->
        arithmetic.plus(left, addition)
    }

fun <C : Any, A : Any, S : Any> minus(
    source: Provider<C>,
    right: Provider<S>,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> =
    source.orElse(arithmetic.emptyValue).zip(
        right.orElse(arithmetic.emptySubtraction)
    ) { left, subtraction ->
        arithmetic.minus(left, subtraction)
    }
```

The derived provider carries provenance and dependencies from both original
inputs, including an input currently resolved through its empty fallback. A
currently missing provider is not snapshotted or discarded; if it later becomes
present before finalization, its value participates normally.

### 4.3 Element providers

A provider of one element is lifted to a provider of zero or one elements:

```kotlin
val additions: Provider<List<E>> =
    element.map(::listOf).orElse(emptyList())
```

A missing element provider therefore contributes no element. It does not
require a fictitious empty value of type `E`.

## 5. Selection, convention, and missing

### 5.1 Concrete operand

This is the complete rule for non-mutating arithmetic with a concrete right
operand `R`:

| Explicit state | Convention plan | Selected result before normalization | `P + R` | `P - R` |
|---|---|---|---|---|
| configured with `V` | `C` | `V` | `V ⊕ R` | `V ⊖ R` |
| configured with `V` | missing | `V` | `V ⊕ R` | `V ⊖ R` |
| unconfigured | `C` | `C` | `C ⊕ R` | `C ⊖ R` |
| unconfigured | missing | missing | `R` | `empty` |
| configured with missing provider | `C` | missing | `R` | `empty` |
| configured with `empty` | `C` | `empty` | `R` | `empty` |

An explicit missing provider does not fall through to the convention. It
remains the selected source, and this particular derived operation then
normalizes its missing result to the empty identity.

If a deprecated nullable setter is retained for compatibility, `set(null)` is
another spelling of an explicit missing binding. It follows the same row; it
does not return the property to `Unconfigured` or reveal the convention.

### 5.2 Provider operand

After property selection, both provider results follow one concise table:

| Left result | Right result | Result of `+` | Result of `-` |
|---|---|---|---|
| `A` | `B` | `A ⊕ B` | `A ⊖ B` |
| `A` | missing | `A` | `A` |
| missing | `B` | `B` | `empty` |
| missing | missing | `empty` | `empty` |

### 5.3 Null and empty

Null is not an arithmetic operand, element, key, or value:

```text
P + null
P - null
```

must fail at the API boundary with an error naming the property and operation.
This is separate from a deprecated nullable `set` overload, whose compatibility
semantics are defined by the foundational specification.

Empty is a present value. An explicit empty collection shadows a convention and
participates as the algebraic identity.

## 6. Compound assignment

Compound assignment is structurally self-referential assignment:

```text
P += R == P.set(P.plus(R))
P -= R == P.set(P.minus(R))
```

The setter substitutes the structural `PropertyRead(P)` with the previous
version of `P`, as defined by the self-reference specification. It does not
realize a value during mutation.

An implementation may expose direct operations that avoid constructing a
temporary expression:

```kotlin
fun plusAssign(right: Addition)
fun minusAssign(right: Subtraction)
```

These must be semantically identical to structural self-assignment.

### 6.1 Previous explicit binding

When `P` already has an explicit binding, compound assignment derives from that
binding and retains its live upstream providers:

```text
explicit V; P += R  => V ⊕ R
explicit V; P -= R  => V ⊖ R
```

If the explicit provider evaluates missing, collection normalization supplies
the empty identity. The convention remains shadowed:

```text
explicit missing; convention C; P += R => R
explicit missing; convention C; P -= R => empty
```

### 6.2 Convention-rooted assignment

When `P` is still unconfigured, the previous version is a live
`ConventionRead(P)`:

```kotlin
p.plusAssign(r)
p.convention(c)
```

has the same result as:

```kotlin
p.convention(c)
p.plusAssign(r)
```

Both produce `c ⊕ r`. This preserves convention priority even when a plugin's
lazy convention callback executes after a user's compound assignment.

A later convention replacement also remains observable through the derived
chain:

```kotlin
p.convention(c)
p.plusAssign(r) // explicit plan derived from ConventionRead(P)
p.convention(d)

// Result: d ⊕ r
```

A later non-self `set` replaces the complete explicit derived chain. It does
not reveal `d`:

```kotlin
p.set(x)

// Result: x
```

### 6.3 Compound-state table

| State before operation | Operation | Current result |
|---|---|---|
| explicit `V`, convention `C` | `P += R` | `V ⊕ R` |
| explicit `V`, convention `C` | `P -= R` | `V ⊖ R` |
| unconfigured, convention `C` | `P += R` | `C ⊕ R` |
| unconfigured, convention `C` | `P -= R` | `C ⊖ R` |
| unconfigured, missing convention | `P += R` | `R` |
| unconfigured, missing convention | `P -= R` | `empty` |
| explicit provider missing, convention `C` | `P += R` | `R` |
| explicit provider missing, convention `C` | `P -= R` | `empty` |

When the chain is convention-rooted, installing a convention later changes the
currently observed result from its empty-based result to the convention-based
result.

### 6.4 Sequential compound operations

Operations are ordered:

```text
P += A
P -= B

P2 = minus(P1, B)
P1 = plus(P0, A)
```

The provider DAG is the history. A non-self replacement such as
`P.set(Q.map(f))` cuts P's previous chain because the right-hand plan contains
no structural read of `P`.

An implementation may flatten or rebalance long chains but must preserve
operation order, dependencies, failures, and diagnostics.

## 7. Concrete examples

### 7.1 Lists

```text
[a, a, b] + [b, c] = [a, a, b, b, c]
[a, a, b] - [b, c] = [a, a]
[a, a, b] + missing = [a, a, b]
missing + [b, c] = [b, c]
```

### 7.2 Sets

```text
{a, b} + {b, c} = {a, b, c}
{a, b} - {b, c} = {a}
{a, b} + missing = {a, b}
missing + {b, c} = {b, c}
```

### 7.3 Maps

```text
{a: 1, b: 2} + {b: 3, c: 4} = {a: 1, b: 3, c: 4}
{a: 1, b: 2} - {b, c} = {a: 1}
{a: 1} + missing = {a: 1}
missing + {b: 2} = {b: 2}
```

### 7.4 Convention examples

```text
P.convention([a])      P + [b] = [a, b]
P.set(empty)           P + [b] = [b]
P.convention([z])      P + [b] = [b]
P.set(missingProvider) P + [b] = [b]
```

## 8. Configurable file collections

A configurable file collection may implement the same derived algebra over an
insertion-ordered set of normalized file identities:

```text
files + notation  union after lazy notation expansion
files - notation  difference after lazy notation expansion
```

File operations must:

- expand file notation lazily;
- preserve task and build dependencies from both operands;
- avoid checking file existence while constructing the provider graph;
- normalize paths using the operand's declared base directory;
- not resolve symlinks merely to establish identity;
- preserve left order and then append new right identities in right order;
- subtract only after both operands are expanded;
- treat a missing file contribution as an empty file set.

Directory notation denotes that directory's identity. Recursive membership is
introduced only by an explicit file-tree operand; `minus(directory)` must not
silently walk the filesystem.

A regular mutation vocabulary is:

```text
files.setFrom(x) == replacement
files.from(x)    == plusAssign(x)
files.remove(x)  == minusAssign(x)
files + x        == read-only derivation
files - x        == read-only derivation
```

## 9. Lifecycle, failures, and implementation

Derived operations obey the foundational lifecycle rules:

- constructing `plus` or `minus` does not mutate or finalize a source;
- compound assignment fails immediately after changes are disallowed or the
  property is finalized;
- finalization fixes the selected derived plan according to the lifecycle
  contract;
- failures identify the property, operation, source location, and failing
  operand when available.

Important distinct failures are:

- null operand, element, map key, or map value;
- ambiguous element-versus-collection overload;
- transform or equality failure;
- mutation after lifecycle closure;
- an opaque or cross-property cycle not eligible for structural substitution.

The implementation should use immutable provider nodes and defensively copy
mutable collections at API boundaries. Repeated list concatenation must not
produce quadratic copying: use a builder, collector, rope, or ordered update
node and build the final collection once per evaluation.

Dependency and provenance metadata should preserve DAG sharing rather than copy
full transitive sets at every link.

## 10. Minimum conformance examples

```kotlin
// List behavior.
assertValue(providerOf(listOf("a")) + listOf("b"), listOf("a", "b"))
assertValue(
    providerOf(listOf("a", "a", "b")) - listOf("b"),
    listOf("a", "a")
)

// Set behavior and stable order.
assertValue(
    providerOf(linkedSetOf("a", "b")) + linkedSetOf("b", "c"),
    linkedSetOf("a", "b", "c")
)

// Map right-biased merge and key subtraction.
assertValue(
    providerOf(linkedMapOf("a" to 1, "b" to 2)) +
        linkedMapOf("b" to 3, "c" to 4),
    linkedMapOf("a" to 1, "b" to 3, "c" to 4)
)
assertValue(
    providerOf(linkedMapOf("a" to 1, "b" to 2)) - setOf("b"),
    linkedMapOf("a" to 1)
)

// Convention participates through ordinary selection.
val p = listProperty<String>()
p.convention(listOf("a"))
assertValue(p + listOf("b"), listOf("a", "b"))

// Explicit empty shadows convention.
p.set(emptyList())
assertValue(p + listOf("b"), listOf("b"))

// Explicit missing does not fall through, but arithmetic normalizes it.
p.set(Provider.missing())
assertMissing(p)
assertValue(p + listOf("b"), listOf("b"))

// Missing provider operand contributes the empty identity.
assertValue(providerOf(listOf("a")) + Provider.missing(), listOf("a"))
assertValue(Provider.missing<List<String>>() + listOf("b"), listOf("b"))

// Compound assignment on an unconfigured property uses its convention root.
val q = listProperty<String>()
q.convention(listOf("a"))
q += listOf("b")
assertValue(q, listOf("a", "b"))

// A convention replacement remains live through a convention-rooted chain.
q.convention(listOf("z"))
assertValue(q, listOf("z", "b"))

// A non-self set cuts the chain without revealing the convention.
q.set(Provider.missing())
assertMissing(q)

// Missing contributions do not erase accumulated values.
val accumulated = listProperty<String>()
accumulated += listOf("foo")
accumulated += Provider.missing<List<String>>()
accumulated += listOf("baz")
assertValue(accumulated, listOf("foo", "baz"))

// Raw zip remains strict.
assertMissing(
    Provider.missing<List<String>>().zip(providerOf(listOf("b"))) { a, b ->
        a + b
    }
)
```

File collection conformance must additionally verify lazy notation expansion,
stable identity ordering, dependency propagation, and subtraction without an
implicit filesystem traversal.

## 11. Design summary

```text
plus/minus are derived from orElse + map/zip
concrete operands use map
provider operands normalize both sides and use zip
ListProperty uses concatenation and value-removal
SetProperty uses insertion-ordered union and difference
MapProperty uses right-biased merge and key subtraction
missing is normalized to empty only inside collection arithmetic
plus/minus never mutate their source
plusAssign/minusAssign use structural previous-version assignment
convention-rooted compound chains retain a live convention base
null is never an operand, element, key, or value
```

The property-selection and convention rules remain those of the foundational
Provider API. Derived collection operations add algebra, not another binding
model.
