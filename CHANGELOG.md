# Changelog

Update this to provide brief descriptions of the rationale and nature of changes to PROTOCOL.md.

## 2015-05-17 14:50 jonasacres
Broke Identities into two public keys: a signing key, and a read-only key used for peering. The purpose of this is to allow a user to keep a system running monitoring blockstore data, without that also necessarily having constant access to the key needed to appear as the user.

## 2015-05-17 14:40 jonasacres
More work on SWARM.md. Added STORAGE.md.

## 2015-05-16 23:00 jonasacres
Broke PROTOCOL.md into multiple files. Started work on defining swarm algorithm.

## 2015-05-12 12:00 jonasacres
Add a `venue` field to `cm_record_header`. This ensures that if a Chainmail group is cloned, all records can be brought in -- and yet, it is clear that the cloned forum is not the original forum, and it is additionally clear how the original forum might be found in the DHT (if the forum is public).

Added a rule that successive Identity records for a given GUID may only change that GUID's public key if the record is signed by the key that is to be replaced. In other words, there is no privilege in the Chainmail granting the right to change someone else's public key. This foils an attack in which someone clones a Chainmail group, and issues new Identity records for certain members in order to impersonate them.

The cost of this is that once a member loses their public key, there is no way for anyone to rescue their identity. They must create a new identity. The server-based analogy to this would be a forum with no password recovery feature, and no ability for administrators to reset passwords.

A possible workaround to this would be to embed a hash field `recovery` into the identity record. A new identity record can change the `pubkey` field without being signed by the previous public key, so long as `hash(new_identity.recovery) == old_identity.recovery`. In this way, there can be a secondary secret known to the identity-holder which can be used with the assistance of a user with the `identity!!` permission to recover the identity. However, once this recovery secret is disclosed to this privileged user, that privileged user would have unchecked ability to hijack the identity.

--

Earlier I also edited the way privmsg records store public keys in their rosetta tables. Previously, the symmetric key was encrypted directly using each recipient's public key. Not being familiar with the cryptanalytic implications of this in any particular asymmetric cipher, let alone all of them, I am suggesting a measure of caution: encrypting the same message into a variety of keys may somehow disclose information, since attackers will be aware that each ciphertext in the rosetta table contains a common plaintext. This may be a non-issue; notably, GPG does not appear concerned with this.

That said, it may be prudent to provide arbitrary padding to alter the plaintext between each recipient.

## 2015-05-12 07:45 jonasacres
Added PermissionGroup to ease bulk management of member permissions. Added "Rules" section for many record types to explain validation rules that must be applied when considering whether a record is valid. Added "reaction:up" and "reaction:down" permissions.

## 2015-05-10 08:30 jonasacres
Edited record header to make it possible to "clone" a community, by replacing `creator` field (a reference to the hash of the creator's current identity) from `cm_record_header`, and removing `authority`. This removes all references to specific hashes in the blockstore from the record header, making records portable. Creators are now identified by `author_guid` and `author_pkey_hash`, which are the author GUID from their Identity record, and the hash of the public key used to actually sign the record.

The dropped `authority` reference in `cm_record_header` has no replacement. This reference served no purpose, since client seeking to validate the records in a block must search backwards from that block to find the latest credential record for a signer, since a referenced credential may have been deleted or revoked. It was a nice foundational piece for a possible implementation of a finite token system, by which users could be allowed to exerise a privilege they don't normally own a limited number of times -- one application for this being an invite system, but no such system is presently specified.
