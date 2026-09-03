# Property Mutation Provenance

## Status and intent

This document describes how a property can record who configured it, so that
diagnostics can name the plugin, script plugin or build script responsible, and
so that [Collaborative Property Mode](COLLABORATIVE_PROPERTY_UPDATES.md) has a
contributor identity to validate update order against.

It is deliberately short. An earlier version specified the mechanism in detail;
this version keeps the model and replaces the speculation with what a prototype
against Gradle actually showed, including the parts that do not work. Numbers
below are measured on the Gradle build itself unless stated otherwise.

This is an architectural proposal, not Gradle's current implementation or a
compatibility promise. It makes no concurrency guarantees.

## Contents

- [1. What provenance identifies](#1-what-provenance-identifies)
- [2. How it works](#2-how-it-works)
- [3. Where the contributor comes from](#3-where-the-contributor-comes-from)
- [4. What it costs](#4-what-it-costs)
- [5. What is not covered](#5-what-is-not-covered)
- [6. What collaborative mode additionally requires](#6-what-collaborative-mode-additionally-requires)
- [7. Minimum conformance cases](#7-minimum-conformance-cases)

## 1. What provenance identifies

| Question | Representation | Purpose |
|---|---|---|
| Who caused the mutation? | `ContributorKey` | Authorization and ordering |
| Which execution caused it? | `ApplicationId` | Diagnostics for one application |
| Where was it declared? | `SourceLocation` | Actionable diagnostics |
| What happened? | `MutationKind` | Explaining the effective plan |

These must not be collapsed into one identifier. An application ID identifies
one runtime application, not a stable contributor. A source location identifies
a call site, not the contributor that caused it. Provider producer and
dependency metadata describes what a value depends on, not who mutated the
property.

Only the first is required. The other three are useful and, as section 4 shows,
the second and third are what make provenance expensive.

## 2. How it works

Gradle already establishes a user-code application context while applying
plugins and scripts, and already restores it across the callbacks it stores.
Provenance is that context, read at the moment a property is mutated.

```text
user code application context   (already exists; restored across callbacks)
            |
            v
    property host          asked once per property: do you track provenance?
            |              asked per mutation: who is running right now?
            v
    mutation boundary      set, convention, unset, add, put
            |
            v
    interned record        (contributor, kind), shared build-wide
```

Four rules make this work:

**Capture at the mutation boundary.** Every binding or update reads the current
attribution where the property mutates, after the mutation has succeeded, so a
rejected mutation leaves no trace. Attribution belongs to the mutation: if
plugin A creates `p.map(f)` and plugin B calls `p.set(thatProvider)`, the
contributor is B. A transform's creation context must not replace it.

**Reach the context through the property's host.** The host is already handed to
every property when it is created, so no property constructor, property factory
or object factory signature has to change.

**Decide whether to track when the property is created, not per mutation.**
Asking the host on every mutation is behaviour-neutral but observable: it broke
188 existing Gradle unit tests that assert a property touches no collaborators
while being mutated. Asking once, at construction, moves that interaction
outside those assertions and reduces the switched-off cost to a field read. Any
implementation will meet the same constraint.

**Intern the records.** A record is `(contributor, kind)` and is shared by every
property that a given contributor mutates the same way, so recording costs one
reference write and no allocation. Section 4 shows that interning is the single
decision that makes provenance affordable, and section 6 what it forbids.

Propagation across deferred configuration needs no new mechanism.
`Application.reapplyLater` already wraps callbacks at registration, so a
property set inside `tasks.register { }`, `tasks.named { }`,
`withType(...).configureEach { }`, `afterEvaluate { }`, `projectsEvaluated { }`,
`pluginManager.withPlugin(...) { }`, `taskGraph.whenReady { }` or a container
`configureEach { }` is attributed to the plugin that registered the callback,
not to whoever triggered it. Nesting works too: a plugin applying another plugin
gets its own attribution restored when the inner application returns.

This is a property of the registration boundary, not of the kind of user code. A
Groovy closure, a Java `Action` and a Kotlin lambda registered through the same
API behave identically, so there is no separate closure problem to solve.

## 3. Where the contributor comes from

| Current user code | Contributor key |
|---|---|
| Binary plugin with an ID | `Plugin(pluginId)` |
| Binary plugin without an ID | `PluginClass(className)` |
| Build script | `BuildAuthor` |
| Applied script plugin | `ScriptPlugin(normalizedUri)` |
| No user code context | `Unknown` |

Contributor keys must be stable across builds. Runtime objects such as a
`ClassLoader` must not be part of persisted identity.

The user code source describes the code being applied but does not say what
*role* it plays. A build script and an applied script plugin are both scripts,
and an initialization script and a settings script are indistinguishable from a
project build script. A script's role must therefore be recorded where the
script is applied, as a role rather than a boolean: conflating an init script
with the build author would let the environment inherit whatever authority the
build author has.

Display names should follow the wording task provenance already uses, so that
`plugin 'com.example.feature'`, `plugin class 'FeaturePlugin'`,
`build file 'lib/build.gradle'` and `script 'other.gradle'` read the same way
everywhere. Naming a build file by a path relative to the build root keeps
multi-project builds unambiguous and, for cross-project configuration, names the
file that performed the mutation rather than the project that owns the property.

## 4. What it costs

Measured against a prototype, on the Gradle build itself (252 subprojects,
263,281 properties created, 100,485 recorded mutations):

| | per mutation | per property | on the Gradle build |
|---|---|---|---|
| No provenance at all | 3.4 ns | — | 0 |
| Compiled in, switched off | 5.1 ns | +8 B | 2.2 MB |
| Contributor only | 13.7 ns | +1 B, or +80 B once mutated twice | 3.3 MB |
| Plus a call site on every mutation | +~7 µs | +~86 B per record | 12 MB, ~720 ms |

Contributor-only provenance is close to free: about 3 MB and a millisecond on a
build whose configuration takes minutes, in a daemon holding several GB.

Three observations shape any implementation:

- **A single mutation must not allocate.** The average configured property in
  the Gradle build is mutated 1.18 times, so a property should hold the interned
  record directly and only promote to a list on a second mutation. Doing this
  reduced the retained cost from 6.9 MB to 1.2 MB on that build.
- **Call sites are a time cost, not a memory cost.** A bounded stack walk costs
  about 7 µs on average in a real build, so capturing one for every mutation
  adds around 700 ms of configuration. The cost scales with the frames the walk
  must skip, and is worst when it finds nothing: on the Gradle build 22% of
  captures reach the depth cap without finding a user frame. Locations should
  therefore be optional and budgeted, in the way problem reporting already
  budgets its stack captures.
- **The existing bounded caller capture is not sufficient as it stands.** It
  stops at the first Gradle frame below a user frame, and a Groovy property
  assignment puts a generated, line-less accessor frame there, so the walk ends
  before reaching the script. A mutation call site needs a walk that steps over
  synthetic user frames.

## 5. What is not covered

The honest limits, measured rather than assumed.

**Only project-scope properties are tracked.** A tracking host exists only at
project scope; global and worker scopes record nothing. On the Gradle build that
is 64% of properties created and 65% of mutations. That population is not
uniform:

- roughly half is internal machinery no one would want attributed, such as
  attribute containers and the managed object registry;
- roughly a third is value isolation and deserialization re-creating properties,
  where provenance belongs to the original mutation and not to the copy;
- the remainder is real user-facing configuration created from a settings-scope
  or otherwise non-project object factory, including plugins' own extension
  objects. Only this last part is a gap worth closing, and closing it is mostly
  a matter of providing a tracking host in more scopes.

**The configuration cache erases provenance.** A task property observed at
execution has no provenance under the configuration cache, on the store run as
well as on a hit, because the property is recreated during deserialization
rather than by the user code that configured it. Provenance must be persisted
with the entry, using stable descriptors rather than runtime application or
class-loader objects, or the feature is blank in any build that uses the cache.

**Attribution can be confidently wrong.** Where propagation does not reach, the
mutation is not left unattributed but attributed to whoever ran it:

| Situation | Attributed to |
|---|---|
| A plugin's own thread or executor | `Unknown` |
| User code a plugin stores itself and runs later | whoever ran it |
| A property mutated as a side effect inside a `Provider` transform | whoever evaluated the transform |

All three are written by one plugin and recorded against someone else:

```kotlin
// In plugin "com.example.feature".

// 1. A thread Gradle did not create. There is no context to restore, so the
//    mutation is recorded as Unknown rather than as this plugin.
executor.submit { task.prop.set("x") }

// 2. A callback this plugin stores itself. Gradle never saw the registration,
//    so it could not wrap it. Recorded as whoever runs it later, which for
//    `Holder.runAll()` called from build.gradle is the build author.
Holder.store { task.prop.set("x") }

// 3. A transform that mutates another property while it is evaluated. Recorded
//    as whoever first calls `trigger.prop.get()`.
trigger.prop.set(provider {
    other.prop.set("x")
    "x"
})
```

The third case is narrow, and worth stating precisely because it is easy to
misread. Setting a property to a mapped provider is a mutation by whoever calls
`set`, and reading a property records nothing at all, not even when the read
finalizes it. So `a.set(b.map { })` in plugin A, later read by plugin B, is
correctly attributed to A. What misattributes is only a lambda that mutates some
*other* property while it runs, because that mutation happens during evaluation
and inherits the evaluator's context.

Only the first case is visible as unattributed; the other two produce a
plausible and wrong contributor. This is the reason collaborative mode must fail
closed rather than trust a recorded contributor. These are not hypothetical: on
the Gradle build, most unattributed mutations come from one plugin mutating
properties from its own asynchronous compiler runner.

**A structural self-update is indistinguishable from a replacement.** Both
`p.set(p.map(f))` and `p.set(unrelatedProvider)` record the same kind, so the
mutation kind alone cannot tell a contribution from a replacement. Collaborative
mode has to treat those differently, which means inspecting the provider plan
for a self-reference rather than reading the kind.

**`ConfigurableFileCollection` is not covered** by a mechanism built on the
property mutation boundary, because it does not share that boundary.

## 6. What collaborative mode additionally requires

Ordinary properties use provenance only as diagnostic metadata and may discard
what a replacing `set` made irrelevant. Collaborative properties additionally
retain the ordered update trace and validate its contributors. Three
consequences follow from section 4:

**The retained record must carry nothing per-occurrence.** Interning is what
makes a full trace affordable: a trace is a list of shared references costing
about 4 bytes per additional record, against about 80 bytes per record without
it. Anything that varies per mutation, such as a call site or an application
ID, makes every record a distinct object. Since the application ID is not used
for contributor ordering, it must be kept out of the retained trace, or held
alongside it as an integer rather than as an object.

**A per-property cap on retained records is not acceptable.** A bounded trace is
right for diagnostics, where truncation only costs detail, but a truncated trace
cannot be validated against a contributor order.

**The granularity is already right.** Each mutation is one entry, and each
collection contribution is its own entry with its own contributor, so the
ordered contributor sequence a validation needs is directly available. A chain
of transforms inside a single update is one entry, which is also correct: the
update is the unit of collaboration, not the transform steps within it.

**Unattributed mutation must be rejected.** `Unknown` cannot be authorized or
placed in contributor order, so a collaborative property must fail before
changing its state, and say so:

```text
Cannot update collaborative property 'enabled': no contributor is active.

The callback executing this mutation did not preserve its registration
context.
```

This fail-closed rule makes a missing propagation visible instead of silently
assigning the mutation to the wrong plugin.

On the Gradle build the practical cost of full retention is the same as
diagnostics retention: the longest trace observed is 8 records and nothing
approaches a cap.

## 7. Minimum conformance cases

- a direct mutation during plugin application has that plugin's ID;
- a plugin applied by class receives a stable implementation contributor;
- a build script mutation belongs to `BuildAuthor`, and an init or settings
  script does not;
- a deferred and a nested deferred callback retain the registrant's contributor;
- applying another plugin temporarily changes and then restores attribution;
- a build file is named unambiguously in a multi-project build, and
  cross-project configuration names the file that mutated the property;
- ordinary properties preserve their existing mutation results, and their
  diagnostics are unchanged when provenance is switched off;
- an unattributed ordinary mutation is reported as `Unknown`;
- an unattributed collaborative mutation fails before changing the property;
- a configuration cache round trip preserves provenance;
- recording provenance does not evaluate Provider transformations;
- recording a single mutation allocates nothing.
