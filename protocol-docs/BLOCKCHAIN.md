# Chainmail Blockchain

## Overview
A blockchain is a data structure, popularized in Bitcoin, in which discrete "blocks" are chained together in a sequence. Each block contains a cryptographic hash of the previous block. As a consequence of this, once a block is added to the chain, it cannot be modified without invalidating the hashes of all successive blocks. A blockchain therefore represents a distributed immutable history.

```
// cm_hash is a buffer containing a cryptographic hash, eg. sha256
// cm_type is an integer enumeration designating a record type
// cm_sig is a crypto signature of a message
// cm_nonce is an arbitrary buffer, used for proof-of-work. This will be discussed later.

struct cm_record_block {
  cm_record_header header;   // standard record header
  cm_hash prev_hash;         // hash of previous cm_block in blockchain (i.e. block whose index == (this.index - 1))

  int index;                 // height of blockchain; 0 for root block, 1 for first block after root, etc.
  cm_record_ref[] added;     // array of references to records added to the blockstore by this block (see "Added Records")
  cm_hash[] revoked;         // array of hashes of records removed from the blockstore by this block (see "Revoked Records")
}

struct cm_record_ref {
  cm_hash record_hash;       // hash of the record being referenced
  int64 size;                // size of record, in bytes, including record header
  cm_type type;              // record type enumeration
}

struct cm_record_header {
  int64 timestamp;           // timestamp of block signature, in seconds since 1970-01-01 00:00:00 -0000
  cm_type type;              // record type enumeration
  cm_hash venue;             // Hash of the root block of the blockchain this record is originally intended for. This provides a way to determine if a record is being used in a clone.
  cm_hash author_guid;       // Author GUID, as specified in author's identity record.
  cm_hash author_pkey_hash;  // Hash of author public key used to sign this record.

  cm_hash payload_hash;      // hash of the payload of this record (all bytes of the record excluding this header)
  int64 payload_size;        // payload size, in bytes
  string description;        // optional description of payload contents
  string specific;           // metadata specific to record type
  
  cm_nonce nonce;            // selected to make record header meet proof-of-work requirements (see "Proof of Work")
  string password;           // password required to handshake with peers; optional
  cm_sig signature;          // signature of this record header, with signature field zeroed out
}
```

### Added Records
Records are not embedded directly within blocks. Instead, they are referenced by hash. Think of a Chainmail blockchain as a sort of distributed filesystem. Each record is a file, and the name of that file is the hash of the record. The right of that file to exist within the filesystem is provided by that file being listed in a valid block within the chain.

This distributed filesystem is referred to as the "blockstore."

The record header is designed to make it possible to validate the authenticity of a record header without actually requiring a client to download the record payload. Therefore, the record header is not hashed directly with the payload, instead contains a hash of the payload.

### Revoked Records
Chainmail has the concept of record revocation. This is a formal statement that the data contained in a record is no longer pertinent. For example, Block 17 might add a Credential record granting rights to add new blocks to the chain to a certain identity ("Alice"). Alice would then be permitted to add new blocks to the chain using her key.

In Block 21, Alice might have her credential revoked. Peers will no longer accept blocks from Alice.

Revoked records are listed in the block for expedience. However, mere presence of a record hash in the revoked list is insufficient to qualify a record as formally revoked. The block must also add a new record that causes the existing record to become revoked. The most obvious example of this is a Revoke record, which explicitly revokes arbitrary records.

However, records may become revoked by other means as well. A Credential record issued to an identity that already has an outstanding Credential revokes that prior Credential, for example.

### Proof of Work
As a countermeasure against spam, proof-of-work requirements can be levied for new records and new blocks. This proof-of-work comes in the form of choosing the "nonce" field of a cm_record_header so that the completed header hashes to a value with a minimum number of leading 0 bits. The number of 0 bits required is determined on a per-record-type basis, and is specified in the Protocol record for a blockchain.

