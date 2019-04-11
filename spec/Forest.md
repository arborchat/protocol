# The Arbor Forest Specification

Arbor models chat conversations as a group of Tree data structures.
This particular collection of trees is called the "Arbor Forest", and
this document specifies its structure.

## Definitions

Node - any element in one of the trees in the arbor forest.
Id - the cryptographic hash of the data in a Node is its Id. Each node type specifies the exact method needed to compute its Id correctly.

## Nodes

There are four types of Node:

- Identity: these Nodes represent a user of Arbor and consist primarily of a public/private key pair and a human-readable name.
- Community: these Nodes serve as the root node for a group of users' conversations.
- Conversation: these Nodes are the direct children of Community Nodes and are the root of a subtree of related Reply Nodes.
- Reply: these Nodes are the children of Conversation Nodes, which each Reply representing a single message from an Arbor user.

### Field Types

The Arbor Forest uses a number of common field types with specific meanings. These types are defined here and will be used to describe each field below.

- Node Type: an 8-bit unsigned integer indicating the type of this Node. Possible values are:
  - 1: Identity
  - 2: Community
  - 3: Conversation
  - 4: Reply
- Hash Type: an 8-bit unsigned integer representing a particular hash algorithm. Possible values:
  - 0: meaning depends on Node Type, usually indicates that the hash field is irrelevant
  - 1: SHA1
  - 2: SHA256
- Hash Digest Length: an 8-bit unsigned integer indicating how many bytes long the output digest of a particular hash function is.
- Hash Value: the actual hash value from some particular hash algorithm with some particular digest size.
- Node Id: A Hash Value that is a reference to another node in the Forest
- Tree Depth: an unsigned 32-bit integer indicating how many levels beneath the root of a particular tree a given node is.
- Content Type: an 8-bit unsigned integer representing a particular content structure (analagous to a MIME type). Valid values are:
  - 0: binary, unknown
  - 1: UTF8 text
  - 2: JSON
- Content Length: an unsigned 16-bit integer representing how many bytes are in another field.
- Blob: binary data with an unspecified length. All Blob fields will have an associated Content Length and Content Type field to clarify their semantics.
### Common Fields

All nodes in the forest share some fields. These are described generally here, though each node type's description may contain more detailed information about the legal values in each field for that node type.

Common fields:

- `node_type` (Node Type)
- `parent_id_type` (Hash Type): the type of the hash algorithm for the `parent_id` field
- `parent_id_bytes` (Hash Digest Length): the number of bytes in the `parent_id` field
- `parent_id` (Node Id): the hash of the parent tree node. For identity nodes, this will always be the null hash (of all zeroes)
- `id_type` (Hash Type): type type of the hash algorithm that should be used to compute this node's id
- `id_bytes` (Hash Digest Length): how many bytes should this node's id be
- `depth` (Tree Depth): the number of levels this node is from the root message in its tree.
- `metadata_type` (Content Type): the type of the `metadata` field
- `metadata_bytes` (Content Length): number of bytes in the `metadata` field
- `metadata` (Blob): arbitrary binary data
- `signature_authority_type` (Hash Type): the type of hash algorithm used to compute the Id of the node that is the signature authority for this node. The valid values are the same as for `parent_id_type`.
- `signature_authority_bytes` (Content Length): the number of bytes in the `signature_authority` field
- `signature_authority` (Node Id): the id of the Identity node that signed this node
- `signature_bytes` (Content Length): the quantity of bytes in the `signature` field
- `signature` (Blob): the actual binary signature of the node. The structure of this field varies by the type of key in the `signature_authority`.

### Identity

An identity node has the following fields:

- `name_bytes` (Content Length): the number of bytes of UTF8 text in the `name` field
- `name` (Blob): the name of the user who controls this key
- `key_type` (Key Type): an unsigned 16-bit integer representing the kind of public key in use. Possible values are:
  - 1: OpenPGP
- `key_bytes`(Content Length): the unsigned 16-bit integer quantity of bytes of data in the `public_key` field
- `public_key` (Blob): the binary representation of the public key
 
### Community

A Community node has the following fields:

- `name_bytes` (Content Length): the number of bytes of UTF8 text in the `name` field
- `name` (Blob): the name of the user who controls this key

### Conversation

A Conversation node has the following fields:

- `content_type` (Content Type)
- `content_bytes` (Content Length)
- `content` (Blob)

### Reply

A Conversation node has the following fields:

- `conversation_id_type` (Hash Type)
- `conversation_id_bytes` (Hash Digest Length)
- `conversation_id` (Node Id)
- `content_type` (Content Type)
- `content_bytes` (Content Length)
- `content` (Blob)

