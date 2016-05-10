PROTOCOL.md

# Chainmail Draft Specification

## What is Chainmail?
Chainmail is an open communications platform with the following goals:

* Privacy. Communications cannot be read by unauthorized parties.
* Anonymity. You are not required to link your identity in a Chainmail community to any other identity, such as your legal name.
* Distributed. There are no central servers in Chainmail. This allows Chainmail to be fundamentally resistant to censorship and compulsory disclosure.
* Plural. There is not a single "Chainmail Network," nor is there a way to identify all the Chainmail networks that may exist.
* Manageable. Individual Chainmail networks have administrators with well-defined rights that are objectively known to all authorized members of the network. These rights enable them to provide administrative services, such as moderation and the admission and expulsion of members.
* Non-profit. Chainmail is an open source project with no interest in making a single cent off the protocol.

## Why should Chainmail exist?
We live in a world which depends on the free and secure flow of information. For example, health care providers must maintain and exchange large amounts of very sensitive information. This information must be transmitted and stored securely, but at the same time, it is unclear who bears responsibility for the storage of that data. A patient, a hospital and a doctor's office all have very valid need-to-know for a given set of electronic health records, as well as a need to update those record. But who gets to "own" those records? In a distributed model like Chainmail, we find a possible solution: no one needs to own the records. The secure transmission of the data can take place without central control.

Consider also political activists who require the ability to communicate safely without state surveillance. In a centralized network, state-sponsored attackers may be able to coerce the owner of the server into compromising activists without their knowledge, or they may be able to take a service down entirely to censor communication. A distributed model such as Chainmail is far more resistant to these kinds of attacks.

More than that, it is not difficult to imagine situations that anyone and everyone may find themselves in where they require privacy and anonymity in their communications, and would be better served if those communications were not under the control of a single entity.

## Could this technology be misused?
This project is founded in the belief that anyone and everyone may find themselves at a point in their lives where they require anonymity or privacy. Furthermore, it appears unlikely that any technology can have sufficient safeguards to prevent any misuse; in fact, it is unclear what things will qualify as "misuse," or who should be in charge of that determination.

## What is the status of Chainmail?
Chainmail is in its planning stages. We are presently looking for a small number of core contributors. At this time, our interest is primarily in developing a cryptographically-viable distributed network that learns lessons from the attacks (both legal and otherwise) that have been mounted against existing networks.

Our concerns right now are designing a set of protocols and data structures that are resistant to the attacks that have been levied against existing projects providing similar services, including Tor, BitTorrent, Bitcoin and various "deep web" services.

## How does it work?
A Chainmail community can be started by anyone. At its heart, a Chainmail community is a blockchain, similar to what is used by Bitcoin. For more information on the blockchain aspect of Chainmail, see the "Chainmail Blockchain" section.

Each block within Chainmail contains a recordset. These records define actions within the network, like adding a new member to the community, or sharing a message. For more information on the kinds of records supported within Chainmail, see the "Chainmail Records" section.

Peers within Chainmail relay new blocks to each other through a peer-to-peer swarming protocol. For more information on this, see the "Chainmail swarming" section.

A cryptographic hash (such as SHA-512/256) of the root block of a Chainmail community creates an objective, non-duplicable identifier for a community. This identifier also has the property that it discloses no information about the community itself. This ID hash can be searched in a distributed hash table (DHT) not unlike that used in BitTorrent. In so doing, community members who lack peering information to actually participate in the community can use the DHT to join the community swarm, without disclosing any information about the community itself to non-members. For more information on this, see the "Chainmail DHT" section.

## Chainmail Blockchain
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
  cm_hash author_guid;       // Author GUID, as specified in author's identity record.
  cm_hash author_pkey_hash;  // Hash of author public key used to sign this record.

  cm_hash payload_hash;      // hash of the payload of this record (all bytes of the record excluding this header)
  int64 payload_size;        // payload size, in bytes
  string description;        // optional description of payload contents
  string specific;           // metadata specific to record type
  
  cm_nonce nonce;            // selected to make record header meet proof-of-work requirements (see "Proof of Work")
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

