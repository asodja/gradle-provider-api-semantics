# Collaborative Property Mode

## Status and intent

This document describes an optional binding mode for properties configured by
Declarative Gradle Reactive Plugins. It extends the model in
[Provider API Foundations](PROVIDER_API_FOUNDATIONS.md) and uses the shared
attribution mechanism in
[Property Provenance](PROPERTY_PROVENANCE.md).

Reactive Plugins operate on `Property` instances directly. A collaborative
property attributes their structural updates and immediately composes them
into one Provider pipeline. Before observation or lifecycle closure, it
validates the recorded update order against the applicable contributor order.

Ordinary properties keep their existing semantics. Collaborative mode is
activated by Declarative Gradle only for properties in its model. This is an
architectural proposal, not Gradle's current implementation or a compatibility
promise. It makes no concurrency guarantees.

## 1. Mode and observation

```text
Ordinary

or

Collaborative
```

A collaborative property accepts attributed source bindings and structural
updates. Each structural update is attributed, recorded, and composed
immediately. Contributor ordering is validated later over the complete update
trace available at that point.

Because the property always has a current Provider plan, ordinary value queries
remain valid:

```text
get()
getOrNull()
isPresent()
```

Before evaluating the plan, a query validates the recorded update order. If it
is valid, the query observes the source and update prefix accepted at that
point. A later update or constraint invalidates the cached validation and may
therefore change a later result or make a later query fail validation. Querying
does not leave collaborative mode or prevent later changes.

Provider derivation is also unchanged:

```kotlin
val derived = property.map(f)
```

Existing Property lifecycle controls retain their ordinary meaning in
collaborative mode. Ordering constraints count as changes to the collaborative
property. Before lifecycle closure, the property validates its update trace.
`disallowChanges()` can do this without evaluating the Provider value and then
rejects later source bindings, updates, and constraints. `finalizeValue()`
validates before fixing the value of the currently composed plan. An
implementation that supports `finalizeValueOnRead()` validates before applying
its normal behavior.

## 2. Property state

Collaborative mode separates source selection from plugin transformations:

```text
CollaborativeState<T>:
    conventionPlan
    explicitSource
    updatePipeline
    updateTrace
    localConstraints
    validationState
```

`updatePipeline` is initially the identity transformation.
`updateTrace` records each structural operation's contributor, kind, and
diagnostic origin as defined by
[Property Provenance](PROPERTY_PROVENANCE.md). `validationState`
caches whether the current trace and constraints have been checked.

The existing explicit-versus-convention rule selects the source:

```text
selectedSource =
    explicitSource,  when explicitly configured
    conventionPlan,  otherwise
```

The effective plan is:

```text
effectivePlan = updatePipeline(selectedSource)
```

A declarative assignment can therefore replace the explicit source without
discarding previously composed plugin transformations. A plugin transformation
does not replace the user's source.

## 3. Bindings and structural updates

Declarative Gradle invokes code in an internal context that identifies whether
it is binding a source or contributing a Reactive Plugin update.

| Operation and context | Meaning |
|---|---|
| Owning plugin calls `convention(q)` | Bind the convention source |
| Declarative source context calls `set(q)` | Bind the explicit source |
| Contributor calls `set(p.map(f))` | Record contributor and compose `Map(Previous, f)` |
| Contributor calls `set(p.flatMap(f))` | Record contributor and compose `FlatMap(Previous, f)` |
| Contributor calls `set(p.zip(q, f))` | Record contributor and compose `Zip(Previous, q, f)` |
| Contributor calls `set(p.plus(q))` | Record contributor and compose `Append(q)` |
| Contributor calls `set(p.minus(q))` | Record contributor and compose `Remove(q)` |
| Reactive Plugin calls non-self `set(q)` | Reject unauthorized replacement |
| Unattributed code mutates a collaborative property | Reject |

A plain declarative assignment uses the source context. Declarative
append/prepend and other self-updates use a reserved build-author contributor.

For a structural self-update:

```kotlin
p.set(p.zip(q, f))
```

the property identifies the transformation:

```text
Zip(Previous, q, f)
```

`Previous` means the update pipeline already accepted for `p`. The property
requires an authorized contributor, records that contributor and the operation,
and composes the transformation onto the pipeline. It does not validate
contributor order at this assignment.

Derivation without assignment remains non-mutating:

```kotlin
val q = p.map(f)
```

A self-read hidden inside an opaque callback is not structural:

```kotlin
p.set(provider { p.get() })
```

It remains an unsupported provider cycle as specified in
[Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md).

## 4. Contributor-order validation

Declarative Gradle supplies the default order for Reactive Plugin contributors,
for example:

```text
base-plugin < feature-plugin < override-plugin < build-author
```

A property may declare local ordering constraints while it remains mutable.
Declarative Gradle combines those hard constraints with global order as the
tie-breaker to produce the property's effective contributor order. The local
constraints must be acyclic. Because validation is separate from composition,
constraints may be declared before or after structural updates.

The recorded updates for one property must form a nondecreasing sequence in the
effective order. Updates from the same contributor preserve source order.

Conceptually, updates and constraints are recorded as follows:

```text
acceptUpdate(contributor, update):
    require contributor is identified and authorized

    updatePipeline = update(updatePipeline)
    updateTrace.append(contributor, update.kind, update.origin)
    validationState = dirty

addConstraint(constraint):
    require property is mutable

    localConstraints.add(constraint)
    validationState = dirty
```

