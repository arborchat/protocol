# Arbor Protocol

This document is a draft of where Arbor is moving towards, not its current state.
Please do not *yet* use this as a reference for interacting with the Arbor code.

## Version 0.1

Arbor is an application layer protocol layered on top of TCP/IP.

### Message types

Arbor has the following kinds of message:
* WELCOME - 0
* QUERY - 1
* NEW_MESSAGE - 2

All Arbor messages are JSON objects in which the `Type` key indicates the
kind of message.

All Arbor messages end with a newline character after the close of the JSON object.

It is illegal for the JSON representation of a single Arbor message to exceed 65536 bytes,
including the newline character that marks the end of the message.

WELCOME messages inform a client of the basic server state information needed to
join the server and communicate with it.

WELCOME messages contain the following JSON fields:
* `Type` the message, type, should be 0 for WELCOME
* `Root` the server's root message ID
* `Recent` an array of recent message IDs. This array may have any number of elements (including none), but all elements must be string message UUIDs.
* `Major` the major protocol version number of the server
* `Minor` the minor protocol version number of the server

QUERY messages are used by clients to request message information from a server.

QUERY messages contain the following JSON fields:
* `Type` the message type, should be a 1 for QUERY
* `UUID` the id of the message that is being queried

NEW_MESSAGE messages are used by the server to deliver message information to clients
(both in response to queries and when another user sends a new message). They are also
used by the client to send new messages to the server.

NEW_MESSAGE messages contain the following JSON fields:
* `Type` the message type, should be a 2 for NEW_MESSAGE
* `UUID` the id of this message, **only valid in messages from the server**. If a client sends a NEW_MESSAGE to the server with the UUID set, it will be ignored or rejected.
* `Parent` the id of this message's parent message.
* `Content` the string contents of the message
* `Timestamp` the UNIX timestamp when the message was sent by the user who composed it. In this case, the UNIX timestamp is the number of seconds since January 1st, 1970
* `Username` the string name of the user who wrote the message. The server does not authenticate users, so this should be treated as a hint of the origin of a message, rather than a reliable source

### Procedure

When a TCP connection is established with an an Arbor server, the
server sends a WELCOME message to the client.

The client is then responsible for issuing QUERY messages to fetch the root message and
the recent messages that allow it to build a message tree.

To send a reply to an existing message, a client composes a NEW_MESSAGE and sets the `Parent`,
`Contents`, `Timestamp`, and `Username` fields. It then sends this message to the server.

When the server receives a NEW_MESSAGE, it assigns it a `UUID` and then sends it as a NEW_MESSAGE
to all clients (including the one that created it).
