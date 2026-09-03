# Property Provenance: What the Prototype Found

A record of what was learned building property mutation provenance against
Gradle, so that the evidence outlives the conversation that produced it.

- The model is in [Property Mutation Provenance](PROPERTY_PROVENANCE.md).
- The code is on `asodja/provenance-prototype` in `gradle/gradle`, with its
  design notes in `PROPERTY_PROVENANCE_PROTOTYPE.md` on that branch.

Everything below is measured unless it says otherwise. Build numbers come
from the Gradle build itself: `gradle help`, 252 subprojects, isolated
projects and the configuration cache disabled so that every project is
configured.

## Contents

- [1. What was built](#1-what-was-built)
- [2. What it costs](#2-what-it-costs)
- [3. What surprised us](#3-what-surprised-us)
- [4. What does not work](#4-what-does-not-work)
- [5. Open decisions](#5-open-decisions)

## 1. What was built

A property records who configured it, read from Gradle's existing user code
application context at the moment of mutation, reached through the
property's host. Records are interned per (contributor, kind). Build logic
additionally gets the call site of its own mutations, baked in by classpath
instrumentation.

```
The value for task ':show' property 'prop' is final and cannot be changed any further.
It was configured by:
  given its convention by plugin 'java-library'
  -> set by plugin 'com.example.feature' at FeaturePlugin.java:12
  -> set by build file 'build.gradle.kts' at build.gradle.kts:7
```

Off by default behind `-Dorg.gradle.internal.property-provenance=true`. 29
unit and 27 integration tests.

## 2. What it costs

Per mutation and per property, against a baseline of the same code with the
provenance fields removed:

| | per mutation | per property |
|---|---|---|
| No provenance at all | 3.4 ns | — |
| Compiled in, switched off | 5.1 ns | +8 B |
| On, property mutated once | 13.7 ns | +1 B |
| On, property mutated more than once | 13.7 ns | +80 B, then ~4 B per record |
| A call site from the stack walk | +7.2 µs | +86 B per record |
| A call site from instrumentation | free | +86 B per record |

On the Gradle build, which creates 263,281 properties and records 100,485
mutations:

| | added heap | added time |
|---|---|---|
| Compiled in, switched off | 2.2 MB | ~0.2 ms |
| Contributor only, full retention | 3.3 MB | ~1 ms |
| A call site on every mutation, walked | 12 MB | ~720 ms |

**Time is never the problem for contributor-only provenance.** The whole
feature is a millisecond on a build whose configuration takes minutes, in a
daemon holding several gigabytes.

## 3. What surprised us

**Callback propagation already works.** `Application.reapplyLater` already
wraps user code at registration, so attribution survives every deferred
configuration point tried:

- `tasks.register { }` and `tasks.named { }`
- `withType(...).configureEach { }` and container `configureEach { }`
- `afterEvaluate { }` and `projectsEvaluated { }`
- `pluginManager.withPlugin(...) { }` and `taskGraph.whenReady { }`

Nested plugin application restores correctly too. No new decorator was
needed, and there is no separate "closure problem": coverage is a property of
the registration boundary, not of the kind of user code.

**Interning is the decision that makes this affordable.** A trace of
interned records costs ~4 B per additional record; without interning it is
~80 B per record, and 200,000 properties with 32 records each exhausted the
heap. Anything varying per mutation — an application ID, a call site —
forfeits that.

**Most properties are configured exactly once.** The average configured
property on the Gradle build is mutated 1.18 times, which is why holding the
first record directly and only allocating a trace on the second cut retained
cost from 6.9 MB to 1.2 MB.

**Capture has to be decided when the property is created.** Asking the host
on every mutation is behaviour-neutral but observable: it broke 188 existing
unit tests that assert a property touches no collaborators while being
mutated.

**The stack walk is seven times more expensive than a microbenchmark
suggests.** 7.2 µs average in a real build against ~1 µs measured with user
code adjacent to the capture point, because cost scales with frames skipped
and 22% of captures reach the 50-frame cap without finding a user frame at
all.

**Instrumentation works, and covers 2.8%.** Of 100,485 located mutations,
2,817 came from instrumented build logic and 75,717 from the walk. Two
causes: the interceptor currently matches only `set`, which is 27.5% of
mutations against 69% conventions; and more fundamentally Gradle's own
distribution is not instrumented and performs most of the mutating.

## 4. What does not work

**Only project-scope properties are tracked.** 64% of properties and 65% of
mutations are invisible, because a tracking host exists only at project
scope. That population is not uniform: roughly half is internal machinery
nobody would want attributed (attribute containers, the managed object
registry), a third is value isolation re-creating properties, and the
remainder is real user-facing configuration created from a settings-scope or
otherwise non-project object factory. Only the last part is worth closing.

**The configuration cache erases provenance.** A task property observed at
execution has none, on the store run as well as on a hit, because the
property is recreated during deserialization rather than by the user code
that configured it. Any build using the cache sees nothing.

**Attribution can be confidently wrong.** Where propagation does not reach,
the mutation is attributed to whoever ran it rather than left unattributed:

| Situation | Attributed to |
|---|---|
| A plugin's own thread or executor | `Unknown` |
| User code a plugin stores itself and runs later | whoever ran it |
| A property mutated as a side effect inside a `Provider` transform | whoever evaluated it |

Only the first is visible. On the Gradle build most unattributed mutations
come from one plugin mutating properties from its own asynchronous compiler
runner, so this is not hypothetical.

**Init and settings scripts are misclassified** as the build author, because
`UserCodeSource.Script` carries no role.

**Groovy gets no call sites.** Both `prop = "x"` and `prop.set(x)` compile
through dynamic dispatch, so there is no JVM call site to rewrite. Kotlin
DSL scripts and Java/Kotlin plugins do get them.

**`ConfigurableFileCollection` is not covered**, as it does not share the
property mutation boundary.

## 5. Open decisions

Ordered by how much they matter.

1. **Persist across the configuration cache.** Without it the feature is
   blank in any build that uses the cache, which is an increasing majority.
2. **Track in more scopes.** Settings and build-tree hosts would close the
   part of the 64% that is real user configuration.
   `UserCodeApplicationContext` is cross-build-session scoped and needs no
   project, so this is mostly wiring; the obstacle is that the origin
   registry is build-tree scoped while the global host is not.
3. **Decide what a contribution model needs.** The trace granularity is
   already right — each update is one entry, each collection contribution
   its own — but a structural self-update and a replacement record the same
   kind, the 32-record cap has to go, and fail-closed rejection does not
   catch the confidently-wrong case above.
4. **Give `UserCodeSource.Script` a role** rather than a boolean, so an init
   script does not inherit the build author's identity.
5. **Widen the interceptor** to `convention`, `add` and `put`, which is
   mechanical and raises coverage of build-logic mutations.
6. **Decide whether the stack walk earns its place** at all, given that
   instrumentation covers the mutations a user can act on and the walk costs
   ~720 ms to cover the rest.