Because the nonce is chosen BEFORE the signature, the process of performing this work is significantly complicated compared to Bitcoin. Specifically, the worker must:
 - Possess the private key of the record signer
 - Implement the signing algorithm in addition to the hashing algorithm

While this does not rule out the use of botnets, GPUs, FPGAs or custom ASIC hardware to perform work, it significantly complicates the matter.

### Blockchain Forks
A fundamental challenge in blockchains is the eventuality of forks. Imagine that two blockchain peers create two blocks, A and B at approximately the same time. They both reference the same block X as their predecessor. A block can be a predecessor only once, so one or both of these blocks will need to be discarded.

Bitcoin addresses this issue through proof-of-work: blocks have very, very high proof-of-work requirements. The Bitcoin blockchain then follows the tallest chain. Thus, anyone who controls 50% of the computing power on the Bitcoin network has the power to dictate history, and can double-spend coins by constructing blockchain forks that are taller than the genuine chain. Because Bitcoin is a financial network with global interest, control of the network has real value, and obtaining 50% of the computing power is very difficult.

This is not going to be a successful model in a small forum. To preserve timely communication, it must be possible to add blocks quickly -- but for a small network to add blocks quickly, the cost for the proof-of-work must be relatively low. Furthermore, the amount of computational power available to do work will be variable, and often times zero, meaning that there will be windows of opportunity for attackers to construct malicious forks.

Chainmail addresses this issue through moderation. Given a set of potential successors (B1, B2, ...) listing block A as a predecessor, clients apply the following preference schedule:

 1. Forks including Arbitration blocks signed by the root identity
 2. Forks including Arbitration blocks signed by identities with the "arbitration" privilege
 3. Forks with highest number of unique signing keys within maximum considerable fork length specified in Protocol block
 4. Fork whose first block has lowest hash

In other words, given a fork, there exist manual intervention opportunities for predefined keys to pick the winning fork, with the original forum identity having ultimate authority. If no manual intervention has been made (or if multiple forks bear manual intervention of equal authority), automatic resolution is used.

Fork length is first considered, on the theory that the best course of action in good-faith forks is to follow the fork with the most activity. When all forks meet or exceed a maximum length, length is no longer considered, on the basis that it is feasible for bad-faith peers to construct Chainmail forks of arbitrary length.

This grants the ability to rewrite history to administrators. While unacceptable in Bitcoin, the fact that this is an information-based network changes the consequences somewhat. People can perceive bad-faith action by administrators readily, and simply defect to a new Chainmail group.

It becomes possible to clone an entire forum with all its original data, but completely different leadership: create a new root block designating a new administrator, pull in the identity records of all the members of the original forum, create new credentials for them in the new chain (excepting maybe those belonging to members who were believed to be the motivation for leaving the original forum in the first place), and create blocks for each of the existing comment, privmsg or other records desired. All that is then necessary is to get sufficient members to follow and contribute to the clone in place of the original. The only apparent barrier to this is the `creator` and `authority` hash references in the block header. This may be an interesting enough feature to warrant a refactor of the headers to support, perhaps by allowing those fields to exist in an unsigned portion of the record, or a portion signed by an authority that may or may not be the record author.

## Chainmail Records

### Block
`cm_type = 0`
See above.

#### Rules

* The `prev_hash` field must be non-null and reference a valid block record. (Violations of this rule are treated specially, and are covered in the "Blockchain Forks" section). Alternatively, the `prev_hash` field may be null, in which case the `index` field must be 0. This alternative case is the root block, and only one root block may exist in a chain.
* The block record referenced must not have previously been referenced as a `prev_hash` for a previous block.
* The `index` field must be equal to 1 plus the `index` field of the block referenced in `prev_hash`.
* None of the hashes in `added` or `revoked` may reference a block record.
* Any records referenced in `revoked` must have a new record which causes their revocation, and that new record must be referenced in `added`.
* All records referenced in `added` must be valid according to the rules of this specification; if any record listed in `added` is invalid, the entire block is rejected.

