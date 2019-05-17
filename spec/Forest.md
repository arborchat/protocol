# The Arbor Forest Specification

Arbor models chat conversations as a group of Tree data structures.
This particular collection of trees is called the "Arbor Forest", and
this document specifies its structure.

## Definitions

Node - any element in one of the trees in the arbor forest.
Id - the cryptographic hash of the data in a Node is its Id. Each node type specifies the exact method needed to compute its Id correctly.
Root Node - any Node with no parent (parent Id is the Null Hash)
Conversation Node - a Reply node that is the direct child of a Community. It is the root of a subtree of Reply nodes, and can be identified by being at Depth 1 and having the Null Hash as a `conversation_id`.

## Nodes

There are three types of Node:

- Identity: these Nodes represent a user of Arbor and consist primarily of a public/private key pair and a human-readable name.
- Community: these Nodes serve as the root node for a group of users' conversations.
- Reply: these Nodes are the children of either Community Nodes or other Reply Nodes, with each Reply representing a single message from an Arbor user.

### Field Types

The Arbor Forest uses a number of common field types with specific meanings. These types are defined here and will be used to describe each field below.

- **Node Type**: an 8-bit unsigned integer indicating the type of this Node. Possible values are:
  - 1: Identity
  - 2: Community
  - 3: Reply
- **Hash Type**: an 8-bit unsigned integer representing a particular hash algorithm. Possible values:
  - 0: Null Hash, this indicates that this is not a hash value and it has no data content whatsoever. If this NodeType shows up in a Qualified Hash, it will have length of 0 and (correspondingly) no data bytes whatsoever.
  - 1: SHA256
- **Content Type**: an 8-bit unsigned integer representing a particular content structure (analagous to a MIME type). Valid values are:
  - 0: binary, unknown
  - 1: UTF8 text
  - 2: JSON
- **Key Type**: an 8-bit unsigned integer representing kind of public key. Valid values are:
  - 1: OpenPGP Public Key
- **Signature Type**: an 8-bit unsigned integer representing a kind of cryptographic signature. Valid values are:
  - 1: OpenPGP binary signature
- **Hash Digest Length**: an 8-bit unsigned integer indicating how many bytes long the output digest of a particular hash function is.
- **Hash Descriptor**: A Hash Type followed by a Hash Digest Length. These two pieces of data fully specify the hash procedure needed to construct a given Hash Value.
- **Hash Value**: the actual hash value from some particular hash algorithm with some particular digest size.
- **Tree Depth**: an unsigned 32-bit integer indicating how many levels beneath the root of a particular tree a given node is.
- **Content Length**: an unsigned 16-bit integer representing how many bytes are in another field.
- **Blob**: binary data with an unspecified length. All Blob fields will have an associated Content Length and Type field to clarify their semantics.
- **Qualified Hash**: A Hash Descriptor followed by a Hash Value. This encodes both the procedure necessary to derive the hash value as well as the expected result. Special values:
  - If the **Hash Type** is 0 and the **Content Length** is zero, this is a reference to the _Null Hash_. This will usually be used in the `parent` field of nodes that are the root of a tree within the Arbor Forest.
- **Qualified Content**: A Content Type followed by a Content Length followed by the Blob that they describe
- **Qualified Key**: A Key Type followed by a Content Length followed by the Blob that they describe.
- **Qualified Signature**: A Signature Type followed by a Content Length followed by the Blob that they describe.
- **Text**: A Content Length followed by a UTF-8 string
- **SchemaVersion**: A 64 bit unsigned integer

### Common Fields

All nodes in the forest share some fields. These are described generally here, though each node type's description may contain more detailed information about the legal values in each field for that node type.

Common fields:

- `version` **SchemaVersion**: an unsigned integer representing the version of the Arbor Forest specification that this node uses. This initial draft is version 1.
- `node_type` **Node Type**: the type of the node within the forest.
- `parent` **Qualified Hash**: the hash of the parent tree node. For identity nodes, this will always be the null hash **of all zeroes**
- `id_desc` **Hash Descriptor**: the hash algorithm and digest size that should be used to compute this Node's Id
- `depth` **Tree Depth**: the number of levels this node is from the root message in its tree.
- `metadata` **Qualified Content**: arbitrary JSON data. Only valid if ContentType is JSON.
- `author` **Qualified Hash**: the id of the Identity node that signed this node
- `signature` **Qualified Signature**: the actual binary signature of the node. The structure of this field varies by the type of key in the `author` field. The Content Type of this field should be a signature type of some kind.

### Common Structure

All node types are signed and hashed with their data laid out in a specific order in memory. The procedure for constructing a node is as follows:

Determine the values of these fields:

- `version`
- `node_type`
- `parent`
- `id_desc`
- `depth`
- `metadata`
- `author`
- all node-specific fields **order specified in the description of each node type**

Write them into a buffer in the order above, with all integers written in network byte order **big endian**.

Sign the contents of the buffer using the key pointed to by `author`, and use it to create the value of `signature`.

Concatenate the value of `signature` to the end of the existing buffer, then hash the entire buffer with the algorithm and digest size specified by `id_desc` to determine the node's actual Id.

### Identity

An identity node has the following fields:

- `name` **Text**: the name of the user who controls this key. Maximum length (in bytes) is 256.
- `public_key` **Qualified Key**: the binary representation of the public key

These fields should be processed in the order given above when signing and hashing the node.
 
### Community

A Community node has the following fields:

- `name` **Text**: the name of the user who controls this key. Maximum length (in bytes) is 256.

These fields should be processed in the order given above when signing and hashing the node.
 
### Reply

A Reply node has the following fields:

- `community_id` **Qualified Hash**
- `conversation_id` **Qualified Hash**
- `content` **Qualified Content**

These fields should be processed in the order given above when signing and hashing the node.
