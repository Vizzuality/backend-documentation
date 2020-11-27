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