### Credential
`cm_type = 1`

Credential records grant permissions to sign particular record types, enter blocks into the blockchain, and take certain privileged actions within records.

```
struct cm_record_credential {
  cm_hash grantee;        // hash of identity record for user receiving these permissions
  string[] permissions;   // array of strings representing individual permissions
}
```

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

| Modifier     | Meaning |
|--------------|----------
| credential@  | may change own display name in credential records
| credential@@ | may change others' display name in credential records
| thread^      | may sticky and unsticky threads
| thread]>     | may post in closed threads
| thread]      | may close own threads
| thread]]     | may close others' threads

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
| post              | user may make post records
| post-             | user may revoke own post records
| post--            | user may revoke post records of others
| \*+post            | user may grant post permission to others in credential records
| \*-post            | user may revoke post permission to others in credential records
| \*+post--          | user may grant permission to revoke post records of others
| \*+post/8192       | user may grant permission to create posts of a given max record size above the default, provided assigned record size is under 8192KiB
| \*-post/8192       | user may restrict other users to create posts to a certain max record size below the default, provided that record size is AT LEAST 8192KiB.

TODO: It would be useful to add member groups to this protocol. Making a general change to the privileges granted to a group of 300 people will require 300 credential record updates; this could be impractical.

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
  cm_hash guid;       // persistent identifier for user
  string name;        // human-readable display name
  string[] contact;   // optional external contact information for this user
  cm_hash avatar;     // optional hash referencing file record containing an avatar image
  cm_pubkey pubkey;   // public key for this user
}
```

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

### Privmsg
`cm_type = 6`

Defines a private message between a set of users. Messages are encrypted with a one-time use symmetric key, which is in turn encrypted with the public keys of each recipient. In this fashion, the Privmsg record does not disclose the actual recipients of a message. It is possible to determine the sender of a Privmsg if the sender creates the Privmsg record themselves. However, they may instead send the cm_record_privmsg to a peer, and request that peer record the Privmsg into the blockchain on their behalf.

The cm_record_privmsg provides all necessary author identity information in encrypted form, accessible only to recipients.

To determine if a private message has been received, clients must attempt their key against the rosetta tables of each Privmsg record in the blockchain.

```
struct cm_record_privmsg {
  string[] rosetta;            // rosetta table encrypting symmetric key using public keys of recipients; optionally padded with random entries to exaggerate number of recipients
  string secret;               // cm_privmsg_secret, encrypted using symmetric key
}

struct cm_privmsg_secret {
  cm_hash sender;              // GUID of sender
  int64 timestamp;             // Timestamp (in epoch seconds) of message creation
  cm_hash reply_to;            // hash of privmsg this message replies to (if any)
  cm_hash[] references;        // records in the blockstore referenced by this message
  string message;              // actual message contents
  string padding;              // random padding of arbitrary length
  cm_sig signature;            // signature of this structure, with "signature" set to 0
}
```

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

  string additional;         // JSON dictionary containing arbitrary additional protocol config data; reserved for future use

  integer max_block_size;    // maximum allowable size for a block payload, in bytes
  integer max_header_size;   // maximum allowable size for a record header, in bytes
  integer max_payload_size;  // maximum allowable size for a record payload, in bytes
  integer[] difficulty;      // number of leading bits that must be 0 in hash of records to satisfy proof-of-work. array indexed by cm_type.

  boolean dht_allowed;       // true <=> peers should list this blockchain in the public distributed hash table
  boolean public;            // true <=> peers may disclose blockstore contents to non-members
}
```

### Reaction
`cm_type = 8`

Provides functionality analogous to upvote, downvote on a record.

```
struct cm_record_reaction {
  cm_hash reference;         // record being reacted to
  string reaction;           // reaction type. "up", "down".
}
```

