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
A Chainmail community can be started by anyone. At its heart, a Chainmail community is a blockchain, similar to what is used by Bitcoin. For more information on the blockchain aspect of Chainmail, see the `protocol-docs/BLOCKCHAIN.md` document.

Each block within Chainmail contains a recordset. These records define actions within the network, like adding a new member to the community, or sharing a message. For more information on the kinds of records supported within Chainmail, see the `protocol-docs/BLOCKCHAIN.md` document.

Peers within Chainmail relay new blocks to each other through a peer-to-peer swarming protocol. For more information on this, see the `protocol-docs/SWARM.md` document.

A cryptographic hash (such as SHA-512/256) of the root block of a Chainmail community creates an objective, non-duplicable identifier for a community. This identifier also has the property that it discloses no information about the community itself. This ID hash can be searched in a distributed hash table (DHT) not unlike that used in BitTorrent. In so doing, community members who lack peering information to actually participate in the community can use the DHT to join the community swarm, without disclosing any information about the community itself to non-members. For more information on this, see the `protocol-docs/DHT.md` document.

## Acknowledgments

This draft specification borrows generously from concepts in the Bitcoin and BitTorrent projects, as well as the Kademlia distributed hash table. In particular, the following documents have proven particularly illuminating:

* http://www.bittorrent.org/beps/bep_0005.html
* https://en.wikipedia.org/wiki/Kademlia
* http://www.academia.edu/3424033/BitTorrents_Mainline_DHT_Security_Assessment

Many thanks to the contributors to Tor, Bitcoin, BitTorrent, as well as the many researchers whose work has been very informative in drafting this specification.
