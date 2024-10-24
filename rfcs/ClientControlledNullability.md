# RFC: Client Controlled Nullability

**Proposed by:**

- [Alex Reilly](https://github.com/twof) - Yelp iOS
- [Liz Jakubowski](https://github.com/lizjakubowski) - Yelp iOS
- [Mark Larah](https://github.com/magicmark) - Yelp Web
- [Sanae Rosen](<social or github link here>) - Yelp Android
- [Stephen Spalding](https://github.com/fotoetienne) - Netflix GraphQL Server Infrastructure
- [Wei Xue](https://github.com/xuewei8910) - Yelp iOS
- [Young Min Kim](https://github.com/aprilrd) - Netflix UI

This RFC proposes syntax that would allow developers to override schema-defined
nullability of fields for individual operations.

## Definitions

- **Required field** - A field which is modified with `!` such that a non-null value is required on 
a Nullable or Non-Nullable type.

- **Optional field** - A field which is modified with `?` such that a null value is allowed on a 
Non-Nullable or Nullable type.

## 📜 Problem Statement

In our experience, client developers have been frustrated that the vast majority of fields are
nullable. We’ve done this in accordance with official best practice, and we largely agree that this
is good practice. From the 
[official GraphQL best practice](https://graphql.org/learn/best-practices/#nullability):

> This is because there are many things that can go awry in a networked service backed by databases
and other
> services. A database could go down, an asynchronous action could fail, an exception could be
thrown. Beyond
> simply system failures, authorization can often be granular, where individual fields within 
a request can
> have different authorization rules.

The problem with the SDL Non-Nullable (!) is that it eliminates the possibility of partial failure
on a given type. This forces schema authors to decide for which fields partial failure is
acceptable. A GraphQL schema author 
may not be in the best position to predict whether partial failure will be acceptable or 
unacceptable for every canvas that makes use of a field.

While the schema can have nullable fields for valid reasons (such as federation), in some cases the
client wants to decide if it accepts a `null` value for the result to simplify the client-side
logic.

## 🧑‍💻 Proposed Solution

Client-controlled Non-Nullable and Nullable designators.

## 🎬 Behavior

Each client controlled nullability designator overrides the schema-defined nullability of the field
it's attached to for the duration of the operation.

### `!`
The proposed client-controlled required designator would have identical semantics to the current 
schema-defined 
[Non-Null](https://spec.graphql.org/draft/#sec-Executing-Selection-Sets.Errors-and-Non-Null-Fields). 
Specifically:

  - If during ExecuteSelectionSet() a field **designated required by the operation or** with a
    non-null fieldType raises a field error then that error must propagate to this entire selection
    set, either resolving to null if allowed or further propagated to a parent field.

### `?`
The proposed client-controlled optional designator would have identical semantics to the current 
schema-defined default behavior. Fields that resolve to `null` return `null` for that field with no
additional side effects.

## ✅ Validation

If a developer executed an operation with two fields name `foo`, one a `String` and the other an 
`Int`, the operation would be declared invalid by the server. The same is true if one of the fields
is designated required but both are otherwise the same type. In this example, `someValue` could be
either a `String` or a `String!` which are two different types and therefor can not be merged:

```graphql
fragment conflictingDifferingResponses on Pet {
  ... on Dog {
    someValue: nickname
  }
  ... on Cat {
    someValue: nickname!
  }
}
```

## ✏️ Proposed syntax

The client can express that a schema field is required by using the `!` syntax in the operation
definition:
```graphql
query GetBusinessName($id: String!) {
  business(id: $id) {
    name!
  }
}
```

### `!`

We have chosen `!` because `!` is already being used in the GraphQL spec to indicate that a field in
the schema is Non-Nullable, so it will feel familiar to GraphQL developers.

### `?`

We have chosen `?` because `?` is used in a few other languages (Swift, Kotlin) that have `!` to 
mean something like the opposite of `!`.

## Use cases

### Improve the developer experience using GraphQL client code generators
Handling nullable values on the client is a major source of frustration for developers, especially
when using types generated by client code generators in strongly-typed languages. The proposed
required designator would allow GraphQL clients to generate types with more precise nullability 
requirements for a particular feature. For example, using a GraphQL client like Apollo GraphQL on 
mobile, the following query
```graphql
query GetBusinessName($id: String!) {
  business(id: $id) {
    name!
  }
}
```
would be translated to the following type in Swift.
```swift
struct GetBusinessNameQuery {
  let id: String
  struct Data {
    let business: Business?
    struct Business {
      /// Lack of `?` indicates that `name` will never be `null`
      let name: String
    }
  }
}
```
If a null business name is not acceptable for the feature executing this query, this generated type
is more ergonomic to use since the developer does not need to unwrap the value each time it’s
accessed.

### 3rd-party GraphQL APIs
Marking field Non-Nullable in schema is not possible in every use case. For example, when a
developer is using a 3rd-party API such as 
[Github's GraphQL API](https://docs.github.com/en/graphql) they won't be able to alter Github's
schema, but they may still want to have certain fields be required in their application. Even within
an organization, ownership rules may dictate that an developer is not allowed to alter a schema they
utilize.

## ✅ RFC Goals

- Non-nullable syntax that is based off of syntax that developers will already be familiar with
- Nullable syntax that is based off of syntax that developers will already be familiar with
- Enable GraphQL client code generation tools to generate more ergonomic types

## 🚫 RFC Non-goals
This syntax consciously does not cover the following use cases:

- **Default Values**
  The syntax being used in this proposal causes queries to propagate an error in the case that
  a `null` is found for a required field. As an alternative, some languages provide syntax (eg `??` 
  for Swift) that says "if a field would be `null` return some other value instead". We have not 
  covered that behavior in this proposal, but leave it open to be covered by future proposals.

## 🗳️ Alternatives considered

### A `@nonNull` official directive

This solution offers the same benefits as the proposed solution. Additionally, this solution has 
good upgrade paths if we later want to provide more behavior options to developers. 
[Relay's `@required` directive](https://mrtnzlml.com/docs/relay/directives#required), for example, 
allows developers to decide how they want their clients to respond in the event that `null` is 
received for a `@required` field.

```graphql
fragment Foo on User {
  address @required(action: THROW) {
    city @required(action: LOG)
  }
}
```

With our current proposal, we don't have a great way to offer this kind of flexibility that would 
mesh nicely with existing GraphQL syntax.

However we think the described behavior acts as a nice, concise default, and is worth the tradeoff.

### A `@nonNull` custom directive

This is an alternative being used at some of the companies represented in this proposal.

While this solution simplifies some client-side logic, it does not meaningfully improve the 
developer experience for clients.

* The cache implementations of "smart" GraphQL clients also need to understand the custom directive 
  to behave correctly. Currently, when a client library caches a `null` field based on an operation 
  without a directive, it will return the `null` field for another operation with this directive.
* For clients that rely on client code generation, generated types typically cannot be customized 
  based on a custom directive. See 
  https://github.com/dotansimha/graphql-code-generator/discussions/5676 for an example. As a result, 
  the optional generated properties still need to be unwrapped in the code.

This feels like a common enough need to call for a language feature. A single language feature would 
enable more unified public tooling around GraphQL.

### Make Schema Fields Non-Nullable

It is intuitive that one should simply mark fields that are not intended to be `null` Non-Nullable 
in the schema. For example, in the following GraphQL schema:

```graphql
type Business {
  name: String
  isStarred: Boolean
}
```

If we intend to always have a `name` and `isStarred` for a `Business`, it may be tempting to mark 
these fields Non-Nullable:

```graphql
type Business {
  name: String!
  isStarred: Boolean!
}
```

Marking schema fields Non-Nullable may introduce problems in a distributed environment where partial 
failure is a possibility regardless of whether the field is intended to have `null` as a valid 
state.

When a Non-Nullable field results in `null`, the GraphQL server will recursively step through the 
field’s ancestors to find the next nullable field. In the following GraphQL response:

```json
{
  "data": {
    "business": {
      "name": "The French Laundry",
      "isStarred": false
    }
  }
}
```

If isStarred is Non-Nullable but returns `null` and business is nullable, the result will be:

```json
{
  "data": {
    "business": null
  }
}
```

Even if `name` returns valid results, the response would no longer provide this data. If business is
Non-Nullable, the response will be:
```json
{
  "data": null
}
```

In the case that the service storing user stars is unavailable, the UI may want to go ahead and 
render the component without a star (effectively defaulting `isStarred` to `false`). A Non-Nullable 
field in the schema makes it impossible for the client to receive partial results from the server, 
and thus potentially forces the entire component to fail to render.

More discussion on [when to use Non-Nullable can be found here](https://medium.com/@calebmer/when-to-use-graphql-non-null-fields-4059337f6fc8)

Also see [3rd-party GraphQL APIs](#3rd-party-GraphQL-APIs) for an instance where it wouldn't be 
possible for a developer to alter the schema for a service they're using.

### Write wrapper types that null-check fields
This is the alternative being used at some of the companies represented in this proposal for the 
time being. It's labor intensive and rote work. It more or less undermines any gains from code 
generation.

### Alternatives to `!`
#### `!!`
This would follow the precedent set by Kotlin. It's more verbose and diverges from GraphQL's SDL
precedent.