# Chainmail Storage

This draft specification provides for the secure storage of data from Chainmail blockchains.

## Filesystem
All paths are listed relative to a designated Chainmail directory, e.g. `~/chainmail`.

### `chainmail/global/masterkey`
Randomly-generated AES256 key, encrypted using a key derived from a passphrase using a to-be-selected KDF (eg. Argon2d, TODO).

### `chainmail/global/mastersalt`
512-bit random salt, encoded in lowercase hexadecimal ascii. Encrypted using `masterkey`.

### `chainmail/keys/:keyid`
Directory containing information pertaining to a keypair known to this Chainmail installation. The keyid is derived from `Hash(Hash(PublicKey) || mastersalt)`.

### `chainmail/keys/:keyid/rekey`
Random AES256 key, encrypted using `masterkey`.

### `chainmail/keys/:keyid/public`
Public key, encoded as specified in `BLOCKCHAIN.md`, and encrypted using `rekey`.

### `chainmail/keys/:keyid/private`
Private key, encoded using the same technique as the public key. Private keys are encrypted using AES-256 in Galois/counter mode. The key for this cipher is derived from a user-supplied passphrase, using a yet-to-be-specified key derivation function such as Argon2d or PBKDF2 (TODO). This ciphertext is then re-encrypted using `rekey`.

Optionally, the user my elect not to set a passphrase on the private key. In this case, the key is still encrypted, using `Hash(PublicKey)` as the passphrase, encoded in lowercase hexadecimal ascii.

### `chainmail/stores/:cryptoid`
This directory contains all the information pertinent to a given blockstore. The `:cryptoid` identifier is derived from the hash of the root block, and is obtained by computing `Hash(RootBlockHash || MasterSalt)`.

### `chainmail/stores/:cryptoid/readkey`
A keyhash (the hash of a public key), encrypted with `masterkey`. This keyhash identifies the keypair used to participate in this blockchain as a reader.

### `chainmail/stores/:cryptoid/storekey`
A randomly-generated AES-256 key specific to this blockstore, encrypted using the public key referenced in the corresponding `readkey`.

### `chainmail/stores/:cryptoid/writekey`
A keyhash (the hash of a public key), encrypted with `storekey`. This keyhash identifies the keypair used to participate in this blockchain as a writer.

### `chainmail/stores/:cryptoid/rosettakey`
A randomly-generated AES-256 key, encrypted using `storekey`. This key is regenerated whenever `rosetta` is regenerated.

### `chainmail/stores/:cryptoid/rosettarand`
The plaintext stored within `rosetta`. Encrypted using `rosettakey`.

### `chainmail/stores/:cryptoid/rosetta`
The current Rosetta table, as used in the swarming protocol. The file is encrypted using AES-256-GCM, using `rosettakey` as the key. The file contains random padding to pad the length to a minimum of 128KB.

### `chainmail/stores/:cryptoid/latest`
`padding:index:hash`, where `index` is the index of the last stored block, `hash` is the hash of this block and `padding` is arbitrary random padding. Encrypted using `storekey`.

### `chainmail/stores/:cryptoid/records/:cryptorecordhash`
This directory contains information pertaining to a specific record. The `:cryptorecordhash` is obtained by computing `Hash(RecordHash || MetaKey)`.

### `chainmail/stores/:cryptoid/records/:cryptorecordhash/recordkey`
A randomly-generated AES-256 key specific to this record, encrypted using the `storekey`.

### `chainmail/stores/:cryptoid/records/:cryptorecordhash/header`
Contents of the `cm_record_header` for this record, encrypted using `recordkey`, plus additional padding.

### `chainmail/stores/:cryptoid/records/:cryptorecordhash/payload`
Contents of the record payload, encrypted using `recordkey`, plus additional padding.

## Fictitious activity

### Fake records
Chainmail randomly creates fictitious records (i.e. ones which are not referenced in any block). These records can be of arbitrary number or size, allowing Chainmail to disguise a given blockstore as being one of arbitrary size and activity. When creating these fictitious records, the `latest` file for the blockstore is regenerated with new padding. Periodically, the `rosettakey`, `rosettarand` and `rosetta` files are regenerated as well.

### File Timestamps
Because the filesystem may store metadata about files, including access, modification and creation timestamps, some information may be divulged about the timing of receipt of certain records. Chainmail contains countermeasures to disguise these timestamps from an adversary.

#### Access and modification times
Chainmail automatically sets access and modification times to arbitrary dates.

#### Creation times
Chainmail randomly recreates files and directories to obscure the creation time in the filesystem.

## Plaintext records
It is conceivable that users may have `File` records, like documents or images, that they may wish to open in an external application. One solution is for the user (or the frontend aplication) to export this file elsewhere on disk, in plaintext for, disconnected from its origins in Chainmail.

This has the inconvenient side-effect of requiring duplicate storage. For users who are willing to sacrifice confidentiality, Chainmail offers thea ability to convert a record to plaintext. In this case, the `chainmail/stores/:cryptoid/records/:cryptorecordhash/recordkey` file is not present, and the header and payload are left unencrypted and stripped of padding. For File records, a symlink to `payload` is created within the record's directory, using the suggested filename. The user can then create additional hardlinks or symlinks anywhere else on their filesystem.

## Secure deletion
Prior to unlinking an object from Chainmail's filesystem, applicable keyfiles should be first overwritten with random bytes and flushed to disk. Then unlinking can take place. Applicable keyfiles are listed below.

| Object        | Files
|---------------|---------
| Record        | `recordkey`
| Rosetta       | `rosettakey`
| Blockstore    | `storekey`, `readkey`
| Keypair       | `rekey`
| Everything    | `masterkey`