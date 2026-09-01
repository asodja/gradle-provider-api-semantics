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

### [Collaborative Property Mode](COLLABORATIVE_PROPERTY_UPDATES.md)

An optional `Property` binding mode for Declarative Gradle Reactive Plugins:

- separates declarative source selection from attributed plugin updates;
- recognizes self-assignment through `map`, `flatMap`, `zip`, and operations
  such as `p = p.plus(v)`;
- validates each update against global contributor order and per-property
  constraints;
- rejects unattributed mutation and unauthorized plugin replacement;
- immediately composes accepted updates using the existing Provider algebra;
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
    |
    +--> Derived Collection Operations
    |
    +--> Self-Referencing Property Bindings
    |         |
    |         +--> compound collection assignment
    |
    +--> Collaborative Property Mode
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
- collaborative properties keep source selection separate from one
  incrementally composed Reactive Plugin update pipeline;
- callback update order must conform to global contributor order and explicit
  property constraints;
- diagnostics and provenance are part of observable behavior.

The words **must**, **must not**, **should**, and **may** are normative in the
specification documents.
