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

## Version 0.1

Arbor is an application layer protocol layered on top of TCP/IP.

Arbor exchanges "messages" over TCP. These are protocol messages, only some of which correspond to messages in
the chat message tree.

### Message types

Arbor has the following message types:
* WELCOME - 0
* QUERY - 1
* NEW_MESSAGE - 2

The numbers after the type names are how the types are referenced in the protocol.

All Arbor messages are JSON objects in which the `Type` key indicates the
kind of message.

All Arbor messages end with a newline character after the close of the JSON object.

It is illegal for the JSON representation of a single Arbor message to exceed 65536 bytes,
including the newline character that marks the end of the message.

Arbor message IDs are strings assigned by the server. They MUST be unique in the history of the server.
It is recommended that server implementers use UUIDs or some hash of the message contents and metadata.
It is illegal to use the empty string as a message ID **except** as the parent message ID of a server's
root message.

#### WELCOME

WELCOME messages inform a client of the basic server state information needed to
join the server and communicate with it.

WELCOME messages contain the following JSON fields:

- `Type` (integer) the message, type, should be 0 for WELCOME
- `Root` (string message ID) the server's root message ID
- `Recent` (array of string message IDs) an array of recent message IDs. This array may have any number of elements (including none), but all elements must be string message IDs.
- `Major` (integer) the major protocol version number of the server
- `Minor` (integer) the minor protocol version number of the server

A sample WELCOME message looks like this:

```json
{"Type":0,"Root":"f4ae0b74-4025-4810-41d6-5148a513c580","Recent":["92d24e9d-12cc-4742-6aaf-ea781a6b09ec","880be029-0d7c-4a3f-558d-d90bf79cbc1d"],"Major":0,"Minor":1}
```

#### QUERY

QUERY messages are used by clients to request message information from a server.

QUERY messages contain the following JSON fields:

- `Type` (integer) the message type, should be a 1 for QUERY
- `UUID` (string message ID) the id of the message that is being queried

A sample QUERY message looks like this:

```json
{"Type":1,"UUID":"f4ae0b74-4025-4810-41d6-5148a513c580"}
```

#### NEW_MESSAGE

NEW_MESSAGE messages are used by the server to deliver message information to clients
(both in response to queries and when another user sends a new message). They are also
used by the client to send new messages to the server.

NEW_MESSAGE messages contain the following JSON fields:

- `Type` (integer) the message type, should be a 2 for NEW_MESSAGE
- `UUID` (string message ID) the id of this message, **only valid in messages from the server**. If a client sends a NEW_MESSAGE to the server with the UUID set, it will be ignored or rejected.
- `Parent` (string message ID) the id of this message's parent message.
- `Content` (string) the string contents of the message
- `Timestamp` (integer) the UNIX timestamp when the message was sent by the user who composed it. In this case, the UNIX timestamp is the number of seconds since January 1st, 1970 00:00:00 UTC
- `Username` (string) the string name of the user who wrote the message. The server does not authenticate users, so this should be treated as a hint of the origin of a message, rather than a reliable source

A sample NEW_MESSAGE looks like this:

```json
{"Type":2,"UUID":"92d24e9d-12cc-4742-6aaf-ea781a6b09ec","Parent":"f4ae0b74-4025-4810-41d6-5148a513c580","Content":"A riveting example message.","Username":"Examplius_Caesar","Timestamp":1537738224}
```

### Procedure

When a TCP connection is established with an an Arbor server, the
server sends a WELCOME message to the client.

The client is then responsible for issuing QUERY messages to fetch the root message and
the recent messages that allow it to build a message tree. Which messages the client queries
for (or whether it actually queries at all) is up to the client implementer.

To send a reply to an existing message, a client composes a NEW_MESSAGE and sets the `Parent`,
`Contents`, `Timestamp`, and `Username` fields. It then sends this message to the server.

When the server receives a NEW_MESSAGE, it assigns it a `UUID` and then sends it as a NEW_MESSAGE
to all clients (including the one that created it).

### Future Protocol Goals

- Create message metadata for client-side protocol extensions.
- Create recommendations for client implementers.
- Discuss how to control access/authentication/authorization to a given server.
- Use message hashes as message IDs.
- Use multiencoding to describe the wire format in use when connecting to a server.
- Implement error messages.
- Implement tree queries.
- Implement explicit query responses.
- Investigate clustered servers.
- Investigate using IPFS instead of a server.
- Consider protocol level user status (to implement "online"/"away"/"offline" type features).
- Consider more precise timestamps.
- Run Arbor over TLS.
- Consider making immediate replies to the root message special as the "root" of a "conversation"
- Track tree depth as a field on messages. This would enable clients to create placeholders for all of the ancestors of a message before it knew their contents.
- Make a firm decision on whether to support any form of message editing.
