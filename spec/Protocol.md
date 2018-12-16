# Arbor Protocol

This document describes the current state of the Arbor protocol. Arbor is experimental and subject to
change. However, the reference server should adhere to this specification.

## Introduction

Arbor is a chat protocol that represents conversation as a tree of messages instead of a linear sequence
of messages. This allows conversation to diverge naturally into several subconversations in a way that is
comprehensible.

To facilitate chat as a tree, Arbor assigns an identifier to every message. Messages are only valid so long
as they are a reply to a previous message. Every new chat message has a reference to the identifier of its
parent message and its own identifier (so that others can reply to *it*).

When an Arbor server starts, there are no messages. Since all messages must be replies to existing messages,
the server creates a special "root" message that has no parent message. Conversation on that server will
start as replies to the root message.

One of Arbor's most important design goals is simplicity. The project will always favor simpler design over
"advanced" features. It is especially important for the server side of the protocol to be simple, as that
facilitates small, easily-auditable server implementations. The current reference server is only 300 lines of Go.

Wherever possible, Arbor will defer responsibilities to existing systems. For instance, it is planned to use
PGP message signing to validate messages from other users rather than having the server authenticate users.
While this does incur overhead, it allows the server to be simpler and re-uses an existing (widely-used)
trust infrastructure instead of building another buggy one.

Security is a design goal, but isn't remotely implemented yet. Arbor currently transmits everything in plaintext.
This is due to a current focus on proving that modeling chat as a tree is actually a good idea. Once that is
established, much more emphasis will be placed on hardening the system.

