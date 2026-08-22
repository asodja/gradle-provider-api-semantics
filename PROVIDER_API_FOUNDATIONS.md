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
Property<T> = monotonic explicit state + convention plan + lifecycle state
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
    fun setMissing()

    fun convention(value: T)
    fun convention(provider: Provider<T>)

    fun finalizeValue()
    fun disallowChanges()
}
```

Deprecated nullable overloads may temporarily exist at an interoperability
boundary. Their meaning is defined in section 5; null never becomes a property
value.

## 2. Values, states, and selection

The following states are deliberately different:

| Term | Meaning |
|---|---|
| present | A provider produced a non-null value. |
| missing | A provider produced no value. |
| unconfigured | No explicit binding has ever been installed. |
| null | An invalid value; legacy nullable setters may translate it to an explicit missing binding. |
| empty | A present collection containing no elements. |

A property has a monotonic explicit state and a convention plan:

```text
explicit:   Unconfigured | Configured(Provider<T>)
convention: Provider<T>, initially Missing(NoConvention)
```

It selects a plan before evaluating that plan:

```text
selectedPlan(P) =
    explicit plan,   when explicit is Configured
    convention plan, when explicit is Unconfigured
```

The first `set` changes `Unconfigured` to `Configured`. Later `set` calls may
replace the explicit plan, including with a missing plan, but no operation
changes the property back to `Unconfigured`.

This one-way transition is the foundation of all other operations. In
particular, missing and unconfigured are different: a configured missing plan
shadows the convention permanently, while an unconfigured property selects the
convention.

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
lower-priority plan. The property continuously applies the priority rule
“explicit, otherwise convention,” independently of callback timing.

A convention:

- can be a constant or a provider;
- remains lazy when supplied by a provider;
- is retained while shadowed by an explicit binding;
- is replaceable while shadowed;
- is selected only while the explicit state is `Unconfigured`;
- is not an `orElse` fallback for an explicit provider that evaluates missing.

The self-reference extension can install an explicit plan that deliberately
contains `ConventionRead(P)` because no explicit binding existed before that
assignment—that is, the property was still unconfigured. In that case the
convention is an input of the selected explicit plan; the property has not
fallen through from an explicit missing result. The separate self-reference
specification defines this extension.

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

`C` and `D` are conventions, `V` and `W` are explicit values, and `Q` is an
explicit provider:

| Operations, in order | Selected binding | Observed result |
|---|---|---|
| none | initial missing convention plan | missing |
| `convention(C)` | convention `C` | result of `C` |
| `convention(C); set(V)` | explicit `V` | `V` |
| `set(V); convention(C)` | explicit `V` | `V` |
| `convention(C); set(Q)` | explicit `Q` | result of `Q` |
| `convention(C); set(missingProvider)` | explicit missing provider | missing |
| `convention(C); set(null)` (deprecated) | explicit missing plan | missing |
| previous row, then `set(W)` | explicit `W` | `W` |
| `convention(C); set(V); convention(D)` | explicit `V` | `V` |
| `convention(C); convention(missingProvider)` | missing convention | missing |

`set` changes only the explicit plan and permanently marks the property as
configured. `convention` changes only the convention plan. Neither operation
silently deletes the other plan, and an ordinary explicit binding can never
fall back to the convention.

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
convention: choose a source because the property is still unconfigured
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

property.convention("c")
upper.get() // "B"; the property remains explicitly configured

property.setMissing()
upper.isPresent() // false; the convention does not reappear
```

The derived provider is not given a separate copy of the convention. It reads
the property's selected plan.

When the explicit provider is missing, primitive operations remain strict:

| Property state | `map` | `flatMap` | `zip(present)` | `orElse(F)` |
|---|---|---|---|---|
| explicit `V`, convention `C` | transform `V` | select from `V` | combine `V` | `V` |
| unconfigured, convention `C` | transform `C` | select from `C` | combine `C` | `C` |
| unconfigured, missing convention | missing | missing | missing | result of `F` |
| explicit provider missing, convention `C` | missing | missing | missing | result of `F` |

## 5. Binding operations and null

The binding operations are:

```text
set(value/provider)          configure or replace the explicit plan
setMissing()                 configure an explicit missing plan
convention(value/provider)   replace the convention plan
```

There is no operation which returns a configured property to `Unconfigured`.
Once a property has received any explicit `set`, every later state is also
explicitly configured. A non-self `set` still replaces the previous plan and
allows it to be reclaimed; monotonic configuration does not mean that the
value itself is immutable.

