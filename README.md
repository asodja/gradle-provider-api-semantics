# Arithmetic Manifest for Gradle Property Values

## Status and intent

This document specifies how arithmetic-like operations **should** behave for a
new Gradle property implementation. It is a semantic design, not a description
of Gradle's current implementation and not a compatibility promise.

The goal is that a programmer can predict an operation from three facts:

1. which value the property selects,
2. which pure collection operation is applied to that value, and
3. whether the expression only derives a value or also rebinds the property.

The words **must**, **must not**, **should**, and **may** are normative.

## Contents

1. Values, bindings, and notation
2. The operation set
3. Convention
4. `map`: the base derivation
5. Collection addition and subtraction
6. Provider operands and `zip`
7. Mutation and compound arithmetic
8. Self-reference
9. Configurable file collections
10. Lifecycle, observation, and failures
11. A clean implementation model
12. Minimum conformance examples
13. Design summary

## 1. Values, bindings, and notation

A property is a named, writable binding that is also readable as a provider:

```text
Property<T> = explicit binding + convention binding + lifecycle state
Provider<T> = a lazy, read-only computation that is either present or missing
```

Property values are non-null. Missing is a state; it is not a value.

This document uses:

| Symbol | Meaning |
|---|---|
| `P` | a property |
| `V` | a present explicit value |
| `C` | a present convention value |
| `R` | the right-hand operand appropriate to an operation |
| `empty` | a present, empty collection |
| `missing` | no value can be produced |
| `unset` | no binding exists in that slot |
| `A ⊕ B` | collection addition defined for the collection kind |
| `A ⊖ B` | collection subtraction defined for the collection kind |

`unset`, `missing`, `null`, and `empty` are deliberately different:

- `unset` describes a property slot.
- `missing` describes a provider result.
- `null` is not a legal property value.
- `empty` is a normal, present collection value.

## 2. The operation set

The small core is more important than a large set of operators. Every public
operation belongs to one of these groups:

| Group | Operations | Purpose |
|---|---|---|
| Derivation | `map`, `flatMap`, `zip`, `orElse` | Create or select a lazy, read-only provider |
| Collection derivation | `plus`, `minus` | Apply collection algebra through `map`, or `zip` for a provider operand |
| Binding | `set`, `unset`, `convention`, `unsetConvention` | Change a property's inputs |
| Atomic update | `update`, `updateOrElse`, `plusAssign`, `minusAssign` | Rebind from the property's previous plan |
| Observation | `isPresent`, `get`, `getOrNull` | Realize or inspect the selected value |
| Lifecycle | `finalizeValue`, `disallowChanges` | End or restrict mutation |

Operator spelling is optional DSL sugar:

```text
P + R   == P.plus(R)        // derives; does not change P
P - R   == P.minus(R)       // derives; does not change P
P += R  == P.plusAssign(R)  // atomically rebinds P
P -= R  == P.minusAssign(R) // atomically rebinds P
```

An API that cannot make the difference between `+` and `+=` obvious should use
the named methods instead.

Collection operators belong only to types with a declared collection algebra.
A scalar `Property<T>` still has `map`, `flatMap`, `zip`, and `update`, but
does not acquire numeric or string arithmetic merely because `T` happens to
offer similarly named methods. A domain-specific scalar operation can always
be written explicitly with `map` or `zip`.

## 3. Convention

A convention is a low-priority binding for a property. It is selected only
when the explicit slot is unset.

```text
selectedPlan(P) =
    explicit plan,   when the explicit slot is set
    convention plan, when the explicit slot is unset and convention is set
    missing plan,    otherwise
```

Selection happens before evaluation. Consequently, an explicit provider that
evaluates to `missing` does **not** fall through to the convention. It was still
the selected binding. This prevents a value from changing sources merely
because an upstream computation temporarily has no result.

A convention:

- is a fallback, not an initial collection element;
- can be a constant or a provider;
- remains lazy when supplied by a provider;
- is ignored while an explicit binding exists;
- is not copied into a derived provider as a second fallback.

The canonical way to remove an explicit binding is `unset()`. An implementation
may omit nullable overloads where the host language permits it. If `set(null)`
reaches an interoperability boundary, however, it must mean exactly `unset()`;
`null` never enters the provider graph. Likewise, `convention(null)` must mean
`unsetConvention()`. The named operations are preferred in new code.

