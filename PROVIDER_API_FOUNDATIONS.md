# Provider API Foundations

## Status and intent

This document specifies the small semantic core of a proposed lazy Provider and
Property API. It describes how values are selected, how conventions interact
with explicit bindings, and how the primitive provider operations behave.

It is a design for a new implementation. It does not describe Gradle's current
implementation and is not a compatibility promise.

Operations built from this core, including collection `plus` and `minus`, are
specified in [Derived Collection Operations](DERIVED_COLLECTION_OPERATIONS.md).
Previous-version assignment is specified separately in
[Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md).

The words **must**, **must not**, **should**, and **may** are normative.

## 1. Provider and Property

A provider is a lazy, read-only computation:

```text
Provider<T> = a computation that produces Present<T> or Missing
```

A property is a mutable binding that is also readable as a provider:

```text
Property<T> = explicit binding + convention binding + lifecycle state
```

A property does not store `null`. Its provider result is either a present,
non-null value or missing.

```kotlin
interface Provider<out T : Any> {
    fun isPresent(): Boolean
    fun get(): T
    fun getOrNull(): T?

    fun <R : Any> map(transform: (T) -> R): Provider<R>
    fun <R : Any> flatMap(transform: (T) -> Provider<R>): Provider<R>
    fun <U : Any, R : Any> zip(
        other: Provider<U>,
        transform: (T, U) -> R
    ): Provider<R>

    fun orElse(fallback: Provider<@UnsafeVariance T>): Provider<T>
}

interface Property<T : Any> : Provider<T> {
    fun set(value: T)
    fun set(provider: Provider<T>)
    fun unset()

    fun convention(value: T)
    fun convention(provider: Provider<T>)
    fun unsetConvention()

    fun finalizeValue()
    fun disallowChanges()
}
```

Deprecated nullable overloads may temporarily exist at an interoperability
boundary. Their meaning is defined in section 5; null never becomes a property
value.

## 2. Values, slots, and selection

The following states are deliberately different:

| Term | Meaning |
|---|---|
| present | A provider produced a non-null value. |
| missing | A provider produced no value. |
| unset | A property slot has no binding. |
| null | An invalid value; legacy nullable setters may translate it to an explicit missing binding. |
| empty | A present collection containing no elements. |

A property has two binding slots:

```text
explicit slot:   unset | Provider<T>
convention slot: unset | Provider<T>
```

It selects a binding before evaluating that binding:

```text
selectedPlan(P) =
    explicit plan,   when the explicit slot is set
    convention plan, when the explicit slot is unset and convention is set
    missing plan,    otherwise
```

This selection rule is the foundation of all other operations.

## 3. Convention

### 3.1 Purpose

A convention is a low-priority binding, normally supplied by a plugin or
shared build logic. It expresses the plugin's initial value while allowing an
explicit user value to take precedence.

Conventions exist to solve an ordering problem created by lazy configuration.
Consider a plugin that creates a task with a `Property<String>`. The plugin
schedules a lazy task-configuration action which supplies an initial value:

```kotlin
// Registered by the plugin. This action may execute later.
task.configure {
    property.set("foo")
}
```

The user then makes an explicit choice:

```kotlin
task.property.set("bar")
```

If the plugin's configuration action executes afterwards, its ordinary
`set("foo")` replaces the user's `"bar"`. Configuration execution order has
accidentally become configuration priority.

The plugin should express its initial value as a convention instead:

```kotlin
// A plugin's lazy callback may execute later.
task.configure {
    property.convention("foo")
}

// The user's explicit choice remains selected.
task.property.set("bar")
```

The result is `"bar"` regardless of which call executes first. If the user
does not configure the property, the result is `"foo"`.

A convention is therefore not an imperative “set this value only if the
property happens to be empty right now” operation. It writes a separate,
lower-priority slot. The property continuously applies the priority rule
“explicit, otherwise convention,” independently of callback timing.

A convention:

- can be a constant or a provider;
- remains lazy when supplied by a provider;
- is retained while shadowed by an explicit binding;
- is replaceable while shadowed;
- is selected only when the explicit slot is unset;
- is not an `orElse` fallback for an explicit provider that evaluates missing.

The self-reference extension can install an explicit plan that deliberately
contains `ConventionRead(P)` because no explicit binding existed before that
assignment. In that case the convention is an input of the selected explicit
plan; the property has not fallen through from an explicit missing result. The
separate self-reference specification defines this extension.

### 3.2 Selection happens before evaluation

An explicit provider remains selected even when it produces missing:

```kotlin
property.convention("default")
property.set(Provider.missing())

property.isPresent() // false
```

The property does not fall through to `"default"`. The explicit configuration
selected `Provider.missing()` as its source of truth. Use `orElse` explicitly
when fallback-after-evaluation is intended:

```kotlin
property.set(source.orElse(providerOf("fallback")))
```

