# Inaccessible

<h2>for removing fields and types from a supergraph</h2>

```raw html
<table class=spec-data>
  <tr><td>Status</td><td>Release</td>
  <tr><td>Version</td><td>0.1</td>
</table>
<link rel=stylesheet href=https://specs.apollo.dev/apollo-light.css>
<script type=module async defer src=https://specs.apollo.dev/inject-logo.js></script>
```

This document defines a [core feature](https://specs.apollo.dev/core) named `inaccessible` for removing fields and types from a supergraph. "Types" will be used throughout this document to refer to Object, Interface, and Union types in GraphQL.

This specification provides machinery to remove fields and types in a *supergraph* which have the `@inaccessible` directive applied.

# How to read this document

This document uses [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) guidance regarding normative terms: MUST / MUST NOT / REQUIRED / SHALL / SHALL NOT / SHOULD / SHOULD NOT / RECOMMENDED / MAY / OPTIONAL.

## What this document isn't

This document specifies only the processing of supergraphs. The mechanics of field and type omission are not specified normatively here. Conforming implementations may choose any approach they like, so long as the result conforms to the requirements of this document.

# Definitions

## API Schema

This specification makes references to an **API schema**. An API schema is a transformed **supergraph**, intended to be served by a [Consumer](https://specs.apollo.dev/join/v0.1/#sec-Actors) as the graph which is queryable by clients. Generally, an API schema can be loosely imagined as a subset of its supergraph (though this is not a requirement). For example, a client has no use for knowledge of federation machinery like the `join__Graph` enum, and as such it is omitted from API schemas.

# Example: Private User Data

*This section is non-normative.*

We'll refer to this example of a supergraph with user data throughout the document:

:::[example](./supergraph.graphql) -- Supergraph example

The supergraph above contains both a field and type marked as `@inaccessible`. As decided by the schema author, these should not end up in the [API schema](#sec-API-Schema). Notice what has been removed in the API schema below as a result of transformation of the supergraph above:
* `User.id` field
* `BankAccount` type
* `User.bankAccount` field (since it _returns_ the `BankAccount` type)

:::[example](./apiSchema.graphql) -- Resulting API schema

# Overview

*This section is non-normative.* It describes the motivation behind the directives defined by this specification.

An *API schema* is the queryable graph served by a [Consumer](https://specs.apollo.dev/join/v0.1/#sec-Actors). Various use cases require that fields and types exist within subgraphs (often as an implementation detail to Federation, i.e. certain `@key` fields) but can not be queried for by clients. In these cases, the Consumer requires knowledge of their existence via the supergraph, but must treat operations requesting these fields and types as invalid.

# Basic Requirements

Schemas using the `inaccessible` core feature must be valid [core schema documents](https://specs.apollo.dev/core/v0.1) with *@core directives* referencing the `core` specification and this specification.

Here is an example `@core` usage:

:::[example](./coreDirectives.graphql) -- @core directives for supergraphs

As described in the [core schema specification](https://specs.apollo.dev/core/v0.1/#sec-Prefixing), your schema may prefix the `@inaccessible` directive by including an `as` argument to the `@core` directive which references this specification. All references to `@inaccessible` in this specification MUST be interpreted as referring to names with the appropriate prefix chosen within your schema.

In order to use the directive described by this specification, GraphQL requires you to include the definition in your schema.

:::[definition](inaccessible.spec.graphql)

* Processors MUST validate that you have defined the directive with the same locations and no arguments, as provided above.
* Processors MUST remove all fields with an `@inaccessible` directive applied.
* Processors MUST remove all Object, Interface, and Union types with an `@inaccessible` directive applied.
* Processors MUST remove all fields which return an Object, Interface, or Union type with an `@inaccessible` directive applied.
* Processors SHOULD remove the `@inaccessible` directive definition, provided there are no usages remaining in the API schema after the previous steps.