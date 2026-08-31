# Collaborative Property Mode

## Status and intent

This document describes an optional collaborative binding mode for the
`Property` model in
[Provider API Foundations](PROVIDER_API_FOUNDATIONS.md). The mode is intended
for properties configured by Declarative Gradle Reactive Plugins.

Reactive Plugins operate on `Property` instances directly. Contributor-aware
collaboration therefore cannot exist only in a separate declarative model: the
property implementation must retain the contributions until the reactive model
has finished configuring it.

Ordinary properties keep the foundational binding semantics. Collaborative
mode is an additional lifecycle and binding mode activated by Declarative
Gradle, not a replacement for `Property` behavior in Kotlin DSL, Groovy DSL, or
ordinary imperative plugins.

This is an architectural model, not a final API or a description of Gradle's
current implementation. This version makes no concurrency guarantees for
`Property`.

## Contents

1. [Why collaboration belongs in Property](#1-why-collaboration-belongs-in-property)
2. [Property modes and lifecycle](#2-property-modes-and-lifecycle)
3. [Collaborative state](#3-collaborative-state)
4. [Binding and update operations](#4-binding-and-update-operations)
5. [Contributor identity](#5-contributor-identity)
6. [Global ordering and property constraints](#6-global-ordering-and-property-constraints)
7. [Resolution and lazy evaluation](#7-resolution-and-lazy-evaluation)
8. [Examples](#8-examples)
9. [Relationship to the foundational model](#9-relationship-to-the-foundational-model)
10. [Adoption boundary](#10-adoption-boundary)
11. [Open design questions](#11-open-design-questions)

## 1. Why collaboration belongs in Property

An external declarative resolver would be sufficient if it were the only code
allowed to configure the eventual Gradle properties. That is not the Reactive
Plugin model. A Reactive Plugin can configure the same `Property` that another
Reactive Plugin or the declarative input configures.

For example:

```kotlin
// Reactive Plugin A
enabled.set(
    enabled.zip(featureAvailable) { current, available ->
        current && available
    }
)

// Reactive Plugin B
enabled.set(
    enabled.zip(forceEnabled) { current, forced ->
        current || forced
    }
)
```

If each `set` immediately rewrites the explicit provider plan, callback
execution order determines the result. Collaborative mode instead records both
updates with their contributor identities and orders them using the Reactive
Plugin collaboration rules.

The property remains responsible for provider selection and evaluation. The
collaborative mode adds only the information required to resolve its binding.

## 2. Property modes and lifecycle

A property has one of two binding modes:

```text
Ordinary
Collaborative
```

An ordinary property has exactly the semantics specified in
[Provider API Foundations](PROVIDER_API_FOUNDATIONS.md).

A collaborative property moves through these phases:

```text
Ordinary
    |
    | Declarative Gradle activates collaboration
    v
Collaborating
    |
    | the reactive model closes this property
    v
Resolved
```

`Collaborating` accepts attributed source bindings and structural updates.
`Resolved` contains one ordinary provider plan and rejects every later
mutation.

The transition to collaborative mode is internal to Declarative Gradle. A
public call such as the following is illustrative, not proposed API:

```kotlin
property.enableCollaborativeMode(context)
```

Reactive Plugins and build authors should not have to enable the mode
themselves.

### 2.1 Queries while collaboration is open

Querying an open collaborative property would expose a result that can change
when another Reactive Plugin contributes later. It would reintroduce the
temporal coupling the mode is intended to remove.

This model therefore rejects value queries while the property is
`Collaborating`, including:

```text
get()
getOrNull()
isPresent()
```

Provider derivation remains allowed:

```kotlin
val derived = property.map(f)
```

The derived provider can be retained as part of an update, but it cannot be
queried until the collaborative property has resolved.

## 3. Collaborative state

The foundational convention and explicit-source states remain separate. A
collaborative property adds attributed updates and ordering information:

```text
CollaborativeState<T>:
    conventionPlan: ProviderPlan<T>
    explicitSource: Unconfigured | ProviderPlan<T>
    updatesByContributor: Map<ContributorId, UpdatePipeline<T>>
    globalOrder: ContributorOrder
    localConstraints: Set<OrderConstraint>
```

The selected source uses the existing property rule:

```text
selectedSource(P) =
    explicitSource(P),  when explicitly configured
    conventionPlan(P),  otherwise
```

The final plan applies the ordered updates to that selected source:

```text
resolvedPlan(P) =
    orderedUpdates(P).fold(selectedSource(P)) { previous, update ->
        update(previous)
    }
```

This separation is important. A declarative user assignment selects the
source; it does not discard transformations contributed by Reactive Plugins.
Similarly, a plugin update does not replace the user's selected source.

## 4. Binding and update operations

Calls on an open collaborative property are interpreted using both their
structure and their Declarative Gradle execution context.

| Context and operation | Collaborative meaning |
|---|---|
| Owning plugin calls `convention(q)` | Bind or replace the convention plan |
| Declarative source context calls `set(q)` | Bind or replace the explicit source plan |
| Contributor calls `set(p.map(f))` | Record `Map(Previous, f)` |
| Contributor calls `set(p.flatMap(f))` | Record `FlatMap(Previous, f)` |
| Contributor calls `set(p.zip(q, f))` | Record `Zip(Previous, q, f)` |
| Contributor calls `set(p.plus(q))` | Record a tagged `Append(q)` update |
| Contributor calls `set(p.minus(q))` | Record a tagged `Remove(q)` update |
| Reactive Plugin calls non-self `set(q)` | Reject unauthorized replacement |
| Unattributed code mutates the property | Reject while collaboration is open |

The execution context is controlled by Declarative Gradle. Plugin code cannot
claim declarative-source or owning-plugin authority for itself.

Declarative Gradle selects the context from the configured operation. A plain
declarative assignment binds the explicit source, while declarative
append/prepend or another self-update runs as the reserved build-author
contributor. The same build author may therefore select a source and also add
an ordered transformation without conflating the two operations.

### 4.1 Structural self-updates

A collaborative update assigns a provider expression that structurally reads
the target property:

```kotlin
p.set(p.map(f))
p.set(p.flatMap(f))
p.set(p.zip(q, f))
p.set(p.plus(v))
```

At the assignment boundary, `set` replaces the target read with `Previous` and
records the update:

```text
p.set(p.zip(q, f))

becomes

updatesByContributor[currentContributor].add(
    Zip(Previous, q, f)
)
```

`Previous` means the resolved contribution prefix ordered below this update.
It does not mean the value that happened to be visible when the callback ran.

Derivation without assignment remains non-mutating:

```kotlin
val q = p.map(f)
```

### 4.2 Opaque self-references

A target read hidden inside user code is not structurally visible:

```kotlin
p.set(provider { p.get() })
```

It cannot become a collaborative update. It remains an unsupported provider
cycle and must fail according to
[Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md).

### 4.3 Structure, not callback interpretation

The resolver does not inspect callback code. These are both valid ordered
updates:

```kotlin
enabled.set(enabled.map { false })
enabled.set(enabled.zip(flag) { current, value -> current && value })
```

The first update can semantically ignore its input, and the second happens to
implement Boolean `AND`. Their callback bodies are opaque to the property.

Built-in operations may retain tags such as `Append` or `Remove` for better
diagnostics and provenance. Tags are not required to prove commutativity or to
resolve ordering; global and local contributor order already determine the
composition.

## 5. Contributor identity

Declarative Gradle invokes every Reactive Plugin action in an internal scoped
context:

```text
ContributionContext:
    applicationId
    pluginId
    configurationScope
    globalOrderKey
```

`applicationId` identifies one application of a plugin. It is more precise
than `pluginId` because the same plugin may be applied in several scopes.

Deferred callbacks must restore the identity of the Reactive Plugin action
that registered them. Execution thread and callback timing are not contributor
identity.

Declarative source assignment and owning-plugin convention binding use
reserved contexts with narrowly defined authority. An ordinary imperative
plugin running outside a registered context cannot mutate a collaborative
property while it is open.

## 6. Global ordering and property constraints

Declarative Gradle provides a global order for Reactive Plugin contributors.
That order supplies the default meaning of composition for every collaborative
property.

For example:

```text
base-plugin
feature-plugin
override-plugin
build-author
```

Source order is preserved between updates from the same contributor.

A property may add exceptional `before` or `after` constraints:

```kotlin
strictMode.collaboration {
    contributor("override-plugin").before("feature-plugin")
}
```

This is illustrative syntax. The constraint belongs to `strictMode`; it does
not reorder those plugins for other properties.

### 6.1 Resolution algorithm

For one property, the resolver:

1. selects only contributors that updated that property;
2. groups each contributor's updates in source order;
3. constructs a graph from the property's local `before` and `after`
   constraints;
4. performs a stable topological sort;
5. uses global contributor order to choose between contributors that are
   otherwise unconstrained;
6. concatenates the resulting contributor pipelines.

Local constraints are hard edges. Global order is the default ordering and the
stable tie-breaker, so a local constraint can express a property-specific
exception without redefining the global order.

Resolution fails when:

- local constraints contain a cycle;
- a participating contributor has no usable global ordering identity;
- a mutation is unattributed;
- a Reactive Plugin attempts unauthorized source replacement; or
- the property is mutated after resolution.

Generic `map`, `flatMap`, and `zip` updates do not conflict merely because
their functions are opaque. Once contributor order is established, their
composition is deterministic.

## 7. Resolution and lazy evaluation

The property must retain update structure until its participating contributors
and local constraints are known. It does not have to retain every update as a
separate object indefinitely. Updates from one contributor can be compacted
into an `UpdatePipeline<T>` while preserving source order and diagnostic
provenance.

When the reactive model closes the property, the resolver substitutes each
ordered `Previous` with the provider plan below it.

For example:

```text
SelectedSource(base)
pluginA: Zip(Previous, a, f)
pluginB: Map(Previous, g)
```

compiles to:

```text
Map(Zip(base, a, f), g)
```

Resolution does not query `base` or `a`. It constructs an ordinary provider
plan. The foundational provider evaluator later preserves:

- laziness;
- missing-value propagation;
- producer and dependency information;
- validation and finalization behavior; and
- the usual `get()` and `isPresent()` results.

After compilation, the property is `Resolved`. Queries delegate to the
resolved provider plan, and every further binding or update is rejected.

## 8. Examples

### 8.1 Boolean property

Suppose global order places `feature-plugin` before `override-plugin`:

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

The resolved provider computes:

```text
(selectedSource && featureAvailable) || forceEnabled
```

The resolver does not need a Boolean-specific conflict policy. If one property
requires the opposite order, it declares a local ordering constraint.

### 8.2 List property

```kotlin
compilerArgs.convention(listOf("-g"))

// java-plugin
compilerArgs.set(compilerArgs.plus(providerOf("-parameters")))

// quality-plugin
compilerArgs.set(compilerArgs.minus(providerOf("-Werror")))

// declarative build author
compilerArgs.set(compilerArgs.plus(providerOf("--enable-preview")))
```

If the global order is `java-plugin`, `quality-plugin`, `build-author`, the
resolved plan is equivalent to:

```kotlin
selectedSource
    .zip(providerOf("-parameters"), listPlus)
    .zip(providerOf("-Werror"), listMinus)
    .zip(providerOf("--enable-preview"), listPlus)
```

The `plus` and `minus` semantics, including missing-value behavior, come from
[Derived Collection Operations](DERIVED_COLLECTION_OPERATIONS.md).

## 9. Relationship to the foundational model

Collaborative mode changes binding, not provider evaluation:

| Existing area | Collaborative behavior |
|---|---|
| Provider values and missingness | Unchanged after resolution |
| `map`, `flatMap`, `zip`, `orElse` | Used to compile the resolved plan |
| Explicit-versus-convention selection | Selects the source before updates |
| Ordinary `set` | Unchanged outside collaborative mode |
| Declarative source `set` | Binds the explicit source without discarding updates |
| Reactive Plugin self-`set` | Records an attributed update |
| Reactive Plugin non-self `set` | Rejected |
| Structural self-reference | Retained until contributor resolution |
| Finalization and `disallowChanges()` | Reject every later change |

For an ordinary property, self-referential assignment immediately captures the
previous plan as specified in
[Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md):

```text
P.set(P.map(f)) -> P.set(previous(P).map(f))
```

For an open collaborative property, the same structural form records a
contribution:

```text
P.set(P.map(f)) -> Map(currentContributor, Previous, f)
```

Once ordered and compiled, both forms produce the same primitive Provider
graph for the same chosen sequence of operations.

## 10. Adoption boundary

Collaborative mode should initially be enabled only for properties that are
part of a Declarative Gradle model configured by Reactive Plugins.

Consequences of this boundary:

- Kotlin DSL and Groovy DSL retain ordinary `Property` semantics.
- Existing imperative plugins retain ordinary semantics for ordinary
  properties.
- Reactive Plugin actions receive contributor identity without general-purpose
  plugin instrumentation.
- Direct imperative mutation of an open collaborative property is diagnosed
  instead of silently entering the contribution order.
- The same public `Property<T>` interface can be backed by a mode-aware
  implementation; a separate public property type is not required.

A diagnostic-only migration phase may record where Reactive Plugins currently
depend on callback order or perform unauthorized replacement before
collaborative semantics are enforced.

## 11. Open design questions

The model intentionally leaves these API and integration details open:

- how Declarative Gradle identifies and activates collaborative properties;
- how global contributor order is declared and how unknown contributors are
  reported;
- which Declarative Gradle contexts may replace the explicit source or
  convention;
- how property closure aligns with reactive-model stabilization,
  `finalizeValue()`, and `disallowChanges()`;
- whether diagnostics retain every original update after contributor pipelines
  have been compacted; and
- how existing Reactive Plugins migrate from arbitrary replacement to
  structural updates.

The core decisions do not depend on those API details:

```text
collaboration is an optional Property mode for Declarative Gradle
Reactive Plugin updates are attributed and retained
declarative source selection is separate from plugin transformations
global contributor order provides the default update order
properties add only exceptional local ordering constraints
resolution compiles to the existing Provider algebra
ordinary Property behavior remains unchanged
```
