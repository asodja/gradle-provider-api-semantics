# Property Mutation Provenance

## Status and intent

This document describes a shared provenance mechanism for property mutations
made by imperative plugins, scripts, Declarative Gradle, and Reactive Plugins.
It supports the contributor-order rules in
[Collaborative Property Mode](COLLABORATIVE_PROPERTY_UPDATES.md), while also
allowing ordinary properties to report where their configuration came from.

This is an architectural proposal, not Gradle's current implementation or a
compatibility promise. It makes no concurrency guarantees.

## 1. What provenance identifies

Property mutation provenance answers four separate questions:

| Question | Representation | Purpose |
|---|---|---|
| Who caused the mutation? | `ContributorKey` | Collaboration authorization and ordering |
| Which execution caused it? | `ApplicationId` | Diagnostics for a particular plugin or script application |
| Where was it declared? | `SourceLocation` | Actionable diagnostics |
| What happened? | `MutationKind` | Explaining the effective property plan |

These concepts must not be collapsed into one identifier. In particular:

- an application ID identifies one runtime application, not a stable
  contributor;
- a source location identifies a call site, not the contributor that caused it;
- Provider producer and dependency metadata describes what a value depends on,
  not who mutated the property.

This document concerns mutation provenance. Existing Provider producer and
dependency propagation remains unchanged.

## 2. Provenance records

Conceptually, a mutation record contains:

```text
MutationRecord:
    originId
    applicationId?
    kind
```

`originId` refers to an interned descriptor:

```text
MutationOrigin:
    contributor
    location?
    frontend
```

Possible contributor keys include:

```text
Plugin(pluginId)
PluginClass(className, implementationIdentity)
Script(normalizedUri)
ReactivePlugin(pluginId)
BuildAuthor
GradleInternal(component)
Unknown
```

New collaborative contributors should have stable explicit IDs. A binary
plugin applied without an ID may fall back to a stable implementation identity,
such as its class name together with the identity of the build logic or plugin
implementation that supplied the class. Runtime objects such as a `ClassLoader`
must not be part of persisted identity.

`MutationKind` describes the property operation, for example:

```text
SetSource
SetConvention
MapUpdate
FlatMapUpdate
ZipUpdate
Append
Remove
Add
AddAll
Put
PutAll
```

The kind is useful for diagnostics. It does not make operations commutative and
does not authorize reordering.

## 3. Allocating `originId`

A build-scoped registry interns normalized origins:

```text
intern(origin):
    if origin is already known:
        return its existing originId
    originId = next build-local ID
    store originId -> origin
    return originId
```

For example:

```text
1 -> Plugin("java"), JavaPlugin.java, IMPERATIVE
2 -> Plugin("com.example.feature"), FeaturePlugin.kt:42, IMPERATIVE
3 -> ReactivePlugin("override"), defaults.dcl:17:5, DECLARATIVE
4 -> BuildAuthor, build.gradle.kts:23, KOTLIN_DSL
```

An `originId` is only a compact handle within one build state. It must not be
used as a contributor key or assumed stable across builds. Configuration Cache
serialization writes the origin table or the referenced descriptors and
re-interns them when loading an entry.

`applicationId` is stored on the mutation occurrence rather than in the
interned origin. This allows repeated applications of the same plugin call site
to share one origin descriptor.

## 4. Capturing imperative plugin and script provenance

Gradle already establishes a user-code application context while applying
binary plugins and scripts. A property mutation context can adapt the current
application as follows:

| Current user-code source | Contributor key |
|---|---|
| Binary plugin with an ID | `Plugin(pluginId)` |
| Binary plugin without an ID | `PluginClass(className, implementationIdentity)` |
| Build script | `BuildAuthor` |
| Applied script plugin | `Script(normalizedUri)` |
| No user-code context | `Unknown` |

The current binary-versus-script distinction is not by itself enough to tell a
build script from an applied script plugin. The script application boundary
must also supply that role, for example by enriching the user-code source
descriptor. Inferring it later from a display name or URI would be fragile.

The current application ID may be copied into the mutation record for
diagnostics. It is not used for contributor ordering.

The user-code context usually identifies a plugin or script but not an exact
line inside a binary plugin. Precise legacy locations may be supplied by
bytecode instrumentation of property mutation call sites. Without that
instrumentation, the plugin ID, implementation class, and script URI still
provide useful provenance.

Call-site instrumentation is a diagnostic supplement, not the primary source
of contributor identity. A helper class may be called by several plugins, so
its class or code source alone cannot identify which plugin caused a mutation.

The existing JVM and Groovy call-interception pipeline can be extended for this
purpose, but its current runtime interceptor input is not sufficient for a
precise location. It can inject the caller class, while source file and line are
currently available to the bytecode visitor and instrumentation-time reporting
rather than to the runtime interceptor. Precise mutation locations therefore
require either new source-file and line parameters in that pipeline or a
property-specific equivalent. This does not require changing how contributor
identity is propagated.

