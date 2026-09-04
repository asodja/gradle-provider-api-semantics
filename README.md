# Provider API Semantics

This repository contains a semantic design for a lazy Provider and Property
API. Its foundations are deliberately close to Gradle's current public model,
while proposing targeted changes and extensions. It focuses on rules
programmers can predict from the API rather than compatibility constraints or
internal implementation details.

## Contents

- [Documents](#documents)
- [Reading order](#reading-order)
- [Shared design principles](#shared-design-principles)

## Documents

### [Provider API Foundations](PROVIDER_API_FOUNDATIONS.md)

The basic semantic model:

- what `Provider<T>` and `Property<T>` represent;
- present, missing, unconfigured, null, and empty;
- explicit configuration state and the convention plan;
- monotonic explicit configuration through `set`;
- convention replacement without `unset` or resurrection through ordinary
  selection;
- current initial values and differences relative to the upcoming Gradle 9.8.0
  release;
- why an explicit missing provider does not fall through to a convention;
- primitive `map`, `flatMap`, `zip`, and `orElse` operations;
- observation, lifecycle, diagnostics, and the base implementation model.

Start here. Every other document builds on these selection and provider
semantics.

### [Property Provenance](PROPERTY_PROVENANCE.md)

The semantic attribution model shared by ordinary and collaborative
properties:

- distinguishes mutation attribution, effective provenance, chronological
  mutation history, and Provider dependencies;
- separates stable contributor identity from diagnostic origin, runtime
  application identity, source location, and semantic operation;
- defines effective provenance as the selected source followed by the
  structural updates that still determine the Provider plan;
- defines mutation history as a separate oldest-first diagnostic view;
- specifies problem-focused console output without exposing property values;
- requires explicit contributor authority and a complete update trace for
  collaborative properties.

### [Property Provenance Implementation](PROPERTY_PROVENANCE_IMPLEMENTATION.md)

Implementation notes and measured evidence from the Gradle prototype:

- captures attribution at the `AbstractProperty` mutation boundary through the
  existing property host and user-code application context;
- documents callback coverage, record interning, and the single-record storage
  optimization;
- reports the measured time and memory costs of contributor and call-site
  provenance;
- records current gaps in scope coverage, configuration-cache persistence,
  script roles, structural operation classification, and callback authority;
- outlines the next representation needed for effective provenance and
  collaborative validation.

### [Collaborative Property Mode](COLLABORATIVE_PROPERTY_UPDATES.md)

An optional `Property` binding mode for Declarative Gradle Reactive Plugins:

- separates declarative source selection from attributed plugin updates;
- recognizes self-assignment through `map`, `flatMap`, `zip`, and operations
  such as `p = p.plus(v)`;
- records each update's contributor and validates the complete trace against
  the property's effective order before observation or lifecycle closure;
- rejects unattributed mutation and unauthorized plugin replacement;
- immediately composes accepted updates using the existing Provider algebra;
- allows constraints to be declared while the property remains mutable;
- allows value queries to observe the validated, currently composed update
  prefix and keeps the ordinary Property lifecycle controls;
- leaves ordinary Kotlin DSL, Groovy DSL, and imperative-plugin property
  semantics unchanged.

### [Derived Collection Operations](DERIVED_COLLECTION_OPERATIONS.md)

Operations expressed from the foundational primitives:

- provider-valued `plus` and `minus` derived directly from `zip`;
- provider-valued addition and subtraction require both operands to be present;
- `ListProperty` concatenation and value-removal;
- `SetProperty` insertion-ordered union and difference;
- `MapProperty` right-biased merge and key subtraction;
- provider-valued operands;
- `plusAssign` and `minusAssign`;
- corresponding configurable file collection behavior.

“Derived” is preferred to “extended”: these operations do not introduce
another property or convention model. They are definitions built from the
small core.

### [Self-Referencing Property Bindings](SELF_REFERENCE_SPEC.md)

Previous-version assignment and its implementation:

- why `P.set(P.map(f))` can mean `P1 = f(P0)` without a cycle;
- structurally inspectable self-references;
- live convention roots when the target was still unconfigured;
- opaque self-references that remain ordinary cycles;
- substitution, lifecycle, performance, and memory retention;
- why non-self replacement cuts a previous self-derived chain.

## Reading order

```text
Provider API Foundations
    +--> Derived Collection Operations
    +--> Self-Referencing Property Bindings --+
    |         +--> compound collection assignment
    |                                         +--> Collaborative Property Mode
    +--> Property Provenance -----------------+
              +--> Property Provenance Implementation
```

Read the foundations first. The other documents build on that core. Compound
collection assignment also uses the self-reference rule.

## Shared design principles

Across all documents:

- these specification versions make no concurrency guarantees for `Property`;
- property selection happens before provider evaluation;
- conventions resolve configuration priority, not missing provider results;
- missing is a state and null is not a value;
- immutable provider plans are separate from mutable property bindings;
- derivation is lazy and non-mutating;
- two-input derivation uses `zip` and preserves both dependencies;
- derived operations are defined from the smallest useful primitive set;
- structurally self-referential assignment captures a previous plan, not a
  realized value;
- mutation provenance keeps stable contributor identity separate from runtime
  application identity, source location, and Provider producer metadata;
- Gradle-managed callbacks retain the causal attribution active when the user
  code was registered;
- collaborative properties keep source selection separate from one
  incrementally composed Reactive Plugin update pipeline;
- callback update order must conform to each property's effective contributor
  order;
- diagnostics and provenance are part of observable behavior.

The words **must**, **must not**, **should**, and **may** are normative in the
specification documents.