Before observation or lifecycle closure, the property validates the trace:

```text
validateOrder():
    if validationState is dirty:
        effectiveOrder = resolve(globalOrder, localConstraints)
        require updateTrace is nondecreasing in effectiveOrder
        validationState = valid
```

An invalid trace fails before any Provider transformation is evaluated. The
property does not reorder updates to make the trace valid.

A property may override the default relationship:

```kotlin
enabled.collaboration {
    contributor("feature-plugin").before("override-plugin")
}
```

The resulting effective order applies only to `enabled`. Other properties keep
the global default unless they declare their own constraints.

Deferred callbacks retain the identity of the Reactive Plugin action that
registered them. Callback execution order is allowed to vary only when it
produces an update sequence compatible with each property's effective order;
otherwise observation or lifecycle closure fails.

The property does not inspect callback bodies or decide whether transformations
commute. Tags such as `Append` and `Remove` are useful for provenance and
diagnostics, not for reordering.

## 5. Immediate composition and deferred validation

The property does not defer Provider construction or replay transformations at
`get()`. It incrementally maintains one Provider transformation pipeline and a
lightweight provenance trace used only for ordering validation.

For example:

```text
initial pipeline: Identity
accept pluginA:   Zip(Previous, a, f)
accept pluginB:   Map(Previous, g)
```

produces:

```text
Map(Zip(selectedSource, a, f), g)
```

and records:

```text
[pluginA: Zip, pluginB: Map]
```

Before observing the value, the property validates the trace and then evaluates
the already composed plan. This preserves laziness, missing-value propagation,
producer information, dependencies, and ordinary Provider validation.

An implementation may store contributor metadata on structural Provider nodes
or in a compact side trace. The observable semantics do not depend on that
choice.

Collection updates compose using
[Derived Collection Operations](DERIVED_COLLECTION_OPERATIONS.md):

```text
Append(q) -> previous.zip(q, plus)
Remove(q) -> previous.zip(q, minus)
```

## 6. Example

Suppose the global contributor order is:

```text
feature-plugin < override-plugin
```

Both updates are composed immediately:

```kotlin
enabled.convention(true)

// feature-plugin
enabled.set(
    enabled.zip(featureAvailable) { current, available ->
        current && available
    }
)

// override-plugin
enabled.set(
    enabled.zip(forceEnabled) { current, forced ->
        current || forced
    }
)
```

The effective plan computes:

```text
(selectedSource && featureAvailable) || forceEnabled
```

The recorded trace is:

```text
feature-plugin -> override-plugin
```

It satisfies the global order, so observation succeeds.

Now consider the reversed application:

```kotlin
// override-plugin
enabled.set(
    enabled.zip(forceEnabled) { current, forced ->
        current || forced
    }
)

// feature-plugin
enabled.set(
    enabled.zip(featureAvailable) { current, available ->
        current && available
    }
)
```

Both updates are composed, producing:

```text
(selectedSource || forceEnabled) && featureAvailable
```

but the recorded trace is invalid under the global order. An observation such
as `enabled.get()` fails before evaluating the Provider plan:

```text
Cannot observe 'enabled': contributor updates are out of order.

Required contributor order:
  feature-plugin < override-plugin

Recorded update order:
  override-plugin -> feature-plugin
```

The reversed application can instead be made valid for this property by
declaring its local order while the property remains mutable. The constraint
may be declared before or after the structural updates:

```kotlin
enabled.collaboration {
    contributor("override-plugin").before("feature-plugin")
}
```

The effective order for `enabled` is then:

```text
override-plugin < feature-plugin
```

The reversed trace is now valid, so observation succeeds with the already
composed plan:

```text
(selectedSource || forceEnabled) && featureAvailable
```

Other properties continue to use the global default. Changing the global order
itself is also possible, but affects every collaborative property without a
local override.

## 7. Relationship to ordinary Property

| Area | Collaborative mode |
|---|---|
| Provider values and missingness | Unchanged |
| Explicit-versus-convention selection | Selects the source before updates |
| Ordinary `set` | Unchanged outside collaborative mode |
| Declarative source `set` | Binds the source without discarding updates |
| Reactive Plugin self-`set` | Records provenance and composes immediately |
| Reactive Plugin non-self `set` | Rejected |
| Property constraints | May change while mutable and invalidate cached validation |
| Value queries | Validate the trace, then observe the composed update prefix |
| Lifecycle controls | Validate the trace before closing ordinary mutations |

Ordinary self-assignment immediately captures the previous plan:

```text
P.set(P.map(f)) -> P.set(previous(P).map(f))
```

Collaborative self-assignment records its contributor and composes the same
Provider operation onto the update pipeline. Before observation or lifecycle
closure, the property validates the recorded contributor sequence. Both modes
ultimately use the same Provider primitives.

## 8. Scope and open questions

Collaborative mode initially applies only to Declarative Gradle model
properties configured by Reactive Plugins. Kotlin DSL, Groovy DSL, and ordinary
imperative-plugin properties retain their existing behavior.

The remaining integration questions are:

- how Declarative Gradle activates the mode;
- how it declares the global contributor order;
- which contexts may bind the explicit source or convention; and
- whether Declarative Gradle applies an ordinary Property lifecycle control
  when reactive configuration completes.