## 5. Capturing Declarative and Reactive Plugin provenance

Declarative Gradle supplies attribution explicitly when applying an assignment
or executing a Reactive Plugin action:

```text
withAttribution(
    contributor = ReactivePlugin("feature-plugin"),
    intent = StructuralUpdate,
    location = declarative source range
):
    execute plugin action
```

A build-author assignment instead uses `BuildAuthor` and the `SourceBinding`
intent. Declarative assignment analysis already has the source element and
operation generation; conversion must preserve the relevant source range when
it invokes the runtime property setter.

Explicit Declarative attribution takes precedence over an enclosing imperative
plugin context. Otherwise a Reactive Plugin invoked through a bootstrap plugin
would be incorrectly attributed to that bootstrap plugin.

The active attribution is therefore selected in this order:

```text
explicit Declarative or Reactive Plugin attribution
    orElse current imperative user-code application
    orElse Unknown
```

## 6. Propagating attribution through callbacks

An active thread-local context alone is insufficient because many mutations
occur in callbacks executed after plugin application. Gradle must capture the
current attribution when user code is registered and restore it when the code
is executed.

For example:

```kotlin
// Executed while plugin "com.example.feature" is active.
tasks.configureEach {
    enabled.set(enabled.map { current -> current && supported })
}
```

`configureEach` stores a contextual callback:

```text
decorate(action):
    snapshot = captureCurrentAttribution()
    return action(argument):
        withAttribution(snapshot):
            delegate.execute(argument)
```

When the callback runs later, `set` still observes
`Plugin("com.example.feature")`.

One internal callback decorator should provide wrappers for the user-code forms
that Gradle stores, including `Action`, `Closure`, `Spec`, `Runnable`,
`Callable`, and relevant functional interfaces. Public Gradle APIs that retain
user code must decorate it at registration rather than storing a raw callback.
This includes task and domain-object configuration, lifecycle listeners,
dependency-resolution callbacks, tooling model builders, and other deferred
configuration actions.

Internally, stored user code should have a contextual type rather than the raw
callback type. This makes context capture an invariant of the storage boundary
instead of a convention each callback subsystem must remember. Architecture
checks can reject new fields or queues that retain raw user callbacks, while
integration tests exercise the supported registration boundaries.

Attribution scopes must be nested and restored in `finally`. If a callback from
plugin A applies plugin B, mutations during B's application belong to B; after
B returns, the active contributor is A again. A callback registered by another
callback captures the currently restored contributor.

The contributor is normally the code that registered the callback. If a build
script registers a callback that later calls a method on a plugin object, the
mutation belongs to `BuildAuthor`, not to the class containing that method.

Gradle can guarantee propagation only for Gradle-managed callbacks. Mutation
from an arbitrary thread or executor created by a plugin has no reliable causal
context and is unattributed unless that mechanism explicitly propagates a
snapshot.

### Existing closure instrumentation

Gradle's existing Groovy closure instrumentation cannot propagate property
provenance as it stands. It wraps `doCall` to track closures participating in
Groovy dynamic-call interception and installs metaclass hooks when necessary.
It does not capture the user-code application that registered a closure or
restore a registration-specific attribution when the closure executes.

Capturing attribution when a closure object is created would also have the
wrong semantics. A closure may be created in one context, registered later in
another, or registered more than once by different contributors. Provenance is
causal, so it belongs to each registration. A wrapper allocated at registration
can carry the correct snapshot for that particular use.

The current closure visitor could be extended as a diagnostic safety hook, but
it should not become the primary propagation mechanism. It also covers only
Groovy closures, while Gradle retains Java and Kotlin `Action` and SAM
implementations as well. Registration-time contextual wrappers provide one
model for all of them.

The implementation split is therefore:

| Requirement | Recommended mechanism |
|---|---|
| Correct plugin or script contributor | Existing user-code application context, captured by a centralized registration-time callback decorator |
| Explicit Reactive Plugin contributor | Declarative attribution scope, captured by the same decorator |
| Precise legacy source file and line | Extend general call interception to pass call-site metadata, or use a property-specific interceptor |
| Missing-context detection | Collaborative property rejects the mutation; instrumentation may report the call site |

No new general closure instrumentation is required for contributor propagation.
The existing callback-decoration approach needs to be made comprehensive and
shared across callback registration points.

Provider transforms are a separate concern. A `map`, `flatMap`, or `zip`
callback may retain its creation context for evaluation diagnostics, but the
contributor of `p.set(p.map(f))` is captured when `set` mutates `p`. Transform
creation context must not replace mutation attribution in contributor-order
validation.

## 7. Recording provenance on properties

Every binding or update captures attribution at the mutation boundary:

```text
mutate(kind, operation):
    attribution = currentAttribution()
    require attribution is allowed by the property's mode
    originId = originRegistry.intern(attribution.origin)
    apply operation
    record(originId, attribution.applicationId, kind)
```

