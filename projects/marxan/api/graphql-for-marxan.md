# Review of GraphQL for the Marxan cloud platform

Some key points from my deep dive into the GraphQL ecosystem.

First, a quick example. We will create a new Git repository/project on the
[SourceHut platform](https://sr.ht/) via its GraphQL API.

```
$ cat example.json
mutation createRepository($name: String!, $description: String) {
  createRepository(name: $name, visibility: PRIVATE, description: $description) {
    id
    updated
    owner {
      canonicalName
    }
  }
}

$ curl -X POST -H 'Authorization: Bearer <token>' \
    -H 'Content-Type: application/graphql'
    -d @example.json
    'https://git.sr.ht/query?variables={"name": "marxan-cloud-ftw", "description": "Components for the MarxanCloud platform"}' | jq
{
  "data": {
    "createRepository": {
      "id": 12345,
      "updated": "2020-12-09T10:07:00Z",
      "owner": {
        "canonicalName": "~hotzevzl"
      }
    }
  }
}
```

Although this is a very simple example, we can nevertheless observe some key
traits of a typical GraphQL API which are relevant for a review of key
differences from REST-ish APIs.

- In simplest use cases, GraphQL is "just JSON over HTTP"

In practice, client and server libraries are needed to do most of the heavy
lifting (similarly to client and server implementations of the JSON:API spec),
but in essence the interface between GraphQL clients and servers is mostly a
pipeline of JSON documents (requests and response) over HTTP: for very simple
interactions, even plain requests via cURL will work fine.

- Requests typically hit a single "endpoint"

In this example, the endpoint is `https://git.sr.ht/query`.

This acts effectively as an "entry point" to the entire GraphQL API, through
which all the client request are channeled.

Meaningful, consistent and elegant URI design, which is typically a core part of
a REST API, does not really apply to GraphQL. All the naming semantics are
conveyed through the GraphQL payload (above shown as raw GraphQL query, but
typically encapsulated in a JSON file which contains both query *and*
variables).

In a REST API, a request similar to the example above, to create a new
repository, would be done via something like `POST /api/v1/repositories`,
with a JSON payload describing the resource that we are trying to create.

- HTTP is used as a "dumb" transport channel

All requests use a single HTTP verb (typically, `POST`: `GET` may be used to
_query_ data only, not to _mutate_ data), besides using a single "endpoint".

This is a major departure from the consistent use of key HTTP verbs in REST
APIs.

- GraphQL is kind of RPC

In their simplest form, GraphQL queries do indeed feel declarative; for example:

```
query {
  me {
    canonicalName
  }
}
```

However, they can be augmented with _parameters_ which may describe server-side
operations of arbitrary complexity; such as plain filters in the example below
(`repositories(filter: { count: 2 }`):

```
query {
  me {
    canonicalName
    repositories(filter: { count: 2 }) {
      cursor
      results {
        id, name, updated
      }
    }
  }
}
```

Or stateful cursors for paginated result sets such as in the example below:

```
query {
  me {
    canonicalName
    repositories(cursor: "<long string with cursor>") {
      cursor
      results {
        id, name, updated
      }
    }
  }
}
```

In my view, GraphQL allows to hit a sweet spot between the meaningful
manipulation of representations of resources as typically done in REST APIs, and
the potential expressiveness and flexibility of RPC interfaces, yet without
forcing to work within the complexity of more elaborate RPC systems (for those
still having SOAP or XML-RPC nightmares from the 2000s).

- GraphQL enforces strongly typed schemas

Although it is indeed possible to create GraphQL schemas that only use basic
scalar types (such as `String`, `Int`, `Float`, `Boolean`), in practice this
would hardly be useful: most often, expressive types are created and composed.

Union types, interfaces, enums are commonly used to curate expressive API
graphs.

"Making impossible states impossible" through strong typing is indeed a major
advantage of the GraphQL query language.

In the example above, the `createRepository()` mutation accepts one required
string parameter (`name`) and one optional string parameter (`description`),
and makes use of an enum type for the `visibility` parameter.

Query payloads that don't validate against the schema are outright rejected (and
GraphQL tooling can help to avoid even creating invalid payloads, especially if
coupled with TypeScript typing).

