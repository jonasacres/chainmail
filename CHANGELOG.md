# Changelog

Update this to provide brief descriptions of the rationale and nature of changes to PROTOCOL.md.

## 2015-05-12 07:45 jonasacres
Added PermissionGroup to ease bulk management of member permissions. Added "Rules" section for many record types to explain validation rules that must be applied when considering whether a record is valid. Added "reaction:up" and "reaction:down" permissions.

## 2015-05-10 08:30 jonasacres
Edited record header to make it possible to "clone" a community, by replacing `creator` field (a reference to the hash of the creator's current identity) from `cm_record_header`, and removing `authority`. This removes all references to specific hashes in the blockstore from the record header, making records portable. Creators are now identified by `author_guid` and `author_pkey_hash`, which are the author GUID from their Identity record, and the hash of the public key used to actually sign the record.

The dropped `authority` reference in `cm_record_header` has no replacement. This reference served no purpose, since client seeking to validate the records in a block must search backwards from that block to find the latest credential record for a signer, since a referenced credential may have been deleted or revoked. It was a nice foundational piece for a possible implementation of a finite token system, by which users could be allowed to exerise a privilege they don't normally own a limited number of times -- one application for this being an invite system, but no such system is presently specified.