Parts of this version of the protocol were inspired by [Dendros](https://gitlab.com/tekktonic/dendros/tree/master),
a similar protocol under development by Daniel Wilkins.

## Version 0.2

Arbor is an application layer protocol layered on top of TCP/IP.

### Vocabulary

A `Message` is a protocol-layer object passed between two hosts speaking the Arbor Protocol. It may or may not
contain a `Post`.

A `Post` is a reply within Arbor's tree of communication.

A `Root Post` is a special `Post` created by a `Server` so that `Client`s always have a legal `Post` to reply to.

A `Server` is an program that connects multiple `Client`s, creates the `Root Post`, and assigns `Post` `id`s.

A `Client` connects to a server and exchanges `Message`s.

### Posts

Posts have the following structure:

```json
{
  "id": "0a1a6f7e-00c6-11e9-ac65-5f4353e5d54b", // some UUID
  "parent": "e2d82da2-00c5-11e9-b301-ffe6e87053f7", // some UUID
  "author": "Examplius Caesar", // some UTF8-string
  "timestamp": 1544918797681875713, // integer, nanoseconds since midnight, January 1, 1970, UTC
  "content": "I am an example", // some UTF8-string
  "meta": { // arbitrary key-value store
    "key": "value"
  }
}
```

To describe each field in detail:

- `id`: This is a unique identifier for the `Post` that is assigned by the `Server`.
  - May contain any subset of legal characters in the Base64 encoding.
  - Maximum length 128 bytes.
  - Must not be the empty string.
- `parent`: This is the unique identifier of the `Post` that this `Post` is a reply to.
  - May contain any subset of legal characters in the Base64 encoding.
  - Maximum length 128 bytes.
  - Must not be the empty string.
  - For a `Root Post`, the `parent` field should be `""` (the empty string).
- `author`: This is the name of the user who created the `Post`.
  - Must be valid UTF-8.
  - May not contain control characters.
  - Maximum length 64 bytes.
- `timestamp`: This is the time at which a message was sent.
  - Must be a positive number.
  - Must be an integer.
  - Must measure nanoseconds since the start of the UNIX epoch (Jan 1, 1970, 00:00:00 UTC)
  - Should be represented by implementations as an unsigned 64-bit integer.
  - Maximum representable time is in the year 2554.
- `content`: The actual content that the user posted.
  - Must be valid UTF-8.
  - Must not be the empty string.
  - Must not contain nothing but whitespace characters.
  - Must not contain control characters.
  - Maximum length 4096 bytes.
- `meta`: Arbitrary post metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.

The length requirements above mean that the maximum data size (not including JSON formatting characters)
of any `Post` is 41288 bytes.

### Messages

Arbor has the following message types:

| Name        | Representation |
| ----------- | ---            |
| WELCOME     | 0              |
| QUERY       | 1              |
| NEW         | 2              |
| RESPONSE    | 3              |
| META        | 4              |

The numbers after the type names are how the types are referenced in the protocol.

All Arbor messages are JSON objects in which the `type` key indicates the
kind of message.

All Arbor messages end with a newline character after the close of the JSON object.

It is illegal for the JSON representation of a single Arbor message to exceed 65536 bytes,
including the newline character that marks the end of the message.

Arbor message IDs are strings assigned by the server. They MUST be unique in the history of the server.
It is recommended that server implementers use UUIDs or some hash of the message contents and metadata.

Each `Message` type has a different schema, so each is described in a section below:

#### WELCOME

WELCOME messages inform a client of the basic `Server` state information needed to
join the `Server` and communicate with it.

WELCOME messages contain the following JSON fields:

- `type` (integer) the message type, should be 0 for WELCOME.
- `root` (string post ID) the server's `Root Post` `id`.
- `leaves` (array of string message IDs) an array of recent message IDs.
  - This array may have any number of elements (including none), but all elements must be string message IDs.
  - This array should only include messages for which the `Server` has no known reply. This keeps the array smaller.
  - The server may, at any time, shrink this array by whatever heuristic it desires. A recommended heuristic would
    be to remove the `id`s of `Post`s that have not been replied to within 24 hours.
- `major` (integer) the major protocol version number of the server. (For this specification, 0)
- `minor` (integer) the minor protocol version number of the server. (For this specification, 2)
- `meta`: Arbitrary metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.

A sample WELCOME message looks like this:

```json
{
  "type": 0,
  "root": "f4ae0b74-4025-4810-41d6-5148a513c580",
  "leaves": [
    "92d24e9d-12cc-4742-6aaf-ea781a6b09ec",
    "880be029-0d7c-4a3f-558d-d90bf79cbc1d"
  ],
  "major": 0,
  "minor": 2,
  "meta": {
    "key": "value"
  }
}
```

#### QUERY

QUERY messages are used by clients to request post information from a server.

QUERY messages contain the following JSON fields:

- `type` (integer) the message type, should be a 1 for QUERY
- `ids` (array of string post `id`s) the requested `Post` `id`s
  - At most 1024 `Post` `id`s may be included in a single `QUERY`.
- `meta`: Arbitrary metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.


A sample QUERY message looks like this:

```json
{
  "type": 1,
  "ids": [
    "f4ae0b74-4025-4810-41d6-5148a513c580",
    "3397ae9e-00ca-11e9-ac8d-57df8dd16d7f"
  ],
  "meta": {
    "key": "value"
  }
}
```

#### NEW

NEW messages are used by the server to deliver post information to clients. They are also
used by the client to send new messages to the server. In contrast to the version 0.1
protocol, NEW messages are not used to respond to QUERY messages.

NEW messages contain the following JSON fields:

- `type` (integer) the message type, should be a 2 for NEW
- `posts` (array of posts), each of which is an object with the JSON schema described in the [Posts](#Posts) section.
  - NOTE: the `id` field is only valid if the message is from a server. If a client is sending new posts to the
    server, it should be left blank. Servers may choose to accept, ignore, or reject clients that send NEW messages
    that already contain post Ids.
- `meta`: Arbitrary metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.

A sample NEW looks like this:

```json
{
  "type": 2,
  "posts": [
    {
      "id": "0a1a6f7e-00c6-11e9-ac65-5f4353e5d54b",
      "parent": "e2d82da2-00c5-11e9-b301-ffe6e87053f7",
      "author": "Examplius Caesar",
      "timestamp": 1544918797681875713,
      "content": "I am an example",
      "meta": {
        "key": "value"
      }
    },
    {
      "id": "3397ae9e-00ca-11e9-ac8d-57df8dd16d7f",
      "parent": "0a1a6f7e-00c6-11e9-ac65-5f4353e5d54b",
      "author": "Brutus",
      "timestamp": 1544921192493452774,
      "content": "This is an honorable example",
      "meta": {
        "key": "value"
      }
    }
  ],
  "meta": {
    "key": "value"
  }
}
```

#### RESPONSE

RESPONSE messages are used by the server to deliver the answers to QUERY messages.

RESPONSE messages contain the following JSON fields:

- `type` (integer) the message type, should be a 3 for RESPONSE
- `posts` (array of posts), each of which is an object with the JSON schema described in the [Posts](#Posts) section.
- `meta`: Arbitrary metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.

A sample RESPONSE looks like this:

```json
{
  "type": 3,
  "posts": [
    {
      "id": "0a1a6f7e-00c6-11e9-ac65-5f4353e5d54b",
      "parent": "e2d82da2-00c5-11e9-b301-ffe6e87053f7",
      "author": "Examplius Caesar",
      "timestamp": 1544918797681875713,
      "content": "I am an example",
      "meta": {
        "key": "value"
      }
    },
    {
      "id": "3397ae9e-00ca-11e9-ac8d-57df8dd16d7f",
      "parent": "0a1a6f7e-00c6-11e9-ac65-5f4353e5d54b",
      "author": "Brutus",
      "timestamp": 1544921192493452774,
      "content": "This is an honorable example",
      "meta": {
        "key": "value"
      }
    }
  ],
  "meta": {
    "key": "value"
  }
}
```

#### META

META messages are used to pass around arbitrary data. Clients or Servers can use them for any
purpose, but the protocol specification defines no uses for them.

META messages contain the following JSON fields:

- `type` (integer) the message type, should be a 4 for META
- `meta`: Arbitrary metadata intended to facilitate protocol extensions.
  - All keys must contain only characters valid in Base64.
  - Maximum key length is 128 bytes.
  - All values must be valid UTF-8 strings.
  - Values may not contain control characters.
  - Maximum value length is 1024 bytes.
  - Maximum of 32 key-value pairs.


A sample QUERY message looks like this:

```json
{
  "type": 4,
  "meta": {
    "presence": "Examplius Caesar" // hypothetical user "join server" protocol extension
  }
}
```

### Procedure

When a TCP connection is established with an an Arbor server, the
server sends a WELCOME message to the client.

The client is then responsible for issuing QUERY messages to fetch the root post and
the recent posts that allow it to build a post tree. Which posts the client queries
for (or whether it actually queries at all) is up to the client implementer.

To send a reply to an existing post, a client composes a NEW and sets the `parent`,
`contents`, `timestamp`, and `author` fields. It then sends this message to the server.

When the server receives a NEW, it assigns all posts in `posts` an `id` and then sends them
all as a NEW to all clients (including the one that created it).

When the server receives a QUERY, it attempts to find the details for all post `id`s in the
`ids` field. This may include querying local storage or attached clients. It sends RESPONSE
messages as it finds answers to the QUERY. A single QUERY may have many responses. If the
server fails to find the details of a post, it is not obligated to send a RESPONSE.

If a client receives a QUERY from the server, it MAY opt to answer the QUERY with a RESPONSE
message if it has the details of the relevant post available. (This is trusting clients
too much, and will improve once message IDs are cryptographic hashes).

### Future Protocol Goals

- Create recommendations for client implementers.
- Discuss how to control access/authentication/authorization to a given server.
- Use message hashes as message IDs.
- Use multiencoding to describe the wire format in use when connecting to a server.
- Implement error messages.
- Implement tree queries.
- Investigate clustered servers.
- Investigate using IPFS instead of a server.
- Consider protocol level user status (to implement "online"/"away"/"offline" type features).
- Run Arbor over TLS.
- Consider making immediate replies to the root message special as the "root" of a "conversation"
- Track tree depth as a field on messages. This would enable clients to create placeholders for all of the ancestors of a message before it knew their contents.
- Make a firm decision on whether to support any form of message editing.