## 4. `map`: the base derivation

`map` creates a provider by transforming the selected value of its source:

```text
map(P, f) =
    present(f(value)), when P produces a present value
    missing,           when P produces missing
```

In API-shaped pseudocode:

```kotlin
interface Provider<out T : Any> {
    fun <R : Any> map(transform: (T) -> R): Provider<R>
}
```

`map` must have these properties:

- **Lazy:** creating the mapped provider does not realize the source.
- **Read-only:** it does not mutate or rebind the source.
- **Missing-preserving:** the transform is not called when the source is
  missing.
- **Non-null:** a transform returning `null` is a programmer error, reported at
  the transform boundary. It is not silently converted to missing.
- **Immutable in effect:** the transform must produce a new collection value;
  it must not mutate the source value.
- **Provenance-preserving:** dependencies, producer information, and useful
  diagnostics flow from source to result.
- **Pure:** transforms must be safe to evaluate lazily, cache, or re-evaluate.

The algebraic laws are observational laws:

```text
P.map(identity)             == P
P.map(f).map(g)             == P.map { g(f(it)) }
missing.map(f)              == missing
```

The implementation may fuse maps when doing so preserves diagnostics and
provenance.

`map` has no special convention rule. A property selects its explicit or
convention plan first; `map` then sees the resulting value.

`flatMap` is the corresponding operation when the transform selects another
provider. It propagates a missing source without invoking the transform and
then returns the selected provider's result. `zip` is the symmetric
two-input operation described in section 6.

`orElse` is the explicit missing-value policy:

```text
P.orElse(Q) = P's value when P is present, otherwise Q's result
```

Unlike a convention, `orElse` is evaluated after a selected provider reports
missing. Collection arithmetic uses this operation internally to interpret a
missing collection contribution as the collection's empty identity. Ordinary
`map`, `flatMap`, and `zip` remain missing-preserving.

## 5. Collection addition and subtraction

For a concrete right-hand operand, `plus` and `minus` first totalize a missing
source to the collection's empty identity, then use `map`. Addition and
subtraction may have different operand types; for example, a map adds entries
but subtracts keys:

```kotlin
interface CollectionArithmetic<C : Any, A : Any, S : Any> {
    val emptyValue: C
    val emptyAddition: A
    val emptySubtraction: S

    fun plus(left: C, addition: A): C
    fun minus(left: C, subtraction: S): C
}

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

This has three immediate consequences:

1. `P + R` and `P - R` never modify `P`.
2. A missing collection operand contributes its empty identity.
3. The convention is naturally included when it is the selected source.

A missing value is not generally the same thing as an empty value. The
conversion is local to collection arithmetic because an additive collection
operation should never make prior contributions disappear. Code that needs
strict missing propagation uses raw `map` or `zip` instead.

This gives addition a programmer-visible monotonicity guarantee: appending to a
list cannot reduce its length, unioning a set or file collection cannot reduce
its membership, and merging a map cannot reduce its key set. An absent provider
is therefore a zero-element contribution, not a veto over contributions made
before or after it.

### 5.1 Algebra by collection kind

The operator consumes the right operand shown in the table. Where both singular
and bulk operands make sense, APIs should also offer unambiguous `plusElement`,
`plusAll`, `minusElement`, and `minusAll` methods. Singular methods have the
same behavior as a singleton bulk operand. Named methods avoid overload
surprises when an element is itself a collection.

| Property kind | Value model | `A ⊕ B` | `A ⊖ B` |
|---|---|---|---|
| `ListProperty<E>` | ordered sequence, duplicates retained | concatenate `B` after `A` | remove every element of `A` equal to any element in `B`; preserve survivor order |
| `SetProperty<E>` | insertion-ordered unique elements | union; values from `A` keep their positions, new values from `B` follow | set difference; preserve survivor order |
| `MapProperty<K,V>` | insertion-ordered unique keys | right-biased merge; `B` replaces values for duplicate keys without moving their key position | remove the keys supplied by `B` |
| `ConfigurableFileCollection` | insertion-ordered unique file identities | union after lazy file-notation expansion | file-identity difference after lazy expansion |

For `MapProperty`, subtraction takes keys, not key-value pairs. A differently
named filtering operation should be used for conditional entry removal.

Equality is the element or key type's declared equality. File identity is
defined separately in section 9.

These laws hold wherever the collection kind admits them:

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

Set and file addition are also idempotent. List concatenation and right-biased
map merge are not commutative. Subtraction is not a general inverse of addition,
so an implementation must never optimize `(A ⊕ B) ⊖ B` to `A`.

### 5.2 Selection and convention table

This table is the complete rule for non-mutating collection arithmetic with a
concrete right operand `R`:

| `P` explicit value | `P` convention | Arithmetic starts with | `P + R` | `P - R` |
|---|---|---|---|---|
| `V` | `C` | `V` | `V ⊕ R` | `V ⊖ R` |
| `V` | unset | `V` | `V ⊕ R` | `V ⊖ R` |
| unset | `C` | `C` | `C ⊕ R` | `C ⊖ R` |
| unset by `set(null)` | `C` | `C` | `C ⊕ R` | `C ⊖ R` |
| unset | unset | missing | `R` | `empty` |
| explicit provider resolves missing | `C` | missing | `R` | `empty` |
| `empty` | `C` | `empty` | `R` | `empty` |

With a provider-valued right operand, resolve both providers normally, then use
`empty` wherever a provider produces missing:

| Left produces | Right produces | Result of `+` | Result of `-` |
|---|---|---|---|
| `A` | `B` | `A ⊕ B` | `A ⊖ B` |
| `A` | missing | `A` | `A` |
| missing | `B` | `B` | `empty` |
| missing | missing | `empty` | `empty` |

A convention never substitutes for an explicit provider that resolves
missing. The explicit binding remains selected; collection arithmetic then
interprets its missing contribution as `empty`.

Null is not a right operand. `P + null`, `P - null`, and null collection
elements must be rejected at the API boundary with an error naming the
property and operation.

### 5.3 Concrete examples

The two tables above are the complete state rules. They apply equally to lists,
sets, maps, and file collections. The collection kind only determines what
`⊕`, `⊖`, and `empty` mean.

```text
Lists
  [a, a, b] + [b, c] = [a, a, b, b, c]
  [a, a, b] - [b, c] = [a, a]
  [a, a, b] + missing = [a, a, b]
  missing     + [b, c] = [b, c]

