# Property Provenance

## Status and intent

This document defines the semantic model for explaining who configured a
property and which of those contributions still determine its effective
Provider plan. It also defines the attribution required by
[Collaborative Property Mode](COLLABORATIVE_PROPERTY_UPDATES.md) to validate
contributor order.

The implementation strategy, prototype results, measurements, and known Gradle
integration gaps are kept separately in
[Property Provenance Implementation](PROPERTY_PROVENANCE_IMPLEMENTATION.md).

This is an architectural proposal, not Gradle's current implementation or a
compatibility promise. It makes no concurrency guarantees.

## Contents

- [1. Provenance concepts](#1-provenance-concepts)
- [2. Attribution](#2-attribution)
- [3. Recording mutations](#3-recording-mutations)
- [4. Effective provenance](#4-effective-provenance)
- [5. Mutation history](#5-mutation-history)
- [6. Diagnostics](#6-diagnostics)
- [7. Collaborative properties](#7-collaborative-properties)
- [8. Minimum conformance cases](#8-minimum-conformance-cases)

## 1. Provenance concepts

Property provenance has several related but distinct views:

| Concept | Question it answers |
|---|---|
| Mutation attribution | Who performed this property mutation? |
| Effective provenance | Which source and structural updates determine the current plan? |
| Mutation history | Which successful mutations occurred, and in what chronological order? |
| Provider dependencies | Which providers, tasks, or external inputs does the value depend on? |
| Attempted mutation | Who performed an operation that was rejected? |

The first three are part of property provenance. Provider dependency metadata
is related but separate: it describes what a value depends on, not who bound or
updated the property. An attempted mutation belongs to the problem being
reported and must not be added to the history of successful mutations.

The distinction between effective provenance and mutation history is
load-bearing. A convention may be shadowed, an ordinary `set` may replace an
earlier plan, and a structural self-update may retain an earlier plan. A flat
chronological list cannot by itself say which of those cases occurred.

## 2. Attribution

A mutation occurrence can have the following attributes:

| Attribute | Meaning | Use |
|---|---|---|
| Contributor | Stable identity responsible for the mutation | Authorization and contributor ordering |
| Diagnostic origin | User-readable source, such as a plugin or build file | Explanations |
| Application | One runtime application of that source | Disambiguating occurrences when needed |
| Source location | File and line containing the mutation | Actionable diagnostics |
| Operation | Semantic effect of the mutation | Effective provenance and explanations |

These attributes must not be collapsed into one identifier. An application
identifies one runtime execution, not a stable contributor. A location
identifies a call site, not its authority. Several build files may share the
`BuildAuthor` contributor while retaining distinct diagnostic origins.

Contributor is required for collaborative ordering. The semantic operation is
required to decide what enters that ordering trace and to explain effective
provenance. Diagnostic origin is required for useful messages. Application and
source location are optional detail.

Typical contributor and origin pairs are:

| Current user code | Contributor | Diagnostic origin |
|---|---|---|
| Binary plugin with an ID | `Plugin(pluginId)` | `plugin 'pluginId'` |
| Binary plugin without an ID | `PluginClass(className)` | `plugin class 'className'` |
| Project build script | `BuildAuthor` | `build file 'relative/path'` |
| Applied script plugin | `ScriptPlugin(normalizedUri)` | `script 'relative/path'` |
| Settings script | distinct settings role | `settings file 'settings.gradle.kts'` |
| Initialization script | distinct environment role | `initialization script 'init.gradle'` |
| No attributable context | `Unknown` | `unknown code` |

Contributor identities used for authorization or persisted state must be stable
across builds. Runtime objects such as class loaders must not be part of that
identity. Script roles must be established where a script is applied; they
cannot be inferred from the fact that the source is a script.

Build-file paths should be relative to the build root. This keeps multi-project
diagnostics unambiguous and, for cross-project configuration, identifies the
file that performed the mutation rather than the project owning the property.

## 3. Recording mutations

Attribution belongs to the mutation boundary: `set`, `convention`, `add`,
`put`, and any other operation that changes property state.

If plugin A creates a mapped Provider and plugin B binds it to a property, the
binding is attributed to B:

```kotlin
// plugin A
val derived = source.map(transform)

// plugin B
target.set(derived)
```

The Provider dependency graph may retain information about `source` and
`transform`, but the property mutation says that plugin B selected that plan.
Creating or evaluating a Provider is not itself a property mutation.

For every attempted mutation, the property conceptually performs these steps:

```text
identify the current origin
classify the requested semantic operation
authorize it when the property is collaborative
apply the property mutation
record it only after it succeeds
```

A rejected mutation therefore leaves property state and mutation history
unchanged. Its attempted origin may still be attached directly to the failure.

Deferred configuration should retain causal attribution. A mutation in a
callback registered by a plugin is attributed to that plugin, not to the code
that later happens to trigger the callback. When causal attribution is not
available, an ordinary property records `Unknown`; a collaborative property
rejects the mutation as described in section 7.

Recording provenance must not evaluate a Provider, execute a transformation,
or reveal the value being configured.

## 4. Effective provenance

Effective provenance follows the structure of the effective Provider plan. It
is not reconstructed from call order.

Its general shape is:

```text
selected source
    -> structural update 1
    -> structural update 2
    -> ...
```

The selected source is the explicit source when the property is explicitly
configured and the convention otherwise, according to
[Provider API Foundations](PROVIDER_API_FOUNDATIONS.md). A shadowed convention
is useful supporting information, but is not a step in the effective chain.

For an ordinary property:

- a non-self replacement establishes a new source and cuts the previous
  effective chain;
- a recognized self-update retains the previous source and appends one
  structural update;
- a self-update performed while the property is unconfigured retains a live
  convention root, so a later convention replacement changes the source at the
  start of the chain;
- nested Provider transforms inside one assigned update are not separate
  property contributions.

Recognizing the difference between a replacement and a self-update requires
structural inspection as specified by
[Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md). The fact that
both operations entered through `set` is not sufficient.

For a collaborative property, source selection and the update pipeline are
separate. Rebinding the declarative source does not discard accepted plugin
updates. The effective chain is the currently selected explicit or convention
source followed by the complete accepted update pipeline.

### Example

Suppose these successful mutations occur. The line numbers identify the four
mutation call sites used by both views below:

```text
 1 | // base plugin
 2 | enabled.convention(true)
 3 |
 4 | // feature plugin
 5 | enabled.set(
 6 |     enabled.zip(featureAvailable) { current, available ->
 7 |         current && available
 8 |     }
 9 | )
10 |
11 | // declarative build source
12 | enabled.set(userChoice)
13 |
14 | // override plugin
15 | enabled.set(
16 |     enabled.zip(forceEnabled) { current, forced ->
17 |         current || forced
18 |     }
19 | )
```

Their chronological order is:

```text
1. line 2: convention bound by the base plugin
2. line 5: zip update contributed by the feature plugin
3. line 12: explicit source bound by the build author
4. line 15: zip update contributed by the override plugin
```

The same state renders as a configuration trace from the resulting plan back to
its selected source:

```text
Configuration trace to source:
    at line 15 [zip update from the override plugin]
    at line 5 [zip update from the feature plugin]
    at line 12 [explicit source from the build author]

Shadowed configuration:
    at line 2 [convention from the base plugin]
```

Line 15 is first because its update is closest to the resulting plan. Line 5
precedes line 12 chronologically, yet appears above it in the configuration
trace because collaborative source selection and the update pipeline are
separate. Line 12 supplies the selected source; the already accepted update
from line 5 and then the update from line 15 are applied to that source. The
convention from line 2 is not selected once line 12 binds an explicit source.

The effective plan computes:

```text
(userChoice && featureAvailable) || forceEnabled
```

Chronological history uses an oldest-first numbered list. A configuration trace
uses stack-trace-shaped frames from the resulting plan back to its source, so
the two views cannot be mistaken for each other.

## 5. Mutation history

Mutation history is an oldest-first sequence of successful operations. It may
include bindings that were later replaced and conventions that are currently
shadowed. This is useful when the question is not only "what determines the
value?" but also "how did configuration arrive here?"

An ordinary property's diagnostic history may be bounded. Truncation loses
detail but does not change property behavior. A renderer should report omitted
entries explicitly rather than silently presenting a partial list as complete.

A collaborative property's contributor update trace is different: it is
correctness state used to validate ordering and must be complete while the
property remains mutable. Console rendering may still abbreviate that trace.
Source bindings that do not participate in contributor ordering need not be
retained in the same unbounded structure.

Each accepted structural update is one history and ordering entry. A chain of
Provider transformations contained inside a single update remains one entry,
because the mutation is the unit of collaboration.

## 6. Diagnostics

Provenance should be silent during a successful ordinary build. It is shown
when it explains a property-related problem or when the user explicitly asks
for a property explanation.

Diagnostics should:

- lead with the property problem;
- show only provenance relevant to that problem by default;
- show effective provenance before historical detail;
- distinguish an attempted mutation from accepted configuration;
- include locations only when they are available and actionable;
- never print property values merely to explain provenance;
- avoid claiming that the contributor who bound a Provider produced all of its
  dependencies.

A failure trace starts with the operation that exposed or attempted to change
the property. It then walks the effective configuration from the resulting
plan back to the selected source. The failed operation is not itself part of
the configuration provenance.

An explicit explanation has no failed-operation frame, so it shows only the
configuration trace:

```text
Property task ':show' property 'enabled'

Configuration trace to source:
    at plugin 'com.example.override' (OverridePlugin.kt:27) [zip update]
    at plugin 'com.example.feature' (FeaturePlugin.kt:41) [zip update]
    at build file 'app/build.gradle.kts' (build.gradle.kts:18) [explicit source]

Shadowed configuration:
    at plugin 'com.example.base' (BasePlugin.kt:18) [convention]
```

A failed mutation should show both sides without adding the attempt to history:

```text
The value for task ':show' property 'enabled' is final and cannot be changed.

Failure trace to source:
    at build file 'app/build.gradle.kts' (build.gradle.kts:24) [set()]
    at plugin 'com.example.feature' (FeaturePlugin.kt:41) [explicit source]

Shadowed configuration:
    at plugin 'com.example.base' (BasePlugin.kt:18) [convention]
```

When an explicitly selected Provider is missing, the message should distinguish
that from an unconfigured property and from convention fallback:

```text
Cannot query task ':show' property 'enabled' because its selected Provider has
no value.

Failure trace to source:
    at task ':show' action (ShowTask.kt:34) [get()]
    at plugin 'com.example.override' (OverridePlugin.kt:27) [zip update]
    at plugin 'com.example.feature' (FeaturePlugin.kt:41) [zip update]
    at build file 'app/build.gradle.kts' (build.gradle.kts:18) [explicit source]

Shadowed configuration:
    at plugin 'com.example.base' (BasePlugin.kt:18) [convention]
```

## 7. Collaborative properties

Collaborative properties use provenance for correctness as well as
diagnostics. They retain the complete ordered sequence of structural updates
and validate it against the property's effective contributor order before
observation or lifecycle closure.

Contributor identity inferred from a general ambient execution context is
sufficient for best-effort ordinary diagnostics, but it is not by itself proof
of authority. Collaborative mutation must occur in an explicit source-binding
or contributor-update context established by Declarative Gradle. This prevents
a callback stored by one plugin and invoked by another source from silently
acquiring the invoker's authority.

An unidentified or unauthorized mutation fails before changing the property:

```text
Cannot update collaborative property 'enabled': no authorized contributor is
active.

Run the update from a Declarative Gradle contributor action that preserves its
registration context.
```

An invalid order fails before any Provider transformation is evaluated and
reports the required relationship and the first conflicting pair:

```text
Cannot query collaborative property 'enabled': contributor updates are out of
order.

Failure trace to source:
    at task ':show' action (ShowTask.kt:34) [get()]
    at plugin 'com.example.feature' (FeaturePlugin.kt:41) [zip update]
    at plugin 'com.example.override' (OverridePlugin.kt:27) [zip update]
    at build file 'app/build.gradle.kts' (build.gradle.kts:18) [explicit source]

Required order:
    plugin 'com.example.feature' before plugin 'com.example.override'

Conflicting updates, in application order:
    1. zip update by plugin 'com.example.override' at OverridePlugin.kt:27
    2. zip update by plugin 'com.example.feature' at FeaturePlugin.kt:41
       This update must precede update 1.

The Provider value was not evaluated.
```

Provenance required for either explanation or validation must survive copying,
isolation, and configuration-cache serialization. Persisted contributor and
origin descriptors must remain stable and must not contain runtime application
or class-loader objects.

## 8. Minimum conformance cases

- a direct mutation during plugin application is attributed to that plugin;
- a plugin applied by class receives a stable implementation contributor;
- project, settings, initialization, and applied script sources have distinct
  diagnostic roles;
- a deferred and a nested deferred callback retain the registrant's origin;
- applying another plugin temporarily changes and then restores attribution;
- a cross-project mutation names the build file that performed it;
- a replacing `set` cuts ordinary effective provenance;
- a structural self-update retains ordinary effective provenance;
- a convention-rooted self-update reflects a later convention replacement;
- a collaborative source rebind retains the accepted update pipeline;
- mutation history includes successful shadowed or superseded operations;
- a rejected mutation appears only as the attempted origin on its problem;
- a failure trace begins with the failed operation and continues through the
  effective configuration to the selected source, while shadowed configuration
  remains outside that trace;
- an unattributed ordinary mutation is represented as `Unknown`;
- an unidentified or unauthorized collaborative mutation fails before changing
  property state;
- an ordering failure identifies the first conflicting updates without
  evaluating the Provider plan;
- a configuration-cache round trip preserves effective provenance and any
  correctness trace;
- recording or explaining provenance does not evaluate Provider
  transformations or print their values.