### Comment
Comment records are where individual forum posts are made. They represent a single post in a single thread. New comments must set the "specific" field of the record header to null. Edits to comments may be made by making new comment records, with the "specific" field set to the hash of the original comment record.

Hash references to comments should always refer to original unedited comments, regardless of how many edits have been made. This way, a single identifier is used to refer to the comment throughout the blockstore.

```
struct cm_record_comment {
  cm_hash thread;         // hash of thread record for thread that this comment belongs to
  cm_hash[] reply_to;     // hashes of comment records for existing comments that this new comment is in reply to
  cm_hash[] references;   // hashes of other entries in the blockstore relevant to this comment; eg. file attachments
  string message;         // actual content of comment
}
```

#### Rules

* The `thread` field must reference a non-revoked thread record in the blockstore.
* The `thread` field must either reference a non-closed thread, or the user signing the comment record must have the `thread]` permission (if they are also the signer for the thread), or the `thread]]` permission (if they are not the signer for the thread).
* Any records referenced in the `reply_to` or `references` arrays must be non-revoked.
* The message field must be non-empty.

### Credential
`cm_type = 1`

Credential records grant permissions to sign particular record types, enter blocks into the blockchain, and take certain privileged actions within records.

```
struct cm_record_credential {
  cm_hash grantee;             // guid of identity record for user receiving these permissions
  string[] permission_groups;  // array of references to PermissionGroup GUIDs to apply to this user
  string[] permissions;        // array of strings representing individual permissions
}
```

#### Rules

* The `grantee` field must reference a valid identity record GUID in the blockstore, and the latest identity record for that GUID must be non-revoked.
* The record signer must have a credential record granting permissions sufficient to account for all change in permissions, as calculated by comparing the grantee's current permissions to those granted by all referenced permission groups in the `permission_groups` field and individual permissions in the `permissions` field.
* The `permission_groups` field must reference valid GUIDs for PermissionGroup records in the blockstore, and the latest PermissionGroup records for each GUID must be non-revoked.

#### Applying permission groups
A credential record provides both individual and group permissions. A user's total permissions are taken to be the union of all permissions in any of their groups, as well as their individual permissions array. In the event that multiple forms of the same permission are granted (e.g. `comment/4096` and `comment/8192`), the privilege granting greater access will be preferred.

#### Permissions
Permissions exist in the form of strings, which are added together in the permissions array of an identity's credential record. Privileges are lowercase names of records. A privilege grants a right to create a record. Additionally, the following permission modifiers apply:

##### Suffixes
| Suffix | Meaning
|--------|---------
| -      | may revoke own records
| --     | may revoke records of others
| /n     | record may have total size up to n KiB (omit for default)
| !      | may amend own record (for supported records)
| !!     | may amend records of others (for supported records)

 
##### Prefixes
| Prefix | Meaning
|--------|---------
| \*+     | may grant permission to others in credential records
| \*-     | may revoke permission from others in credential records
| \*\*+    | equivalent to \*+, with additional permission to grant \*+ and \*\*+ to others.
| \*\*-    | equivalent to \*-, with additional permission to grant \*- and \*\*- to others.

##### Record-specific modifiers
The following modifiers are specific to particular record types.

| Modifier      | Meaning |
|---------------|----------
| credential@   | may change own display name in credential records
| credential@@  | may change others' display name in credential records
| thread^       | may sticky and unsticky threads
| thread]>      | may post in closed threads
| thread]       | may close own threads
| thread]]      | may close others' threads
| reaction:up   | may create reaction records with the "up" reaction
| reaction:down | may create reaction records with the "down" reaction

##### Special permissions
The following permissions are not connected to any record type.

| Permission | Meaning
|------------|----------
| super      | User has all permissions, and may create records of any size.
| peer       | May receive plaintext of blockstore contents.