### 3.3 Binding-state table

`C` and `D` are conventions, `V` is an explicit value, and `Q` is an explicit
provider:

| Operations, in order | Selected binding | Observed result |
|---|---|---|
| none | missing plan | missing |
| `convention(C)` | convention `C` | result of `C` |
| `convention(C); set(V)` | explicit `V` | `V` |
| `set(V); convention(C)` | explicit `V` | `V` |
| `convention(C); set(Q)` | explicit `Q` | result of `Q` |
| `convention(C); set(missingProvider)` | explicit missing provider | missing |
| `convention(C); set(null)` (deprecated) | explicit missing plan | missing |
| previous row, then `unset()` | convention `C` | result of `C` |
| `convention(C); set(V); unset()` | convention `C` | result of `C` |
| `convention(C); set(V); convention(D)` | explicit `V` | `V` |
| previous row, then `unset()` | convention `D` | result of `D` |
| `convention(C); unsetConvention()` | missing plan | missing |

`set` changes only the explicit slot. `convention` changes only the convention
slot. Neither operation silently deletes the other slot.

## 4. Primitive provider operations

Primitive operations create lazy, read-only provider plans. They do not mutate
their inputs.

### 4.1 `map`

`map` transforms one present source value:

```text
map(P, f) =
    Present(f(value)), when P produces Present(value)
    Missing,           when P produces Missing
```

The transform is not called for a missing source.

`map` must be:

- lazy: construction does not evaluate the source;
- missing-preserving;
- non-mutating;
- non-null: a null transform result is an error at the transform boundary;
- provenance-preserving: producer and dependency information flows to the
  result;
- safe to evaluate according to the provider's caching and lifecycle policy.

The observational laws are:

```text
P.map(identity)             == P
P.map(f).map(g)             == P.map { g(f(it)) }
Missing.map(f)              == Missing
```

Map fusion is an implementation optimization, not a semantic difference. It
must preserve diagnostics, dependencies, missing reasons, and transform order.

### 4.2 `flatMap`

`flatMap` transforms a present value into another provider:

```text
flatMap(P, f) =
    result of f(value), when P produces Present(value)
    Missing,            when P produces Missing
```

The transform is not called for a missing source. The selected inner provider
remains lazy and contributes its dependencies and missing reason.

Because a `flatMap` transform may choose a provider dynamically, dependencies
introduced inside an opaque transform are not structurally inspectable for
self-reference substitution. See the separate self-reference specification.

### 4.3 `zip`

`zip` combines two present provider values symmetrically:

```text
zip(P, Q, f) =
    Present(f(a, b)), when P produces Present(a) and Q produces Present(b)
    Missing,          when either input produces Missing
```

| Left result | Right result | `zip` result |
|---|---|---|
| `Present(A)` | `Present(B)` | `Present(f(A, B))` |
| `Present(A)` | missing | missing |
| missing | `Present(B)` | missing |
| missing | missing | missing |

`zip` is lazy, non-mutating, and dependency-preserving for both inputs. A
two-input operation should use `zip` rather than pretending the right input is
captured by a one-input `map`.

### 4.4 `orElse`

`orElse` selects a fallback after evaluating the primary provider:

```text
P.orElse(Q) =
    P's value, when P is present
    Q's result, when P is missing
```

This is deliberately different from a convention:

```text
convention: choose a source because the explicit slot is unset
orElse:     choose a result because the primary source evaluated missing
```

`orElse` is the primitive operation for an explicit missing-value policy.

### 4.5 Operations observe the selected property binding

Provider operations applied to a property observe whichever binding the
property selects at evaluation time:

```kotlin
property.convention("a")
val upper = property.map(String::uppercase)

upper.get() // "A"

property.set("b")
upper.get() // "B"

property.unset()
upper.get() // "A"
```

The derived provider is not given a separate copy of the convention. It reads
the property's selected plan.

When the explicit provider is missing, primitive operations remain strict:

| Property state | `map` | `flatMap` | `zip(present)` | `orElse(F)` |
|---|---|---|---|---|
| explicit `V`, convention `C` | transform `V` | select from `V` | combine `V` | `V` |
| unset, convention `C` | transform `C` | select from `C` | combine `C` | `C` |
| unset, no convention | missing | missing | missing | result of `F` |
| explicit provider missing, convention `C` | missing | missing | missing | result of `F` |

## 5. Binding operations and null

The binding operations are:

```text
set(value/provider)          replace the explicit binding
unset()                      remove the explicit binding
convention(value/provider)   replace the convention binding
unsetConvention()            remove the convention binding
```

Named removal operations are canonical. They distinguish a missing provider
from an unset property slot.

New APIs should not accept null in binding operations. If legacy nullable
overloads must remain temporarily, they should be deprecated and translated at
the API boundary as follows:

