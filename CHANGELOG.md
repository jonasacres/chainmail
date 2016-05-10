# Changelog

Update this to provide brief descriptions of the rationale and nature of changes to PROTOCOL.md.

## 2015-05-10 08:30 jonasacres
Edited record header to make it possible to "clone" a community, by moving creator from `cm_record_header` to `cm_record_ref`. This removes all references to specific hashes in the blockstore from the record header, making records portable. This does have the effect of significantly increasing the size of `cm_record_ref`. This would have the effect of removing any identity information from the `cm_record_header`, which means that the `cm_record_ref` would be necessary to validate the signature. To mitigate this, the author's GUID and current public key hash are referenced in the record header. These remain invariant even in clones, and provide necessary information to identify an owner's key in the blockstore.

Dropped "authority" reference in `cm_record_header`. Any client seeking to validate the records in a block must search backwards from that block to find the latest credential record for a signer, since a referenced credential may have been deleted or revoked.
