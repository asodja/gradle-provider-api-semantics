# Collaborative Property Mode

## Status and intent

This document describes an optional binding mode for properties configured by
Declarative Gradle Reactive Plugins. It extends the model in
[Provider API Foundations](PROVIDER_API_FOUNDATIONS.md).

Reactive Plugins operate on `Property` instances directly. A collaborative
property attributes their structural updates, validates that they are applied
in the declared contributor order, and immediately composes them into one
Provider pipeline.

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
updates. Each accepted update is checked and composed immediately.

Because the property always has a valid current Provider plan, ordinary value
queries remain valid:

```text
get()
getOrNull()
isPresent()
```

A query observes the source and update prefix accepted at that point. A later
accepted update may therefore change a later observation, just as a later
ordinary property mutation can. Querying does not leave collaborative mode or
prevent later updates.

Provider derivation is also unchanged:

```kotlin
val derived = property.map(f)
```

Existing Property lifecycle controls retain their ordinary meaning in
collaborative mode. For example, `disallowChanges()` rejects later source
bindings and updates, while `finalizeValue()` fixes the value of the currently
composed plan and rejects later changes. An implementation that supports
`finalizeValueOnRead()` applies its normal behavior.

## 2. Property state

Collaborative mode separates source selection from plugin transformations:

```text
CollaborativeState<T>:
    conventionPlan
    explicitSource
    updatePipeline
    lastContributor
```

`updatePipeline` is initially the identity transformation.

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
| Contributor calls `set(p.map(f))` | Validate and compose `Map(Previous, f)` |
| Contributor calls `set(p.flatMap(f))` | Validate and compose `FlatMap(Previous, f)` |
| Contributor calls `set(p.zip(q, f))` | Validate and compose `Zip(Previous, q, f)` |
| Contributor calls `set(p.plus(q))` | Validate and compose `Append(q)` |
| Contributor calls `set(p.minus(q))` | Validate and compose `Remove(q)` |
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

`Previous` means the update pipeline already accepted for `p`. After validating
the contributor, the property composes the transformation onto that pipeline.

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

Before its first structural update, a property may declare local ordering
constraints. Declarative Gradle combines those hard constraints with global
order as the tie-breaker to produce the property's effective contributor order.
The local constraints must be acyclic. Changing them after the first update is
rejected because the existing pipeline has already been composed.

Updates to one property must arrive in nondecreasing effective order. Updates
from the same contributor preserve source order.

Conceptually, an update is accepted as follows:

```text
acceptUpdate(contributor, update):
    require contributor is identified
    require lastContributor <=property contributor

    updatePipeline = update(updatePipeline)
    lastContributor = contributor
```

If an earlier contributor updates the property after a later contributor, the
assignment fails. The property does not retain the update and reorder it later.

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
still produces an update sequence compatible with each property's effective
order.

The property does not inspect callback bodies or decide whether transformations
commute. Tags such as `Append` and `Remove` are useful for provenance and
diagnostics, not for reordering.

## 5. Immediate composition

The property does not keep a list of updates until `get()`. It incrementally
maintains one Provider transformation pipeline.

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

No value is queried while composing the pipeline. Evaluating the current plan
preserves laziness, missing-value propagation, producer information,
dependencies, and validation.

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

This succeeds because updates are applied in that order:

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

The reversed application fails:

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

After accepting the `override-plugin` update, the property cannot accept an
update from the earlier `feature-plugin`:

```text
Cannot update 'enabled' from feature-plugin.

Required contributor order:
  feature-plugin < override-plugin

Observed update order:
  override-plugin < feature-plugin
```

The reversed application can instead be made valid for this property by
declaring its local order before either update:

```kotlin
enabled.collaboration {
    contributor("override-plugin").before("feature-plugin")
}
```

The effective order for `enabled` is then:

```text
override-plugin < feature-plugin
```

The reversed application succeeds and composes:

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
| Reactive Plugin self-`set` | Validates order and composes immediately |
| Reactive Plugin non-self `set` | Rejected |
| Value queries | Observe the currently composed update prefix |
| Lifecycle controls | Keep their ordinary finalization and mutation rules |

Ordinary self-assignment immediately captures the previous plan:

```text
P.set(P.map(f)) -> P.set(previous(P).map(f))
```

Collaborative self-assignment first validates its contributor, then composes
the same Provider operation onto the update pipeline. Both modes ultimately
use the same Provider primitives.

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
