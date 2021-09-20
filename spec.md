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
# Algorithms

## Is Inaccessible?

Return true if the named element {element} is inaccessible.

IsInaccessible(document, element) :
  1. If {element} is a FieldDefinition, **Return** {IsMarkedInaccessible(document, element)}
  2. For each Definition or Extension {e} in {document},
    - If {e} and {element} have the same Name and {IsMarkedInaccessible(document, e)}, **Return** true
  3. **Return** false

Return true iff the named element {element} is marked as inaccessible.

IsMarkedInaccessible(document, element) :
  1. Let {assignments} be the result of assigning features via {AssignFeatures(document)}
  2. For each Directive {d} on {element},
    1. If {assignments}`[`{element}`]` is a Directive whose `feature` argument is this spec, **Return** {true}
  3. **Return** {false}

## Removed Inaccessible Elements

Given a schema document, return the set of all schema elements which should be removed in the API.

RemoveInaccessible(document) :
  1. For each named schema element {e} defined in the {document},
    1. If {IsInaccessible(document, e)} is {true}, {Remove(document, e)}

Remove(document, element) :
  1. If {element} is a Field or Input Field,
    1. Delete the definition of {element} in {document}
    2. Let {parent} be the parent type of {f} in {document}
      2. If {parent} has no fields, {Remove(document, parent)}
  2. If {element} is a type:
    1. Delete all definitions and extensions of {element} in {document}    
    2. If {element} is an output type (object type, interface type, or union type),
      1. For each Field {f} in {document} where {f} has a Return Type of {element}, {Remove(document, f)}
      2. For each Union {u} in {document} where {element} is a member of {u},
        1. Delete {element} as member of {u}
        2. If {u} has no members, {Remove(document, u)}
      3. If {element} is an interface,
        1. For each type or interface {t} which implements {element} in {document},
          - Delete {element} as an interface conformance of {t}
    3. If {element} is an input type (scalar, enum, or input object type):
      1. For each Field {f} in {document} where {f} has any argument of type {element}, {Remove(document, f)}
      2. For each input object type {t} in {document},
        1. For each input field {f} of {t} where the type of {f} is {element}, {Remove(document, f)}