##### Examples
The following are examples of applications of the prefix/suffix system.

| Permission string | Meaning
|-------------------|---------
| comment              | user may make comment records
| comment-             | user may revoke own comment records
| comment--            | user may revoke comment records of others
| \*+comment            | user may grant comment permission to others in credential records
| \*-comment            | user may revoke comment permission to others in credential records
| \*+comment--          | user may grant permission to revoke comment records of others
| \*+comment/8192       | user may grant permission to create comments of a given max record size above the default, provided assigned record size is under 8192KiB
| \*-comment/8192       | user may restrict other users to create comments to a certain max record size below the default, provided that record size is AT LEAST 8192KiB.

### File
`cm_type = 2`

A File record does not have a structured payload; instead, it is a file of arbitrary type. The "specific" section of the record header contains a string of the following format: "MIMETYPE;FILENAME", where MIMETYPE is the MIME content type of the file, and FILENAME is a recommended name for the file to be used on the client's local filesystem if the file is downloaded.

### Identity
`cm_type = 3`

An Identity record introduces a public key to the network. A new identity sets its guid field to the hash of the pubkey field. This guid remains persistent, even if the user elects to replace his or her public key in the future.

The contents of the user's Identity record can be updated by creating a new Identity record with the same guid field. This automatically revokes the previous Identity record for that GUID.

```
// cm_pubkey is a datatype containing a public key

struct cm_record_identity {
  cm_hash guid;        // persistent identifier for user
  string name;         // human-readable display name
  string[] contact;    // optional external contact information for this user
  cm_hash avatar;      // optional hash referencing file record containing an avatar image
  cm_pubkey pubkey;    // public key for this user; used to sign records.
  cm_pubkey pubkey_ro; // secondary public key used for blockchain peering; may not be used to sign records.
}
```

#### Rules