### Revoke
`cm_type = 9`
Marks a set of records as being revoked from the blockstore.

```
struct cm_record_revoke {
  cm_hash[] revoked;         // list of records to be revoked
}
```

### Thread
`cm_type = 10`

Defines a thread in which Comment records can be made. New threads set the "specific" field of the record header to null.

Edits to threads set the "specific" field to the record hash of the original thread record. Hash references to topics should always refer to original unedited topics, regardless of how many edits have been made. This way, a single identifier is used to refer to the comment throughout the blockstore.

```
struct cm_record_thread {
  string title;              // title for the thread
  string op;                 // original post content for thread
  string[] attributes;       // array of attributes applied to this thread. initial possibilities: "sticky", "lock"
}
```

### DHTBasis
`cm_type = 11`

By default, the DHT key for a blockchain is considered to be the hash of its root block. If a community finds it necessary to change its identity in the DHT, a DHTBasis record may be calculated. Community members will use the hash of the most recent block including a DHTBasis record as the community's ID in the DHT.

This will have the effect of invalidating all previous magnet links to the community.

This record type has no payload.

### Arbitration
`cm_type = 12`

Indicates that the signer believes this block exists in the canonical blockchain. Used for fork resolution.

This record type has no payload.

## Chainmail Swarming

**TODO**

This is a substantial part of the protocol, and describes how peers will actually communicate the blockstore to one another. The following points may be of consideration:

1. Data must be encrypted in transit to ensure privacy, and it must not be possible for non-members to masquerade as members and receive the blockstore.

2. It is undesirable to require nodes to spend bandwidth and storage acquiring all records in the blockstore; consider a very large record (like a very large database dump or other data file) that spans many gigabytes, but is unlikely to be topical for all members of the group for all time.

3. Accidental blockchain forks represent a nuisance in this protocol, and the best defense against this is low-latency distribution of a new block to all peers.

4. A topology in which one node can prevent a subset of other nodes from learning about a new block, even temporarily, is highly undesirable.

5. From BitTorrent, we've seen that downloading individual blocks of a file from a multitude of peers is often much faster than downloading an entire file from a single peer, despite the apparent overhead involved in distributed transmission.

6. Encryption may not be sufficient to prevent censorship; it would be ideal if traffic can be made to look like other routine traffic, such as HTTPS.

## Chainmail DHT
Chainmail allows magnet links, similar to BitTorrent's implementation of Kademlia. These magnet links are of the form: `magnet:?xt=urn:cmh:$HASH`, where $HASH is the hash of the root block of the Chainmail community to be joined. (See "Hashing" for details on the hashing algorithm used.)

Chainmail DHT nodes begin by generating a 256-bit ECDSA keypair. The node then generates a GUID, by computing the hash of this keypair. By generating the GUID in this fashion, Chainmail seeks to avoid the kinds of Sybil attacks that have been employed against the BitTorrent DHT, by adding a minor computational cost to creating a node as well as making it exceptionally difficult to claim GUIDs arbitrarily similar to a given target.

### Joining the DHT

Chainmail DHT clients join the DHT by selecting a bootstrap server from a well-known list. This draft specification envisions a list of default bootstrap servers being listed by hostname, one of which is selected at random. This list is stored in an easily-accessible standalone JSON text file, and can be replaced by clients as needed, for example in the event that the default bootstrap servers become compromised or unavailable, or if a client prefers to join an alternative DHT. This list is a list of the form (GUID, IP, Port).

All DHT requests are sent over UDP. Each packet (request or response) has the following format:

```
// cm_dht_hash is a buffer for a cryptohash, in the algorithm specified by the final protocol
// cm_dht_pubkey is a buffer for a public key, in the algorithm specified by the final protocol, expressed in some standardized format
// cm_dht_req_type is an enumeration specifying a request type

struct cm_dht_message {
  string m;                   // cm_dht_signable, encoded as JSON, re-encoded in base64
  cm_dht_sig s;               // signature of hash(m)
} // serialize to json

struct cm_dht_signable {
  cm_dhr_header hdr;          // header data, common to all messages
  ...                         // arbitrary keys, depending upon response type
}

struct cm_dht_header {
  cm_dht_hash qi;             // requestor's GUID
  cm_dht_hash si;             // responder's GUID
  cm_dht_pubkey p;            // sender's public key

  integer n;                  // random number selected by requestor; responses must copy this field from request
  integer t;                  // request type
}
```

Having selected a bootstrap peer, the joining client sends a "find" request:

```
// find request type = 0
struct cm_dht_find_req {
  cm_dht_hash target;         // GUID which node is attempting to locate
}

struct cm_dht_find_resp {
  cm_dht_hash tok;            // token. must be copied into 
  cm_dht_node_ref[] nodes;    // list of up to K nodes from responder's routing table, which are closest to requested target
  cm_dht_node_ref[] members;  // peer references known to this node for the requested hash
}

struct cm_dht_node_ref {
  cm_dht_hash g;              // remote node's guid.
  ip_address a;               // remote node's ip
  integer p;                  // udp port number upon which remote node is listening for dht traffic
}
```

The joining client sets "target" to its own GUID. The bootstrap peer then replies per protocol, with the K nodes from its own routing table which are closest (ad defined by the distance function defined in "Distance calculation") to the requested GUID. The joining client then repeats its request (a find for its own GUID) to each of these nodes, and adds the response nodes to its own routing table.

Peers receiving find requests will add the requestor's GUID, IP and port to their own routing tables, provided there is space to do so.

### Querying the DHT for a Chainmail group

Recall that each Chainmail group has a unique blockchain, specific to that group. The hash of this group defines a unique, non-duplicable identifier for the group which discloses no information about the contents or nature of the group. These hashes act as keys in our distributed hash table, and can be placed into magnet links.

Given a hash from a magnet link, a DHT client consults its routing table, and selects the K closest peers, using the distance function defined in the "Distance calculation" of this document. It then performs a find_node request against each of these peers.

Each peer responds with cm_dht_node_ref objects for the K nodes from its routing table whose GUIDs are closest to the requested hash. If a peer has been asked to store references to actual group members, that group peerlist is returned in the "members" array. This response includes a token, of the form: `hash(requestor IP + nonce)`, where the nonce is an arbitrary in-memory token stored by the peer and updated every 5 minutes.

The requestor maintains a list of cm_dht_node_ref objects received during it search, sorted by distance to the Chainmail group hash. This list is referred to here as the "search list." The requestor uses this sorted list to determine which nodes to query next, preferring the nodes which have minimal distance.

Queried peers whose response includes no GUIDs less distant than their own are considered to be potential results, and are added to the "results" list. Each peer in the results list is sent a message requesting to store the requestor's IP and port as a swarm peer.

### Joining a DHT group

When a peer identifies "result" DHT nodes as described in the "Querying the DHT for a Chainmail group" section, it sends a "announce" request:

```
struct cm_dht_announce_req { // request type = 1
  cm_dht_hash target;         // target discussion group hash which announcing peer will accept swarm traffic for
  cm_dht_hash tok;            // copied from peer's cm_dht_find_node_resp
  integer p;                  // udp port number upon which joining peer is listening for discussion group swarm traffic
}

struct cm_dht_announce_resp {
  // no payload data; simply respond with standard header
}
```

The receiving peer then maintains a hash table, keyed on the hash in the `target` field, and valued in lists of [IP, Port number, Time seen] pairs. Peers may then perform memory management of their tables as follows:

* Lists may be capped to an arbitrary length, and pruned by last-seen time
* The hash table may be capped to an arbitrary number of key-value pairs. When pruning individual key-value pairs, lists whose most recent last seen time is oldest can be deleted first.

### Refreshing the DHT