```text
set(null)        == set(Provider.missing())
convention(null) == convention(Provider.missing())
```

`set(null)` is therefore an explicit choice which shadows a convention. Only
`unset()` removes that choice and reveals the convention again. Treating
`set(null)` as `unset()` is broken behavior because it changes the selected
source rather than representing the user's explicit assignment.

The compatibility boundary creates a `Missing` plan; null itself must not
enter the provider graph or become a present value. A transform, collection
element, map key, or map value that produces null is an error at its API
boundary.

An empty collection is an ordinary present value. It shadows a convention just
like any other explicit value.

## 6. Observation and missing values

Observation realizes a provider according to its lifecycle policy:

```text
isPresent()  reports whether evaluation is present
get()        returns a present value or fails with a missing-value diagnostic
getOrNull()  returns a present value or null as an observation result
```

`getOrNull()` does not make null a property value. It only represents missing
at the observation boundary.

A missing result should carry a reason or path. Diagnostics should distinguish:

- no explicit or convention binding;
- a selected provider that evaluated missing;
- a missing input to `map`, `flatMap`, or `zip`;
- a cycle;
- a transform failure.

## 7. Lifecycle

Lifecycle operations apply to the mutable property cell:

- `disallowChanges()` prevents later binding mutations;
- `finalizeValue()` fixes the selected value according to the lifecycle
  contract;
- binding operations attempted after lifecycle closure fail at the mutation;
- constructing `map`, `flatMap`, `zip`, or `orElse` does not mutate or finalize
  a source;
- finalization may replace a provider plan with a fixed result and release
  provider nodes that are no longer externally referenced.

One evaluation must observe one coherent property revision. Provider evaluation
occurs outside the property mutation lock.

## 8. Implementation model

The implementation separates mutable binding selection from immutable provider
computation:

```text
                              +------------------+
explicit slot -------------->|                  |
                              | selection node   |----> effective Provider<T>
convention slot ------------>|                  |
                              +------------------+
                                       |
                                       v
                           Map / FlatMap / Zip / OrElse
                              immutable provider DAG
```

A property cell contains:

```text
explicitPlan:   unset | ProviderNode<T>
conventionPlan: unset | ProviderNode<T>
lifecycle:      mutable | changesDisallowed | finalized
revision:       monotonically increasing identifier
```

Provider nodes include:

```text
Missing(reason)
Constant(value)
PropertyRead(propertyId)
ConventionRead(propertyId)
Map(source, transform, origin)
FlatMap(source, transform, origin)
Zip(left, right, transform, origin)
OrElse(primary, fallback, origin)
Fixed(result)
```

Evaluation returns a tagged `Present(value)` or `Missing(reason)`. It never uses
null as an internal control signal.

Nodes should preserve DAG sharing and compact provenance metadata. They must not
copy full transitive dependency sets at every node when that would produce
quadratic memory growth.

## 9. Minimum conformance examples

```kotlin
// Convention supplies the value while explicit is unset.
val p = property<String>()
p.convention("default")
assertValue(p, "default")

// Explicit value shadows but does not delete the convention.
p.set("explicit")
assertValue(p, "explicit")
p.unset()
assertValue(p, "default")

// A later convention is retained while explicit remains selected.
p.set("explicit")
p.convention("new default")
assertValue(p, "explicit")
p.unset()
assertValue(p, "new default")

// An explicit missing provider does not fall through to convention.
p.set(Provider.missing())
assertMissing(p)

// A deprecated nullable setter has the same explicit-missing semantics.
p.set(null)
assertMissing(p)
p.unset()
assertValue(p, "new default")

// orElse is explicit fallback-after-missing.
assertValue(Provider.missing<String>().orElse(providerOf("fallback")), "fallback")

// map observes the selected binding lazily.
val mapped = p.map { it.uppercase() }
assertValue(mapped, "NEW DEFAULT")
p.set("later")
assertValue(mapped, "LATER")

// Raw zip is missing-preserving.
assertMissing(
    Provider.missing<String>().zip(providerOf("x")) { a, b -> a + b }
)

// Empty is present and shadows a collection convention.
val list = listProperty<String>()
list.convention(listOf("a"))
list.set(emptyList())
assertValue(list, emptyList())
```

## 10. Design summary

```text
Property is a mutable explicit/convention binding and also a Provider
selection chooses explicit, then convention, then missing
selection occurs before provider evaluation
set replaces only the explicit slot
unset removes only the explicit slot
convention replaces only the convention slot
an explicit missing provider does not fall through to convention
map and flatMap transform one present source
zip combines two present sources
orElse is the explicit fallback-after-missing operation
deprecated set(null) installs an explicit missing plan; it does not unset
null is never a property value
```

This core is intentionally small. Collection arithmetic and other domain
operations should be defined from these primitives rather than given separate
rules for property selection or convention handling.