* The `guid` field must either be the hash of the `pubkey` field (using the group's preferred hash algorithm), or the hash of the `pubkey` field of the first identity record of matching `guid` appearing in the blockchain.
* The `avatar` field must reference a non-revoked File record in the blockstore. The `specific` field of this file's record header must show an acceptable MIME type.
* The `pubkey` and `pubkey_ro` fields must not appear as the `pubkey` or `pubkey_ro` field of any other non-revoked identity record.
* If the `pubkey` or `pubkey_ro` field differs from the `pubkey` or `pubkey_ro` field of the previous identity record for the same GUID, then the new identity record must be signed by the key listed in the previous record's `pubkey` field.

#### Acceptable avatar MIME types

* image/gif
* image/jpeg
* image/png
* image/svg+xml

#### External contact information
The user can optionally supply external contact information, if desired. e.g. `email:joe@example.com`, `skype:janesmith761231`.

### Metadata
`cm_type = 4`

Provides basic human-readable about the forum, like title and rules. The insertion of a new Metadata record automatically revokes any previous Metadata record in the chain.

```
struct cm_record_metadata {
  string title;          // title of forum
  string description;    // text description of forum purpose
  string rules;          // HTML containing forum rules
  cm_hash[] resources;   // References to records in the blockstore. Placeholder for future standard.
}
```

### Objection
`cm_type = 5`

This provides a distributed alternative to the post reporting function found on many forums. In this case, the Objection is made public as opposed to being sent privately to moderation staff.

```
struct cm_record_objection {
  cm_hash reference;  // record hash of the record found to be objectionably
  string comment;     // optional comment explaining nature of objection
}
```

#### Rules

* The `reference` field must reference a non-revoked record in the blockstore

### Privmsg
`cm_type = 6`

Defines a private message between a set of users. Messages are encrypted with a one-time use symmetric key, which is in turn encrypted with the public keys of each recipient. In this fashion, the Privmsg record does not disclose the actual recipients of a message. It is possible to determine the sender of a Privmsg if the sender creates the Privmsg record themselves. However, they may instead send the cm_record_privmsg to a peer, and request that peer record the Privmsg into the blockchain on their behalf.

The cm_record_privmsg provides all necessary author identity information in encrypted form, accessible only to recipients.

To determine if a private message has been received, clients must attempt their key against the rosetta tables of each Privmsg record in the blockchain.

```
struct cm_record_privmsg {
  string[] rosetta;            // one element per recipient, with each element being symmetric key encrypted using public key of recipient; optionally contains additional elements of random data to exaggerate number of recipients.
  string secret;               // cm_privmsg_secret, encrypted using symmetric key
}

struct cm_privmsg_secret {
  cm_hash sender;              // GUID of sender
  cm_hash[] recipients;        // GUIDs of each recipient.
  int64 timestamp;             // Timestamp (in epoch seconds) of message creation
  cm_hash reply_to;            // hash of privmsg this message replies to (if any)
  cm_hash[] references;        // records in the blockstore referenced by this message
  string message;              // actual message contents
  string padding;              // random padding of arbitrary length
  cm_sig signature;            // signature of this structure, with "signature" set to 0
}
```

#### Rules

* The `rosetta` field must be a non-empty array.
* The `secret` field must be a non-empty string.

#### Rosetta table format

Entries in the rosetta table are of two kinds:

* Base64 strings, whose content is a rosetta entry encrypted using the public key of a single recipient and encoded in Base64.
* Random base64 strings, of similar length to records bearing encrypted keys.

Rosetta entries are strings with the following format: `length(x) + x + key + y`, where `x` and `y` are randomly chosen padding, `length` is a function returning a string representing a decimal length of the input string, and `key` is the Base64 encode of the symmetric key. The symmetric key must be of the algorithm stated in the `sym_algorithm` field of the group's most recent Protocol record.

### Protocol
`cm_type = 7`
Protocol records specify basic information about the protocol settings to be used for the blockchain.

```
struct cm_record_protocol {
  string blockstore_fmt;     // string identifier for blockstore format. use "cmbsf_0"
  string blockstore_ruleset; // string identifier for blockstore ruleset. use "cmbsrs_0"
  string swarm_proto;        // string identifier for swarming protocol. use "cmswarm_0"

  string hash_algorithm;     // string identifier designating hashing algorithm, e.g. SHA256
  string sym_algorithm;      // string identifier designating symmetric encryption algorithm, e.g. AES256
  string asym_algorithm;     // string identifier designating asymmetric encryption algorithm, e.g. RSA4096

  string additional;         // JSON dictionary containing additional protocol config data; reserved for future use
  string custom;             // JSON dictionary containing custom config data; contents will never be specified in this document

  integer max_block_size;    // maximum allowable size for a block payload, in bytes
  integer max_header_size;   // maximum allowable size for a record header, in bytes
  integer max_payload_size;  // maximum allowable size for a record payload, in bytes
  integer[] difficulty;      // number of leading bits that must be 0 in hash of records to satisfy proof-of-work. array indexed by cm_type. If the array contains a null, or is too short to contain a record type, default to 0.

  boolean dht_allowed;       // true <=> peers should list this blockchain in the public distributed hash table
  boolean public;            // true <=> peers may disclose blockstore contents to non-members
}
```

#### Rules

* `blockstore_fmt` must be a valid Chainmail blockstore format. (Currently, only `cmbsf_0` is defined, and refers to the format defined in this document.)
* `blockstore_ruleset` must be a valid Chainmail blockstore ruleset. (Currently, only `cmbsrs_0` is defined, and refers to the ruleset defined in this document.)
* `swarm_proto` must be a valid Chainmail blockstore swarming protocol. (Currently, only `cmswarm_0` is defined.)
* `hash_algorithm` must be a valid hashing algorithm.
* `sym_algorithm` must be a valid symmetric encryption algorithm.
* `asym_algorithm` must be a valid asymmetric encryption algorithm.
* `additional` must be a JSON dictionary.
* `custom` must be a JSON dictionary.
* `max_block_size` must be an integer greater than XXXX. TODO: Fill in XXXX with absolute minimum block size.
* `max_header_size` must be an integer greater than XXXX. TODO: Fill in XXXX with absolute minimum header size.
* `max_payload_size` must be an integer greater than XXXX. TODO: Fill in XXXX with absolute minimum payload size.
* `difficulty` must be an array of integers >= 0.
* `dht_allowed` must be a boolean.
* `public` must be a boolean.
* `blockstore_fmt`, `blockstore_ruleset`, `swarm_proto`, `hash_algorithm`, `sym_algorithm` and `asym_algorithm` must be equal to the values held in the earliest protocol record in the blockchain.

### Reaction
`cm_type = 8`

Provides functionality analogous to upvote, downvote on a record.

```
struct cm_record_reaction {
  cm_hash reference;         // record being reacted to
  string reaction;           // reaction type. "up", "down".
}
```

#### Rules

* `reference` must be a valid reference to a non-revoked record in the blockstore
* `reaction` must be one of the following: `up`, `down`.
* `reaction` may be `up` only if the record signer has the `reaction:up` permission.
* `reaction` may be `down` only if the record signer has the `reaction:down` permission.

### Revoke
`cm_type = 9`
Marks a set of records as being revoked from the blockstore.

```
struct cm_record_revoke {
  cm_hash[] revoked;         // list of records to be revoked
}
```

#### Rules

* Each element in `revoked` must be a reference to a non-revoked record in the blockstore
* The record signer must have a current credential record granting privileges to delete each record in the `revoked` list

### Thread
`cm_type = 10`

Defines a thread in which Comment records can be made. New threads set the "specific" field of the record header to null.

Edits to threads set the "specific" field to the record hash of the original thread record. Hash references to threads should always refer to original unedited threads, regardless of how many edits have been made. This way, a single identifier is used to refer to the thread throughout the blockstore.

```
struct cm_record_thread {
  string title;              // title for the thread
  string op;                 // original post content for thread
  string[] attributes;       // array of attributes applied to this thread. initial possibilities: "sticky", "lock"
}
```

#### Rules

* The `thread` field must be a non-empty string
* The `op` field must be a non-empty string
* The `attributes` field may contain the `sticky` string only if a) the record signer has the `thread^` permission and is also the record signer of the original version of the thread record, b) the record signer has the `thread^^` permission and is not the record signer of the original version of the thread record, or c) the previous version of the thread record had the `sticky` attribute.
* The `attributes` field may contain the `lock` string only if a) the record signer has the `thread]` permission and is also the record signer of the original version of the thread record, b) the record signer has the `thread]]` permission and is not the record signer of the original version of the thread record, or c) the previous version of the thread record had the `lock` attribute.

### DHTBasis
`cm_type = 11`

By default, the DHT key for a blockchain is considered to be the hash of its root block. If a community finds it necessary to change its identity in the DHT, a DHTBasis record may be calculated. Community members will use the hash of the most recent block including a DHTBasis record as the community's ID in the DHT.

This will have the effect of invalidating all previous magnet links to the community.

This record type has no payload.

### Arbitration
`cm_type = 12`

Indicates that the signer believes this block exists in the canonical blockchain. Used for fork resolution.

This record type has no payload.

### PermissionGroup
`cm_type = 13`

Edits to PermissionGroups set the "specific" field to the record hash of the original PermissionGroup record. Hash references to PermissionGroups should always refer to original unedited PermissionGroups, regardless of how many edits have been made. This way, a single identifier is used to refer to the PermissionGroup throughout the blockstore.

```
struct cm_record_permissiongroup {
  string title;            // title for this permission group
  string[] permissions;    // permissions granted by this group
}
```