Peers participating in blockchains whose Protocol record indicates that the community should be advertised in the DHT should attempt to locate swarm peers using the methodology described above every 60 minutes, regardless of whether they already have active participation in the swarm. In this way, active peers do not simply "age out" of the DHT, and the DHT remains current even as individual nodes join and drop out.

#### Tokens
The `tok` field is a countermeasure against malicious clients forging UDP packets from victims in order to solicit a DDoS attack. Nodes calculate their `tok` field by maintaining a queue of 3 32-bit random nonces in memory. Every 5 minutes, a new nonce is inserted into the queue, and if the queue length exceeds 3, the oldest nonce is removed.

When generating the `tok` field of `cm_dht_find_resp`, nodes calculate `hash(N + Requestor IP Address)`, where N is the client's most recent nonce. When evaluating the `tok` field of the `cm_dht_announce_req`, the receiving peer compares the supplied token against `hash(M + Requestor IP Address)`, where M is each individual nonce in its table. In this way, tokens are specific to requestor IP addresses, but require in-memory storage of only the 3 most recent 32-bit nonce values.

### Routing tables

Each DHT peer maintains a routing table, of the following form:

```
struct dht_routing_table_entry {
  cm_dht_hash guid;       // remote peer's public key
  ip_address address;     // remote peer's IP
  integer port;           // UDP port number upon which remote peer accepts DHT traffic
  int status;             // status code. 0 = known-good, 1 = known-accessible, 2 = questionable, 3 = known-bad
  timestamp lastseen;     // time we last saw this node
}
```

The routing table is implemented in terms of a list of buckets. Each bucket has a minimum and maximum GUID that it will accept, and contains a maximum of K entries. At initialization, the routing table has a single bucket, whose range is 0 to 2^256 (based on the 256-bit hash recommended in the "Hashing" section).

#### Inserting new entries into the routing table
When an entry is to be inserted into the routing table, the node first identifies the bucket whose allowable range (as defined by its minimum and maximum acceptable GUIDs) includes the GUID of the entry to be inserted. If this bucket has fewer than K entries, the record is inserted into the bucket, and the insertion operation is complete.

If the bucket has K entries whose status code indicates that they are known-good, then the incoming entry is discarded and is not inserted into the table, unless the bucket's range includes the node's own GUID. If this is the case, the bucket is split into two buckets, each with half the range of the parent bucket. The parent bucket is removed from the table and the child buckets are added, and each of the parent bucket's entries are inserted into the appropriate child bucket. The insertion algorithm is then repeated.

If the bucket has entries whose status code indicates that they are known-bad, then one such entry is selected arbitrarily to be replaced by the new entry.

If the bucket has entries whose status code indicates that they are questionable, the questionable entries are listed from least recently seen to most recently seen. Entries are popped from this list and pinged. If an entry fails to respond to a ping, the ping test is repeated. If the entry fails to respond again, the entry is removed from the list and replaced with the incoming entry. If the selected entry responds to either of the ping attempts, it is marked known-good and its last seen time is updated, and the insertion algorithm is repeated.

#### Status
An entry is known-good if it has responded to a query in the past 15 minutes. An entry is also known-good if it has ever responded to a query, and has issued a query to us in the past 15 minutes.

An entry is known-accessible if it has ever responded to a query, but has not issued a query or responded to a query in the past 15 minutes. For the purposes of the insertion algorithm, this is equivalent to "questionable."

An entry is questionable if it has never responded to a query of ours.

An entry is known-bad if it has failed to respond to 2 consecutive requests.

#### Bucket freshening

Each bucket has a last changed timestamp indicating when it was last freshened. This timestamp is updated in any of the following events:

 * An entry is pinged and responds
 * An entry is inserted into the bucket
 * An entry in the bucket is replaced by another entry

