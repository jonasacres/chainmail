# Chainmail DHT
Chainmail allows magnet links, similar to BitTorrent's implementation of Kademlia. These magnet links are of the form: `magnet:?xt=urn:cmh:$HASH`, where $HASH is the hash of the root block of the Chainmail community to be joined. (See "Hashing" for details on the hashing algorithm used.)

Chainmail DHT nodes begin by generating a 256-bit ECDSA keypair. The node then generates a GUID, by computing the hash of this keypair. By generating the GUID in this fashion, Chainmail seeks to avoid the kinds of Sybil attacks that have been employed against the BitTorrent DHT, by adding a minor computational cost to creating a node as well as making it exceptionally difficult to claim GUIDs arbitrarily similar to a given target.

## Joining the DHT

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

## Querying the DHT for a Chainmail group

Recall that each Chainmail group has a unique blockchain, specific to that group. The hash of this group defines a unique, non-duplicable identifier for the group which discloses no information about the contents or nature of the group. These hashes act as keys in our distributed hash table, and can be placed into magnet links.

Given a hash from a magnet link, a DHT client consults its routing table, and selects the K closest peers, using the distance function defined in the "Distance calculation" of this document. It then performs a find_node request against each of these peers.

Each peer responds with cm_dht_node_ref objects for the K nodes from its routing table whose GUIDs are closest to the requested hash. If a peer has been asked to store references to actual group members, that group peerlist is returned in the "members" array. This response includes a token, of the form: `hash(requestor IP + nonce)`, where the nonce is an arbitrary in-memory token stored by the peer and updated every 5 minutes.

The requestor maintains a list of cm_dht_node_ref objects received during it search, sorted by distance to the Chainmail group hash. This list is referred to here as the "search list." The requestor uses this sorted list to determine which nodes to query next, preferring the nodes which have minimal distance.

Queried peers whose response includes no GUIDs less distant than their own are considered to be potential results, and are added to the "results" list. Each peer in the results list is sent a message requesting to store the requestor's IP and port as a swarm peer.

## Joining a DHT group

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

## Refreshing the DHT

Peers participating in blockchains whose Protocol record indicates that the community should be advertised in the DHT should attempt to locate swarm peers using the methodology described above every 60 minutes, regardless of whether they already have active participation in the swarm. In this way, active peers do not simply "age out" of the DHT, and the DHT remains current even as individual nodes join and drop out.

### Tokens
The `tok` field is a countermeasure against malicious clients forging UDP packets from victims in order to solicit a DDoS attack. Nodes calculate their `tok` field by maintaining a queue of 3 32-bit random nonces in memory. Every 5 minutes, a new nonce is inserted into the queue, and if the queue length exceeds 3, the oldest nonce is removed.

When generating the `tok` field of `cm_dht_find_resp`, nodes calculate `hash(N + Requestor IP Address)`, where N is the client's most recent nonce. When evaluating the `tok` field of the `cm_dht_announce_req`, the receiving peer compares the supplied token against `hash(M + Requestor IP Address)`, where M is each individual nonce in its table. In this way, tokens are specific to requestor IP addresses, but require in-memory storage of only the 3 most recent 32-bit nonce values.

## Routing tables

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

### Inserting new entries into the routing table
When an entry is to be inserted into the routing table, the node first identifies the bucket whose allowable range (as defined by its minimum and maximum acceptable GUIDs) includes the GUID of the entry to be inserted. If this bucket has fewer than K entries, the record is inserted into the bucket, and the insertion operation is complete.

If the bucket has K entries whose status code indicates that they are known-good, then the incoming entry is discarded and is not inserted into the table, unless the bucket's range includes the node's own GUID. If this is the case, the bucket is split into two buckets, each with half the range of the parent bucket. The parent bucket is removed from the table and the child buckets are added, and each of the parent bucket's entries are inserted into the appropriate child bucket. The insertion algorithm is then repeated.

If the bucket has entries whose status code indicates that they are known-bad, then one such entry is selected arbitrarily to be replaced by the new entry.

If the bucket has entries whose status code indicates that they are questionable, the questionable entries are listed from least recently seen to most recently seen. Entries are popped from this list and pinged. If an entry fails to respond to a ping, the ping test is repeated. If the entry fails to respond again, the entry is removed from the list and replaced with the incoming entry. If the selected entry responds to either of the ping attempts, it is marked known-good and its last seen time is updated, and the insertion algorithm is repeated.