- Caching is... different

A major point for GraphQL skepticism, but likely in practice not all that
relevant in projects like the Marxan cloud platform.

GraphQL queries (as opposed to mutations and subscriptions) could indeed be
cached by using the `GET`-verb endpoint, companion to the `POST` one normally
used (with some caveats).

In practice, for authenticated GraphQL APIs where query results may be different
for each authenticated user, even this may not be a major issue, especially
when combined with application-level caching strategies, which are available at
different levels of granularity (from whole-request down to individual fields).

Several common caching strategies are implemented in major GraphQL client and
server implementations, making it possible to rely on existing best practices
and keeping development complexity under control.

Moreover, expensive queries that need to be exposed to non-authenticated users
could also be made available over ad-hoc REST endpoints, thus allowing to enjoy
the benefits of HTTP-layer gateway caching.

## Some more points to consider

More traits of GraphQL API practices cannot be gleaned from the initial example
above. I am briefly outlining here the ones that are, in my opinion, most
relevant to the Marxan cloud platform scenario.

- Authentication is often kept as a REST subsystem

This seems to be common practice, and allows to use different established
authentication strategies (e.g. password authentication, 2FA, OAuth2, etc.).

Once a token or key of some sort is obtained, this can be used in HTTP headers
as commonly done in `Bearer` type authorization strategies or equivalent.

- Introspection augments the type system

In practice, this is often used to provide functionality similar to what OpenAPI
interfaces do; in the case of GraphQL, documentation of the whole API graph is
consistently done via introspection, so the GraphQL schema serves both as
parsing, validation and query processing layer as well as documentation layer
(the same may be true in some OpenAPI server implementations, in practice).

Documentation on *how to use* a GraphQL API is typically still needed like with
most non-trivial APIs, although the type system combined with introspection and
meaningful naming of queries, mutations and parameters provides a convenient
core documentation layer.

- Authorization

Fine-grained access control is pretty much equally doable via REST and GraphQL.

For example, only allowing users with a specific role to set or modify specific
fields, or to set them to specific values under specific circumstances, or to
query specific fields.

GraphQL may make some access control rules more expressive (for example, by
using access decorators in the schema definition) and easier to understand at a
glance, although I would argue this is mostly a tooling concern as well as a
matter of preference.

- N+1 problem

GraphQL resolvers by default typically traverse a query graph node by node,
depth-first and may process each node in isolation, which could easily lead to
very inefficient database queries (defaults are actually up to server
implementations - this would be a worst-case scenario).

GraphQL server implementations may provide, either as core functionality or
through a library/module, lazy loading/async/batching functionality which
essentially traverses the whole query graph (or parts of it), compiling a query
plan and eventually executing this to load data from database through the least
possible number of queries (simplifying things a bit here).

This pattern ("DataLoader") may limit the ability to compute/estimate cost of
individual resolvers, so in case of very complex query graphs, things will still
need to be optimised for a sane balance between query performance and the
ability to constrain queries whose computational cost is considered unacceptable
for the specific context.

- API Versioning

A common strategy seems to be evolutive versioning, or basically deprecating
affected fields (the GraphQL schema description language provides a
`@Deprecated` directive for this) and introducing their eventual replacements,
while keeping the deprecated ones in place until it is possible/reasonable to
break existing API contracts.

- Errors

Besides sidestepping the semantics of HTTP verbs besides `GET` and `POST`,
GraphQL also handles errors differently than via HTTP status codes. In practice,
client and server implementations take care of error handling similarly to how
HTTP status codes would be handled by REST frameworks. Additionally, GraphQL
error handling can be granular, for example surfacing different kinds of errors
as they apply to different parts of a query.

- Federation

GraphQL APIs for distinct microservices can be federated at the API layer itself
(e.g. via Apollo Federation), essentially providing a GraphQL-focused gateway
that allows clients to see a single GraphQL API even though parts of its
functionality may be provided by independent microservices.

We didn't use nor needed this functionality in Wildlife Insights, but as we are
likely going to rely on a small set of microservices for data processing
operations, this could be a useful feature for the Marxan cloud platform. We
could of course equally do some ad-hoc lightweight proxying of REST requests in
the API.