When a bucket has not been freshened in 15 minutes, it is refreshed. This is done by doing a "find" request for a random GUID selected from within the acceptable range of the bucket. This causes the node to discover a burst of peers within the range of the bucket, not only prompting the node to ping stale peers, but providing alternatives to replace them with.

### Pinging peers

The protocol provides a simple ping request to test if peers are available:

```
struct cm_dht_ping_req { // type = 2
  // no specific ping request payload; header is sufficient
}

struct cm_dht_ping_resp {
  // no specific ping response payload; header is sufficient
}
```

### Choosing K

BitTorrent choses K=8, meaning that buckets contain 8 entries and replies to its "FIND_NODE" requests (the analogue to `find` in Chainmail DHT) include up to 8 peers. The choice of K is a tradeoff between network integrity, against memory utilization in peer lists and bandwidth utilization in `find` responses. In other words, a higher K means that the network is more tightly connected, finds hash values in fewer steps and new nodes discover peers more quickly; at the cost of requiring more network requests of larger size, and additional memory to store peer tables.

There appears to be no motivation in choosing K=2^n, other than aesthetic value; i.e. there appears to be no technical restriction on K=9 or other integers with prime factors other than 2, although it worth noting that K=2^n may present various small efficiencies at different points in the protocol.

This draft specification chooses K=8 to match BitTorrent, on the basis that BitTorrent's choice appears to have worked. It is worth noting that BitTorrent's DHT has a hash length of 160 bits, whereas this draft specification provides a hash length of 256 bits. No consideration has been made as to whether there is a consequent benefit of increasing K to compensate for the increased sparseness of the hash table, or other consequences of the increased hash length.

### Distance calculation

The DHT requires a distance function of the form `dist GUID, GUID -> Integer`. This draft specification uses the same distance function used in Kademlia: the XOR of each GUID, interpreted as an unsigned integer.

### Hashing

This draft specification uses an implementation of SHA512/256 (https://eprint.iacr.org/2010/548.pdf) for the DHT, which is a 256-bit application of SHA-512, motivated by the interesting property of modern 64-bit CPUs to compute SHA-512 hashes more efficiently than they do SHA-256 hashes.

This also foils a seemingly esoteric partial-message collision attack against MD4, MD5, SHA-1 and SHA-2 described in "Cryptography Engineering" by Ferguson, Schneier and Kohno.

### Choice of security level

This protocol modifies the Kademlia implementation used by BitTorrent by basing node GUID on public key, which nodes are required to prove ownership of in each transaction. Consequently, there is substantial CPU overhead in this protocol compared to BT.

The purpose of this is to prevent nodes from selecting arbitrary GUIDs, by which they can accomplish censorship of discussion groups by positioning themselves as the nodes whose GUIDs have minimal distance to the discussion group in question, and then intentionally return incorrect peerlists or refuse to process announce requests.

As of this writing, no effort has been made in this drafting process to evaluate the implications of the CPU cost of adding cryptography. A specific concern is whether, under typical traffic loads that will be experienced by peers, the added CPU cost will be heavy enough to discourage users from participating in the DHT.

The CPU cost is directly related to the security level offered by our choice of algorithms; this document currently specifies 256-bit ECDSA, and SHA512/256. Each of these algorithms are presently believed to offer a security level of 128 bits. This may be excessive, considering that the sole object of this security is to make it infeasible for attackers to choose arbitrary GUIDs.

A lower security level may suffice to make malicious GUID selection impractical, while reducing the resource cost of application.

# Acknowledgments

This draft specification borrows generously from concepts in the Bitcoin and BitTorrent projects, as well as the Kademlia distributed hash table. In particular, the following documents have proven particularly illuminating:

* http://www.bittorrent.org/beps/bep_0005.html
* https://en.wikipedia.org/wiki/Kademlia
* http://www.academia.edu/3424033/BitTorrents_Mainline_DHT_Security_Assessment

Many thanks to the contributors to Tor, Bitcoin, BitTorrent, as well as the many researchers whose work has been very informative in drafting this specification.