Applying the operation and retaining its record are conceptually one atomic
mutation. A rejected or failed operation does not remain in the property's
provenance trace; its captured attribution may still be used to report the
failure.

Attribution belongs to the property mutation. For example, if plugin A creates
`p.map(f)` and plugin B later calls `p.set(thatProvider)`, the assignment is a
mutation by B. Provider-node provenance may additionally describe where `f`
was created, but it must not replace B as the contributor used for ordering.

Ordinary and collaborative properties use the same capture mechanism but apply
different policy:

| Mode | Provenance behavior |
|---|---|
| Ordinary | Retain effective provenance for diagnostics; mutation semantics are unchanged |
| Collaborative | Also retain the ordered update trace and validate its contributors |

Outside collaborative mode, an implementation may discard provenance made
irrelevant by a replacing `set` or convention replacement. It need not retain
a complete audit history. Provenance for collection contributions and
structural self-updates remains relevant while those contributions remain in
the effective plan.

Collaborative source bindings store the provenance of the selected explicit or
convention source. Structural updates append compact mutation records to the
collaborative update trace. The already composed Provider pipeline remains the
source of value evaluation; provenance is not replayed to compute the value.

## 8. Legacy mutations of collaborative properties

An imperative plugin with an active user-code context is attributed, not
anonymous. It can participate naturally when the operation has structural
meaning, for example:

```text
convention(...)
add(...), addAll(...)
put(...), putAll(...)
p.set(p.map(...))
p.set(p.zip(...))
```

Declarative Gradle may include that plugin's stable contributor key in the
applicable order. No source change to the imperative plugin is required.

An arbitrary imperative `p.set(otherProvider)` is a replacement, not a
structural collaboration update. It keeps ordinary replacement semantics on an
ordinary property. On a collaborative property it must either run in an
explicitly authorized source-binding context or be rejected; it must not be
silently reinterpreted as a collaborative transformation.

## 9. Unknown provenance and diagnostics

Ordinary mode may record a shared `Unknown` origin without changing behavior.
Collaborative mode must reject an unattributed mutation because `Unknown`
cannot be authorized or placed in contributor order.

```text
Cannot update collaborative property 'enabled': no contributor is active.

Mutation call site:
  FeatureHelper.kt:42

The callback executing this mutation did not preserve its registration
context.
```

This fail-closed rule makes missing callback decoration visible instead of
silently assigning the mutation to the wrong plugin. Instrumented call-site
information can improve the error, but must not guess the contributor.

## 10. Storage and lifecycle

Provenance metadata should be compact and allocated lazily:

- intern contributor and origin descriptors in a build-scoped table;
- store small origin IDs and operation tags on mutation records;
- allocate the full ordered trace only for collaborative properties;
- avoid adding provenance fields to every Provider transformation node;
- avoid increasing the base size of every ordinary property merely to hold a
  nullable provenance object.

An implementation may attach ordinary effective provenance to immutable
binding or contribution nodes and use a compact side trace for collaborative
order validation. Structural Provider substitution must preserve relevant
provenance when it path-copies a plan.

Before observation or lifecycle closure, collaborative order validation reads
the contributor keys referenced by the trace. It does not evaluate Provider
transforms. Configuration Cache encoding must preserve any provenance needed
for later diagnostics or validation, using stable descriptors rather than
runtime application or class-loader objects.

## 11. Minimum conformance cases

A provenance implementation should verify at least these cases:

- a direct mutation during imperative plugin application has that plugin's ID;
- a plugin applied by class receives a stable implementation contributor;
- a build-script mutation belongs to `BuildAuthor`;
- a deferred and a nested deferred callback retain the registrant's
  contributor;
- applying another plugin temporarily changes and then restores attribution;
- Declarative assignment locations survive runtime conversion;
- explicit Reactive Plugin attribution overrides an enclosing bootstrap-plugin
  context;
- ordinary properties preserve their existing mutation results;
- a mixed imperative and Reactive Plugin trace validates by contributor key;
- an unattributed ordinary mutation is reported as `Unknown`;
- an unattributed collaborative mutation fails before changing the property;
- a Configuration Cache round trip preserves stable provenance descriptors;
- recording provenance does not evaluate Provider transformations.

## 12. Design summary

```text
imperative plugin or script context ----+
                                        |
Declarative explicit context -----------+--> current attribution
                                                |
callback capture and restoration ---------------+
                                                |
property mutation ------------------------------+
                                                v
                                  interned origin + mutation record
                                                |
                         +----------------------+-------------------+
                         |                                          |
               ordinary diagnostics                  collaboration validation
```

One provenance mechanism serves both imperative and Declarative plugins.
Ordinary properties use it only as diagnostic metadata. Collaborative
properties additionally require attributable contributors and use their stable
keys to validate update order. The Provider value algebra remains unchanged.