### Status
An entry is known-good if it has responded to a query in the past 15 minutes. An entry is also known-good if it has ever responded to a query, and has issued a query to us in the past 15 minutes.

An entry is known-accessible if it has ever responded to a query, but has not issued a query or responded to a query in the past 15 minutes. For the purposes of the insertion algorithm, this is equivalent to "questionable."

An entry is questionable if it has never responded to a query of ours.

An entry is known-bad if it has failed to respond to 2 consecutive requests.

### Bucket freshening

Each bucket has a last changed timestamp indicating when it was last freshened. This timestamp is updated in any of the following events:

 * An entry is pinged and responds
 * An entry is inserted into the bucket
 * An entry in the bucket is replaced by another entry

When a bucket has not been freshened in 15 minutes, it is refreshed. This is done by doing a "find" request for a random GUID selected from within the acceptable range of the bucket. This causes the node to discover a burst of peers within the range of the bucket, not only prompting the node to ping stale peers, but providing alternatives to replace them with.

## Pinging peers

The protocol provides a simple ping request to test if peers are available:

```
struct cm_dht_ping_req { // type = 2
  // no specific ping request payload; header is sufficient
}

struct cm_dht_ping_resp {
  // no specific ping response payload; header is sufficient
}
```

## Choosing K

BitTorrent choses K=8, meaning that buckets contain 8 entries and replies to its "FIND_NODE" requests (the analogue to `find` in Chainmail DHT) include up to 8 peers. The choice of K is a tradeoff between network integrity, against memory utilization in peer lists and bandwidth utilization in `find` responses. In other words, a higher K means that the network is more tightly connected, finds hash values in fewer steps and new nodes discover peers more quickly; at the cost of requiring more network requests of larger size, and additional memory to store peer tables.

There appears to be no motivation in choosing K=2^n, other than aesthetic value; i.e. there appears to be no technical restriction on K=9 or other integers with prime factors other than 2, although it worth noting that K=2^n may present various small efficiencies at different points in the protocol.

This draft specification chooses K=8 to match BitTorrent, on the basis that BitTorrent's choice appears to have worked. It is worth noting that BitTorrent's DHT has a hash length of 160 bits, whereas this draft specification provides a hash length of 256 bits. No consideration has been made as to whether there is a consequent benefit of increasing K to compensate for the increased sparseness of the hash table, or other consequences of the increased hash length.

## Distance calculation

The DHT requires a distance function of the form `dist GUID, GUID -> Integer`. This draft specification uses the same distance function used in Kademlia: the XOR of each GUID, interpreted as an unsigned integer.

## Hashing

This draft specification uses an implementation of SHA512/256 (https://eprint.iacr.org/2010/548.pdf) for the DHT, which is a 256-bit application of SHA-512, motivated by the interesting property of modern 64-bit CPUs to compute SHA-512 hashes more efficiently than they do SHA-256 hashes.

This also foils a seemingly esoteric partial-message collision attack against MD4, MD5, SHA-1 and SHA-2 described in "Cryptography Engineering" by Ferguson, Schneier and Kohno.

## Choice of security level

This protocol modifies the Kademlia implementation used by BitTorrent by basing node GUID on public key, which nodes are required to prove ownership of in each transaction. Consequently, there is substantial CPU overhead in this protocol compared to BT.

The purpose of this is to prevent nodes from selecting arbitrary GUIDs, by which they can accomplish censorship of discussion groups by positioning themselves as the nodes whose GUIDs have minimal distance to the discussion group in question, and then intentionally return incorrect peerlists or refuse to process announce requests.

As of this writing, no effort has been made in this drafting process to evaluate the implications of the CPU cost of adding cryptography. A specific concern is whether, under typical traffic loads that will be experienced by peers, the added CPU cost will be heavy enough to discourage users from participating in the DHT.

The CPU cost is directly related to the security level offered by our choice of algorithms; this document currently specifies 256-bit ECDSA, and SHA512/256. Each of these algorithms are presently believed to offer a security level of 128 bits. This may be excessive, considering that the sole object of this security is to make it infeasible for attackers to choose arbitrary GUIDs.

A lower security level may suffice to make malicious GUID selection impractical, while reducing the resource cost of application.