The convention plan is initially missing. Replacing it with a missing provider
has the same observable result as having no convention, so a separate
`unsetConvention()` operation is unnecessary.

New APIs should not accept null in binding operations. If legacy nullable
overloads must remain temporarily, they should be deprecated and translated at
the API boundary as follows:

```text
set(null)        == setMissing() == set(Provider.missing())
convention(null) == convention(Provider.missing())
```

`set(null)` is therefore an explicit choice which shadows a convention
permanently, until another explicit plan replaces it. Treating `set(null)` as
removal is broken behavior because it changes the selected source rather than
representing the user's explicit assignment.

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

- an unconfigured property whose convention plan is missing;
- an explicitly configured missing plan;
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
explicit state ------------->|                  |
                              | selection node   |----> effective Provider<T>
convention plan ------------>|                  |
                              +------------------+
                                       |
                                       v
                           Map / FlatMap / Zip / OrElse
                              immutable provider DAG
```

A property cell contains:

```text
explicitState:  Unconfigured | Configured(ProviderNode<T>)
conventionPlan: ProviderNode<T> = Missing(NoConvention)
lifecycle:      mutable | changesDisallowed | finalized
revision:       monotonically increasing identifier
```

`Configured` is monotonic. A `set` may replace its node but cannot restore the
`Unconfigured` variant. This lets selection test one stable state bit rather
than interpret a missing provider result as absence of configuration.

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
// Convention supplies the value while the property is unconfigured.
val p = property<String>()
p.convention("default")
assertValue(p, "default")

// The first explicit set permanently configures the property.
p.set("explicit")
assertValue(p, "explicit")

// Replacing the convention cannot make it reappear through ordinary selection.
p.convention("new default")
assertValue(p, "explicit")

// An explicit missing provider does not fall through to convention.
p.set(Provider.missing())
assertMissing(p)
p.convention("another default")
assertMissing(p)

// A later explicit value replaces the missing plan.
p.set("later")
assertValue(p, "later")

// A deprecated nullable setter has the same explicit-missing semantics.
p.set(null)
assertMissing(p)
p.set("recovered")
assertValue(p, "recovered")

// orElse is explicit fallback-after-missing.
assertValue(Provider.missing<String>().orElse(providerOf("fallback")), "fallback")

// map observes the selected binding lazily.
val q = property<String>()
q.convention("default")
val mapped = q.map { it.uppercase() }
assertValue(mapped, "DEFAULT")
q.set("explicit")
assertValue(mapped, "EXPLICIT")

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

## 10. Behavior deliberately discarded from the current API

This specification is a redesign, not a compatibility description. Gradle's
[current `Property` API](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/Property.html)
allows `set(null)` and `value(null)` to discard the explicit value so that the
convention becomes selected. Its
[`SupportsConvention` API](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/SupportsConvention.html)
also exposes `unset()` and `unsetConvention()`.

This design deliberately discards those behaviors:

| Current-style behavior | New design |
|---|---|
| `set(null)` or `value(null)` removes the explicit binding | Deprecated compatibility spelling for `setMissing()` |
| `unset()` returns a property to its never-configured state | No equivalent; explicit configuration is monotonic |
| Clearing an explicit value reveals a convention | A convention never reappears through ordinary selection after `set` |
| `convention(null)` or `unsetConvention()` removes a convention slot | Replace the convention plan with `Provider.missing()` |
| Nullable binding overloads are normal configuration operations | New APIs reject null; compatibility overloads are deprecated |

There is intentionally no hidden replacement for “reset to convention.” A
caller that wants a particular value or provider must set that source
explicitly. The design retains ordinary explicit replacement, explicit missing
plans, convention replacement, and lifecycle finalization.

This trade-off removes reversible binding-selection state and prevents an old,
shadowed convention from unexpectedly becoming observable again.

## 11. Design summary

```text
Property is a mutable binding and also a Provider
explicit state moves once from Unconfigured to Configured
selection chooses convention only while explicit is Unconfigured
selection occurs before provider evaluation
set configures or replaces the explicit plan
setMissing installs an explicit missing plan
there is no unset operation and conventions do not reappear through selection
convention replaces the convention plan
an explicit missing provider does not fall through to convention
map and flatMap transform one present source
zip combines two present sources
orElse is the explicit fallback-after-missing operation
deprecated set(null) is another spelling of setMissing
null is never a property value
```

This core is intentionally small. Collection arithmetic and other domain
operations should be defined from these primitives rather than given separate
rules for property selection or convention handling.
