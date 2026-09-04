# Property Provenance Implementation

## Status and intent

This document records implementation ideas and evidence for the semantic model
in [Property Provenance](PROPERTY_PROVENANCE.md). It describes the prototype on
the
[`asodja/provenance-prototype`](https://github.com/gradle/gradle/tree/asodja/provenance-prototype)
branch of `gradle/gradle`, what was measured, and what a subsequent
implementation would need to change.

The prototype implements the smallest ordinary-property slice: contributor
identity captured at the property mutation boundary, a bounded chronological
mutation history, and provenance in selected error messages. It does not
implement effective provenance, collaborative authorization and ordering, or
configuration-cache persistence.

Everything is behind an internal flag and is off by default:

```text
-Dorg.gradle.internal.property-provenance=true
```

An additional opt-in flag enables call-site stack walking:

```text
-Dorg.gradle.internal.property-provenance.stack-walk=true
```

With provenance disabled, existing diagnostic messages remain unchanged. The
prototype branch has 29 focused unit tests and 27 integration tests.

## Contents

- [1. Prototype integration](#1-prototype-integration)
- [2. Retained representation](#2-retained-representation)
- [3. Callback attribution](#3-callback-attribution)
- [4. Source locations](#4-source-locations)
- [5. Measured cost](#5-measured-cost)
- [6. Tracking coverage](#6-tracking-coverage)
- [7. Correctness gaps](#7-correctness-gaps)
- [8. Configuration cache](#8-configuration-cache)
- [9. Next implementation shape](#9-next-implementation-shape)
- [10. Recommended sequence](#10-recommended-sequence)

## 1. Prototype integration

Gradle already establishes a user-code application context while applying
plugins and scripts, and restores it across callbacks registered through its
normal configuration APIs. The prototype reads that context where the property
mutates:

```text
UserCodeApplicationContext.current()
        |
        | existing context, restored by Application.reapplyLater(...)
        v
ProjectBackedPropertyHost.currentMutation(kind)
        |
        v
MutationOriginRegistry.recordFor(source, kind)
        |
        v
AbstractProperty mutation boundary
```

The property host is the injection point. It is already provided to properties,
so no `PropertyFactory`, `ObjectFactory`, or property-constructor signature had
to change.

The prototype introduced these main concepts:

| Prototype type | Role |
|---|---|
| `ContributorKey` | Stable contributor identity |
| `MutationOrigin` | Contributor and user-readable origin derived from `UserCodeSource` |
| `MutationKind` | Setter-level operation such as `SET_SOURCE` or `SET_CONVENTION` |
| `MutationRecord` | Interned origin and mutation-kind pair |
| `MutationOriginRegistry` | Build-tree registry, interning table, and feature switch |
| `MutationTrace` | Bounded chronological history retained by a property |

The concrete seams are:

- `PropertyHost.tracksMutationProvenance()` is read once when a property is
  created and cached on the property;
- `PropertyHost.currentMutation(kind)` obtains the current attribution;
- `ValueState.host()` lets `AbstractProperty` reach the existing host;
- `AbstractMinimalProvider.describeProvenance(...)` is a no-op hook overridden
  by properties for missing-value diagnostics;
- `UserCodeSource` supplies the plugin or script source;
- existing task-provenance wording supplies display names such as
  `plugin 'x'` and `build file 'build.gradle.kts'`.

Checking whether provenance is enabled on each mutation was semantically
harmless but observably changed collaborator interactions. It broke 188 unit
tests that asserted a property touched no collaborators while being mutated.
Reading the decision once at construction both avoids that interaction and
reduces the disabled path to a field read.

The semantic model requires a two-phase refinement for collaborative use:
resolve and authorize the context before changing state, but append retained
provenance only after the mutation succeeds. The prototype only needs the
second half because ordinary provenance is diagnostic metadata.

## 2. Retained representation

`MutationRecord` instances are interned. Every property mutated in the same way
by the same origin can point at the same record. Recording a mutation therefore
requires a shared-reference write rather than one new object per occurrence.

The first mutation is stored directly. A property promotes that field to a
`MutationTrace` only when a second mutation arrives. This matters because the
average configured property in the Gradle build was mutated only 1.18 times.
The optimization reduced retained trace cost from about 6.9 MB to 1.2 MB on
that build.

The prototype retains successful mutations even after replacement, because it
implements chronological history. It caps that history at 32 records and
reports how many later mutations were omitted. The longest trace measured on
the Gradle build was eight, so the cap was not reached there.

That representation is suitable for bounded ordinary diagnostics, but it is
not yet the effective provenance defined by the semantic model:

- `SET_SOURCE` does not distinguish structural self-update from replacement;
- a flat history does not identify the selected source;
- it cannot identify which earlier source remains live through a self-update;
- it presents shadowed conventions and selected bindings in one list.

It is also not sufficient as the collaborative correctness trace. Contributor
ordering must be checked against every accepted update, so that trace cannot be
capped.

Anything that varies per occurrence defeats simple record interning. A source
location or runtime application should therefore be optional side data, a
parallel integer identifier, or part of a separately bounded diagnostic
occurrence rather than part of the full collaborative trace.

## 3. Callback attribution

`UserCodeApplicationContext.Application.reapplyLater(...)` already captures
application context when Gradle stores user code. The existing
`DefaultCollectionCallbackActionDecorator` and
`DefaultListenerBuildOperationDecorator` use that mechanism, so the prototype
did not add another general callback decorator.

The following registration points were verified:

| Registration point | Recorded contributor |
|---|---|
| Direct mutation during plugin application | registering plugin |
| `tasks.register { }` | registering plugin |
| `tasks.named { }` and `.configure { }` | registering plugin |
| `tasks.withType(...).configureEach { }` | registering plugin |
| Container `configureEach { }` | registering plugin |
| `project.afterEvaluate { }` | registering plugin |
| `gradle.projectsEvaluated { }` | registering plugin |
| `pluginManager.withPlugin(...) { }` | registering plugin |
| `gradle.taskGraph.whenReady { }` | registering plugin |
| Plugin applying another plugin | inner plugin, then outer context restored |

Coverage is a property of the registration boundary, not the user-code
language. A Groovy closure, Java `Action`, and Kotlin lambda registered through
the same boundary behave the same way.

The following have not been fully audited: tooling model builders, worker
actions, build services, flow actions, and init-script `beforeProject` and
`afterProject` callbacks.

## 4. Source locations

The prototype has two location mechanisms.

### Classpath instrumentation

Build-logic instrumentation rewrites an eligible call such as:

```java
property.set(value)
```

to a helper call carrying its source file and line as constants. This is exact
and adds no stack-walking cost. The location is published only for the duration
of that mutation, avoiding a stale location on a later uninstrumented call.

The prototype interceptor currently covers `set`. Java and Kotlin plugins in
instrumented build logic and Kotlin DSL scripts can receive locations. Groovy
build scripts use dynamic dispatch for assignment and explicit `set`, so they
need the separate Groovy interception path. Gradle distribution classes are
not instrumented.

### Bounded stack walk

An optional stack walk covers mutations without an instrumented call site. It
uses a property-specific caller capture that skips synthetic, line-less user
frames. The existing problem-reporting caller walk stops too early for Groovy
property assignment.

Captures are capped at 2,000 per build in the prototype. With that cap removed,
a real Gradle build measured an average 7.2 microseconds per walk. Twenty-two
percent reached the 50-frame limit without finding a usable location, making
unsuccessful captures the most expensive case.

Measured over 100,485 recorded mutations:

```text
locations from instrumentation    2,817  ( 2.8%)
locations from the stack walk    75,717  (75.4%)
no location found                21,951  (21.9%)
```

Instrumentation coverage was low partly because 69% of measured mutations were
conventions while only `set` was intercepted. More fundamentally, most
mutations came from Gradle's own uninstrumented distribution. Instrumentation
still covers the user-authored mutations for which a location is most
actionable.

The resulting default policy should be a location for instrumented user code
when available and contributor-only attribution otherwise. Stack walking can
remain an explicitly budgeted diagnostic option.

## 5. Measured cost

The following measurements compare the prototype with the same code after its
provenance fields were removed:

| Mode | Time per mutation | Retained memory |
|---|---:|---:|
| No provenance code | 3.4 ns | baseline |
| Compiled in, disabled | 5.1 ns | +8 B per property |
| Enabled, property mutated once | 13.7 ns | about +1 B per property |
| Enabled, property mutated more than once | 13.7 ns | +80 B, then about 4 B per record |
| Unbudgeted stack-walk location | +7.2 us | about +86 B per located record |

The Gradle build used for the larger measurement configured 252 subprojects
with isolated projects and the configuration cache disabled:

```text
properties created                       263,281
properties configured at least once       85,242
properties mutated more than once         13,703
mutations recorded                       100,485
longest history                                8 records
```

Estimated whole-build impact from those counts was:

| Mode | Added heap | Added time |
|---|---:|---:|
| Compiled in, disabled | 2.2 MB | about 0.2 ms |
| Contributor only, full retention | 3.3 MB | about 1 ms |
| Locations with the prototype budget | 3.5 MB | about 14 ms |
| Walked location on every mutation | 12 MB | about 720 ms |

Contributor-only provenance is therefore inexpensive on this evidence.
Interning and the single-record optimization are the decisions that make it
so. Per-occurrence locations are primarily a time concern.

## 6. Tracking coverage

The prototype installs a tracking host only in project scope:

```text
properties created with a tracking host       95,003  (36%)
properties created with a non-tracking host  168,248  (64%)

mutations attributed to a contributor         99,936  (35%)
mutations tracked but unattributed                549  (0.2%)
mutations on an untracked property            184,310  (65%)
```

The untracked population is not uniform:

- roughly half is internal machinery such as attribute containers and the
  managed object registry;
- roughly a third is isolation and deserialization recreating properties,
  where provenance belongs to the original property;
- the remainder includes real user-facing settings-, build-tree-, and
  non-project-scope extension properties.

Only the last category should simply gain another tracking host. Recreated
properties need provenance transferred from their source rather than
attributed to deserialization.

`UserCodeApplicationContext` is cross-build-session scoped and does not require
a project, so broader tracking is mostly wiring. The prototype registry is
build-tree scoped while the global property host is global scoped; those scopes
must be reconciled.

## 7. Correctness gaps

### Plausible but incorrect attribution

Where context propagation does not reach, the result may be wrong rather than
absent:

| Situation | Prototype attribution |
|---|---|
| Plugin-owned thread or executor | `Unknown` |
| Callback stored privately by a plugin and invoked later | whoever invoked it |
| Mutation of another property inside a Provider transform | whoever evaluated the transform |

Only the first is visibly unattributed. In the measured Gradle build, 343 of
the 549 tracked-but-unattributed mutations came from a plugin's asynchronous
compiler runner.

This is acceptable only as a documented limitation of best-effort ordinary
diagnostics. Collaborative authorization must require an explicit Declarative
Gradle contributor capability instead of trusting ambient diagnostic
attribution.

### Script role

The prototype propagated a `topLevelScript` boolean into `UserCodeSource`, which
distinguished build scripts from applied script plugins. It still classified
settings and initialization scripts as the build author. The input needs a
role such as build, settings, initialization, or applied script rather than a
boolean.

### Semantic operation

Both `p.set(p.map(f))` and `p.set(unrelatedProvider)` currently record
`SET_SOURCE`. Effective provenance and collaborative authorization must first
inspect the Provider structure and classify the former as an update and the
latter as a replacement.

### Collection and scope gaps

`ConfigurableFileCollection` does not share the `AbstractProperty` mutation
boundary and is not covered. `DirectoryProperty` and `RegularFileProperty` are
covered through `DefaultProperty`.

Collection contributions enter through the mutation boundary, but the current
diagnostic hooks can reduce their explanation to the last contribution. The
next representation should retain each accepted collection update as its own
semantic entry.

## 8. Configuration cache

Configuration-cache serialization currently collapses a property to its
Provider plan. The property object and its mutation trace are recreated during
deserialization, so a task property observed during execution has no provenance
on either the store run or a cache hit.

This is not optional polish. A production provenance feature must serialize:

- stable contributor and diagnostic-origin descriptors;
- effective source and update provenance;
- the complete collaborative correctness trace;
- only the bounded diagnostic history the product intends to preserve.

Runtime application objects and class loaders must not enter the cache entry.
Recreated and isolated properties must inherit serialized provenance rather
than recording deserialization as a new mutation.

## 9. Next implementation shape

The smallest extension from the prototype is to retain effective provenance
alongside, rather than derive it from, mutation history:

```text
PropertyProvenanceState
    conventionBinding: RecordId?
    explicitBinding: RecordId?
    effectiveUpdates: compact sequence of RecordId
    diagnosticHistory: bounded sequence of occurrences
```

The conceptual mutation path becomes:

```text
context = current mutation context
operation = structurally classify the requested mutation

if collaborative:
    require an explicit source-binding or contributor-update capability

apply the semantic mutation

if successful:
    update effective provenance
    append diagnostic history
    append the complete collaborative update trace when applicable
```

For ordinary properties, a replacement resets the effective source and update
sequence, while a self-update appends. For collaborative properties, a source
binding replaces only the source origin and an accepted contributor update
appends to both the Provider pipeline and the correctness trace.

The existing registry can continue interning stable origin-and-operation
records. Locations and application IDs can be held as optional integer side
data. A collaborative trace should use a compact integer array rather than one
object per occurrence.

Rendering then reads effective state directly:

```text
selected explicit or convention origin
    -> effective update records
```

It reads chronological history only when historical detail is requested.

## 10. Recommended sequence

1. Keep the semantic distinction between effective provenance and mutation
   history covered by focused model tests.
2. Add semantic operation classification and effective source/update state to
   the prototype.
3. Render effective provenance for existing final-value and missing-value
   problems, retaining the bounded history as secondary detail.
4. Introduce explicit collaborative source and contributor capabilities before
   using provenance for authorization.
5. Persist provenance through isolation and the configuration cache.
6. Add tracking hosts for user-facing settings and build-tree scopes.
7. Extend interception to convention and collection operations, and decide
   whether the optional stack walker is worth retaining.
