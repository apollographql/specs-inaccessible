# Inaccessible

<h2>for removing fields and types from a core schema</h2>

```raw html
<table class=spec-data>
  <tr><td>Status</td><td>Release</td>
  <tr><td>Version</td><td>0.1</td>
</table>
<link rel=stylesheet href=https://specs.apollo.dev/apollo-light.css>
<script type=module async defer src=https://specs.apollo.dev/inject-logo.js></script>
```

This document defines a [core feature](https://specs.apollo.dev/core) named `inaccessible` for removing fields and types from a core schema. "Types" will be used throughout this document to refer to Object, Interface, and Union types in GraphQL.

This specification provides machinery to remove fields and types in a *core schema* which have the `@inaccessible` directive applied.

# How to read this document

This document uses [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) guidance regarding normative terms: MUST / MUST NOT / REQUIRED / SHALL / SHALL NOT / SHOULD / SHOULD NOT / RECOMMENDED / MAY / OPTIONAL.

## What this document isn't

This document specifies only the processing of a core schema. The mechanics of field and type omission are not specified normatively here. Conforming implementations may choose any approach they like, so long as the result conforms to the requirements of this document.

# Definitions

## Processor

This specification makes references to **Processors**. Processors are described in the [Actors section of the `@core` spec](https://specs.apollo.dev/core/v0.2/#sec-Actors) as an actor which can perform transformations on a core schema. In the case of `@inaccessible`, the Processor will be expected to remove various parts of a core schema.

# Example: Sensitive User Data

*This section is non-normative.*

We'll refer to this example of a core schema with sensitive user data throughout the document:

:::[example](./schema.graphql) -- Core schema example

The schema above contains both a field (`User.id`) and type (`BankAccount`) that are marked as `@inaccessible`. These symbols should be omitted from the processed schema anywhere they would appear. When the processed schema below is generated from this core schema, notice what has been removed:
* `User.id` field
* `BankAccount` type
* `User.bankAccount` field (because it _returns_ the `BankAccount` type)
* `Account` union's `BankAccount` type

:::[example](./processedSchema.graphql) -- Core schema after processing

# Overview

*This section is non-normative.* It describes the motivation behind the directives defined by this specification.

A core schema which has been processed according to the inaccessible spec is a queryable graph, intended to be served by a [Data Core](https://specs.apollo.dev/core/v0.2/#sec-Actors). Various use cases require that fields and types should not be visible to or queried for by clients. The `@inaccessible` directive fulfills this requirement, providing schema authors a mechanism to specify which fields and types should be omitted from the processed schema.

# Basic Requirements

Schemas using the `inaccessible` core feature must be valid [core schema documents](https://specs.apollo.dev/core/v0.2) with *@core directives* referencing the `core` specification and this specification.

Here is an example `@core` usage:

:::[example](./coreDirectives.graphql) -- required @core directives

As described in the [core schema specification](https://specs.apollo.dev/core/v0.2/#sec-Prefixing), your schema may prefix the `@inaccessible` directive by including an `as` argument to the `@core` directive which references this specification. All references to `@inaccessible` in this specification MUST be interpreted as referring to names with the appropriate prefix chosen within your schema.

In order to use the directive described by this specification, GraphQL requires you to include the definition in your schema.

:::[definition](inaccessible.spec.graphql)

## Producer Responsibilities

[Producers](https://specs.apollo.dev/core/v0.2/#sec-Actors) MUST include a definition of the directive compatible with the above definition and all usages in the document.

## Processor Responsibilities

The Processor is responsible for removing all inaccessible elements from the schema output. Note in the `InaccessibleRemoval` algorithm below that because a Union can belong to another Union's set of types, the removal of a Union type may have an upwards "cascading" effect, causing other Unions to become candidates for removal.

InaccessibleRemoval() :
  1. Collect types for removal.
    * Collect all Object, Interface, and Union types marked as `@inaccessible`.
    * Collect all Union types whose types are **all** marked as `@inaccessible`.
      * If any Union types were collected, revisit uncollected Unions for possible removal due to  Union "nesting" (Unions composed of other Unions).
      * Repeat until no more Unions are collected.
  1. Remove all Object, Interface, and Unions collected for removal.
  1. Remove all fields which return an Object, Interface, or Union which was removed.
  1. Remove all Object types which no longer have fields due to prior field removal.
  1. Remove all references to Interface types marked as `@inaccessible`. Object types must have inaccessible Interfaces removed from their list of implementations. (i.e. `type Object implements X & Y { ... }`)
  1. _(optional)_ Remove the `@inaccessible` directive definition, provided there are no usages remaining in the processed schema.
  1. _(optional)_ Remove the `@core` usage which references this spec, provided there are no `@inaccessible` usages remaining in the processed schema and the `@inaccessible` directive definition has been removed.