Sets
  {a, b} + {b, c} = {a, b, c}
  {a, b} - {b, c} = {a}
  {a, b} + missing = {a, b}
  missing + {b, c} = {b, c}
```

Convention follows the same rule without a separate matrix:

```text
P.convention([a])              P + [b] = [a, b]
P.set(null)                    P + [b] = [a, b]
P.set(missingProvider)         P + [b] = [b]
```

The last line does not fall through to the convention. The explicit provider
is selected, resolves to missing, and then contributes the empty identity.

`plusElement` and `minusElement` lift an element to a singleton collection. A
missing element provider is lifted to an empty collection, so it contributes
nothing. The separate `Element` and `All` names avoid ambiguity between
`Provider<E>` and `Provider<Iterable<E>>`.

## 6. Provider operands use normalized `zip`

`map` has one dynamic input. When the right operand is also a provider, the
operation has two dynamic inputs and must say so:

```kotlin
interface Provider<out T : Any> {
    fun <U : Any, R : Any> zip(
        other: Provider<U>,
        transform: (T, U) -> R
    ): Provider<R>
}

fun <C : Any, A : Any, S : Any> plus(
    source: Provider<C>,
    right: Provider<A>,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> =
    source.orElse(arithmetic.emptyValue).zip(
        right.orElse(arithmetic.emptyAddition)
    ) { left, resolvedRight ->
        arithmetic.plus(left, resolvedRight)
    }

fun <C : Any, A : Any, S : Any> minus(
    source: Provider<C>,
    right: Provider<S>,
    arithmetic: CollectionArithmetic<C, A, S>
): Provider<C> =
    source.orElse(arithmetic.emptyValue).zip(
        right.orElse(arithmetic.emptySubtraction)
    ) { left, resolvedRight ->
        arithmetic.minus(left, resolvedRight)
    }
```

Raw `zip` remains lazy and missing-preserving: it produces missing if either
input is missing. Collection arithmetic supplies `orElse` identities before
zipping, so both inputs to this particular `zip` are present. The derived
provider carries the provenance and dependencies of both original inputs,
including an input currently resolved through its empty fallback. Calling this
two-source operation `map` would conceal a dependency and should be avoided.

The fallback does not discard or snapshot a currently missing provider. If that
provider later becomes present before finalization, its value participates in
the derived collection normally.

A provider of one element is first lifted to a provider of zero or one
elements:

```kotlin
val additions: Provider<List<E>> =
    element.map(::listOf).orElse(emptyList())
```

This makes a missing element provider contribute no elements without requiring
a fictitious identity value of type `E`.

## 7. Mutation and compound arithmetic

`set` replaces the explicit binding. In contrast, `update` atomically creates a
new explicit binding from the property's **previous selected plan**:

```kotlin
interface Property<T : Any> : Provider<T> {
    fun set(value: T?) // interoperability form; null delegates to unset()
    fun set(provider: Provider<T>)
    fun unset()
    fun convention(value: T?) // null delegates to unsetConvention()
    fun convention(provider: Provider<T>)
    fun unsetConvention()

    fun update(transform: (T) -> T)
    fun updateOrElse(seed: T, transform: (T) -> T)
}
```

Collection compound operations are derived updates:

```text
P += R  == P.updateOrElse(empty) { previous -> previous ⊕ R }
P -= R  == P.updateOrElse(empty) { previous -> previous ⊖ R }
```

These equations use concrete operands. With a provider-valued operand, the
atomic update installs:

```text
Zip(
    OrElse(snapshot(previous), emptyValue),
    OrElse(right, emptyOperand),
    operation
)
```

The identity and provenance rules from section 6 still apply.

The update captures an immutable copy of the previous **plan**, not its realized
value. It therefore remains lazy. A concrete update installs a `Map`; a
provider-valued compound update installs a `Zip`. In either case, the captured
left source cannot point back to the mutable property cell.

Plain `update` remains missing-preserving. `updateOrElse` is the explicit
initializing form, and collection compound arithmetic always uses it with the
collection identity. Consequently, `P += R` initializes a missing `P` with
`R`, while `P -= R` initializes a missing `P` with `empty`.

### 7.1 Compound operation and convention table

| Before operation | Operation | Explicit binding afterwards | Observed result |
|---|---|---|---|
| explicit `V`, convention `C` | `P += R` | `map(snapshot(V), ⊕ R)` | `V ⊕ R` |
| explicit `V`, convention `C` | `P -= R` | `map(snapshot(V), ⊖ R)` | `V ⊖ R` |
| unset, convention `C` | `P += R` | `map(snapshot(C), ⊕ R)` | `C ⊕ R` |
| unset, convention `C` | `P -= R` | `map(snapshot(C), ⊖ R)` | `C ⊖ R` |
| unset, no convention | `P += R` | `map(empty, ⊕ R)` | `R` |
| unset, no convention | `P -= R` | `map(empty, ⊖ R)` | `empty` |
| explicit provider missing, convention `C` | `P += R` | `map(orElse(snapshot(missing), empty), ⊕ R)` | `R` |
| explicit provider missing, convention `C` | `P -= R` | `map(orElse(snapshot(missing), empty), ⊖ R)` | `empty` |
| unset by `set(null)`, convention `C` | `P += R` | `map(snapshot(C), ⊕ R)` | `C ⊕ R` |

Replacing the convention after a compound update does not rewrite that update:

```kotlin
p.convention(C)
p.plusAssign(R)    // captures the plan for C; explicit is now set
p.convention(D)    // D is shadowed; result is still C ⊕ R
p.unset()          // explicit update is removed; result is now D
```

If `C` is itself a live provider, updates to that provider remain observable.
Changing `P`'s convention slot is different: the update captured the slot's old
plan, not the mutable slot.

## 8. Self-reference

There are two very different cases.

### 8.1 Safe derived references

It is valid for another property to depend on `P`:

```kotlin
val withDefaults: Provider<List<String>> = p.plus(defaults)
q.set(withDefaults)
```

The provider remains lazy and observes the selected value of `P` when realized.

### 8.2 Rebinding a property from itself

This expression creates a cycle and must be rejected:

```kotlin
p.set(p.map { it + items }) // invalid
p.set(p + items)            // invalid for the same reason
```

The supported spelling is an atomic update:

```kotlin
p.updateOrElse(emptyList()) { it + items }
p += items
```

`update` means “derive from the plan bound immediately before this operation.”
It never means “read whatever this same property cell points to later.”
Only that implicit left-hand read is replaced by the snapshot. If an explicit
provider-valued right operand reads `P`, it is still a self-reference and must
be rejected as a cycle.

Direct cycles, indirect cycles, and convention cycles must be diagnosed with a
path that a programmer can act on:

```text
Cannot bind ':compile.inputs' to this provider: dependency cycle detected
  :compile.inputs -> mapped provider at build.gradle.kts:41
  mapped provider -> :defaults
  :defaults -> :compile.inputs
Use updateOrElse/plusAssign for an intentional collection update.
```

The implementation must not guess a fixed point, recurse until overflow, or
silently capture an already-realized value. A recurrence belongs in an explicit
state/fold abstraction, not in a property graph.

Sequential updates are ordered and use successive plans:

```text
P += A
P -= B

final plan = map(map(orElse(snapshot(original P), empty), ⊕ A), ⊖ B)
```

An implementation may compact this chain only when the observable ordering,
errors, provenance, and collection semantics remain unchanged.

## 9. Configurable file collections

A configurable file collection should implement the same collection-property
contract over a `FileSet` value. Its special behavior belongs in file-notation
resolution, not in the arithmetic operators.

File operations must:

- expand file notation lazily;
- avoid checking file existence while the operation graph is constructed;
- preserve task/build dependencies from both operands;
- normalize a path to an absolute, syntactically normalized path using the
  operand's declared base directory;
- not resolve symlinks or require the path to exist merely to establish
  identity;
- deduplicate by normalized file identity;
- preserve the left operand's order, then append new right-side identities in
  right-side order;
- perform subtraction after both sides have been expanded to identities.
- treat a missing file operand as an empty `FileSet`, so an absent contribution
  never discards files contributed elsewhere.

Adding or subtracting a directory notation operates on the identity produced
by that notation. Recursive membership exists only when the operand explicitly
denotes a file tree. This prevents `minus(directory)` from acquiring a hidden
and expensive “walk the filesystem” meaning.

Because subtraction needs the resolved exclusion set, its result depends on
both operands even when the right side ultimately removes no files.

The mutation vocabulary should be regular:

```text
files.setFrom(x)  == replacement binding
files.from(x)     == plusAssign(x)
files.remove(x)   == minusAssign(x)
files + x         == read-only derived file provider
files - x         == read-only derived file provider
```

Names retained for source compatibility should delegate to these semantics,
not introduce a second arithmetic model.

## 10. Lifecycle, observation, and failures

Arithmetic does not bypass property lifecycle rules.

- Updating a property after `disallowChanges()` must fail at the update call.
- Updating a finalized property must fail at the update call.
- Finalization realizes and fixes the selected plan according to the lifecycle
  contract; later changes in upstream providers are then not observed.
- Merely creating `P + R`, `P - R`, or `P.map(f)` does not mutate or finalize
  `P`.
- Realizing a derived provider may finalize its source only when the source's
  explicitly selected lifecycle policy requires that behavior.

Failures should identify the property path, operation, source location when
available, and failing operand. Important distinct diagnostics are:

- a missing value observed by strict `get`, `map`, or raw `zip`;
- null operand, element, or transform result;
- transform failure;
- mutation after lifecycle closure;
- direct or indirect dependency cycle;
- ambiguous element-versus-collection overload.

## 11. A clean implementation model

The implementation can be small if it separates immutable computations from
mutable bindings.

```text
                              +------------------+
explicit slot -------------->|                  |
                              | selection node   |----> effective Provider<T>
convention slot ------------>|                  |
                              +------------------+
                                       |
                                       v
                         OrElse / Map / Zip / file expansion
                              immutable provider DAG
```

### 11.1 Immutable provider nodes

Use a typed directed acyclic graph with nodes such as:

```text
Missing
Constant(value)
OrElse(primary, fallback, origin)
Map(source, transform, origin)
Zip(left, right, transform, origin)
FileExpansion(notation, baseDirectory, origin)
Fixed(value)                         // after finalization, when applicable
```

Each node carries provenance and dependency metadata. Node evaluation returns a
tagged result such as `Present(value)` or `Missing(reason)`; it never uses null
as a control signal.

### 11.2 Mutable property cell

A property cell contains:

```text
explicitPlan:   unset | ProviderNode<T>
conventionPlan: unset | ProviderNode<T>
lifecycle:      mutable | changesDisallowed | finalized
revision:       monotonically increasing identifier
```

Selection produces an immutable plan. Evaluation of that plan occurs outside
the mutation lock.

### 11.3 Atomic update algorithm

Conceptually:

```kotlin
fun update(transform: (T) -> T) = cell.mutateAtomically {
    requireMutable()
    val previous = snapshotSelectedPlan()
    explicitPlan = Map(previous, transform, callSite())
    revision += 1
}

fun updateOrElse(seed: T, transform: (T) -> T) = cell.mutateAtomically {
    requireMutable()
    val previous = OrElse(snapshotSelectedPlan(), Constant(seed), callSite())
    explicitPlan = Map(previous, transform, callSite())
    revision += 1
}
```

`snapshotSelectedPlan()` copies graph references from the current slots and
must not return a node that reads the same mutable cell. That invariant is what
makes self-updates safe without eager evaluation.

For a provider-valued collection addition, the same critical section installs
a `Zip` of the totalized previous plan and totalized right plan. No operand is
realized while the cell is locked.

All other binding operations and lifecycle transitions on one property cell
must be atomic. Provider evaluation may be concurrent, but a single evaluation
must observe one coherent revision.

### 11.4 Validation

Cycles should be rejected as early as the graph has enough information to find
them, then checked again during realization for graphs assembled indirectly.
Collection values should be defensively copied into persistent or immutable
representations at API boundaries.

## 12. Minimum conformance examples

Any implementation of this manifest should pass at least these examples:

```kotlin
// Convention participates before mapping.
val p = listProperty<String>().convention(listOf("a"))
assertValue(p + listOf("b"), listOf("a", "b"))
assertValue(p, listOf("a"))

// Explicit empty is present and shadows convention.
p.set(emptyList())
assertValue(p + listOf("b"), listOf("b"))

// Null clears the explicit slot at a compatibility boundary.
p.set(null)
assertValue(p, listOf("a"))

// An explicit missing provider does not fall through to the convention.
// The property remains missing, while collection arithmetic uses its identity.
p.set(Provider.missing())
assertMissing(p)
assertValue(p + listOf("b"), listOf("b"))

// Compound arithmetic uses the previous plan without a cycle.
p.unset()
p += listOf("b")
assertValue(p, listOf("a", "b"))

// Replacing the convention does not rewrite an existing explicit update.
p.convention(listOf("z"))
assertValue(p, listOf("a", "b"))
p.unset()
assertValue(p, listOf("z"))

// Generic self-binding is rejected with a cycle path.
assertCycle { p.set(p + listOf("x")) }

// A missing collection operand contributes the empty identity.
val absent = listProperty<String>()
assertValue(absent + listOf("x"), listOf("x"))
assertValue(absent - listOf("x"), emptyList())

// A provider operand is a two-source computation.
val right = providerOf(listOf("b"))
assertValue(providerOf(listOf("a")) + right, listOf("a", "b"))
assertValue(providerOf(listOf("a")) + Provider.missing(), listOf("a"))
assertValue(Provider.missing<List<String>>() + right, listOf("b"))
assertValue(
    Provider.missing<List<String>>() + Provider.missing(),
    emptyList()
)

// An absent additive contribution does not erase earlier or later additions.
val accumulated = listProperty<String>()
accumulated += listOf("foo")
accumulated += Provider.missing<List<String>>()
accumulated += listOf("baz")
assertValue(accumulated, listOf("foo", "baz"))

// Raw zip remains strict when strict missing propagation is required.
assertMissing(
    Provider.missing<List<String>>().zip(right) { a, b -> a + b }
)
```

File collection conformance must additionally verify lazy notation expansion,
stable ordering, normalized identity, dependency propagation from both sides,
and subtraction without filesystem traversal unless a file tree was requested.

## 13. Design summary

The complete model is:

```text
convention chooses a source only when explicit is unset
map transforms one selected source lazily
zip transforms two selected sources lazily
collection plus/minus normalize missing operands to the empty identity
plus/minus then use map for concrete operands and zip for provider operands
plusAssign/minusAssign are identity-seeded atomic updates
null is never a value
missing remains distinct from empty outside collection arithmetic
self-binding is a cycle; intentional previous-value changes use updateOrElse
file behavior is ordinary collection algebra after lazy notation expansion
```

This gives implementations room to optimize while leaving programmers with one
predictable semantic story.
