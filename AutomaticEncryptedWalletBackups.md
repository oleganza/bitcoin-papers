Automatic Encrypted Wallet Backups (DRAFT)
==========================================

<pre>
  BIP: not assigned yet
  Title: Automatic Encrypted Wallet Backups
  Author: Oleg Andreev <oleganza@gmail.com>
  Status: Draft
  Type: Standards Track
  Created: 2015-02-16
</pre>

Abstract
--------

This BIP describes a scheme to encrypt, identify and decrypt automatic wallet backups using a wallet's master key. The scheme is intended for seamless and secure restoration of the full wallet information using just the wallet's master key.

Motivation
----------

Hierarchical wallet scheme [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) allows user to back up the master key only once in a convenient location under user's control. User can safely send and receive funds without a need to back up his wallet after every transaction. Later all funds can be restored with just a single master key that could have been backed up in the past.

The problem arises when the wallet needs to store additional metadata about transaction (e.g. notes, payment receipts or P2SH redeem scripts), those must be backed up as soon as possible or even as a prerequisite to sending a transaction. Frequently asking a user to back up their wallet would be inconvenient. However, we may encrypt the wallet with high-entropy key derived from the wallet's master key and back up the encrypted wallet on a 3rd party service (or services) automatically. This automatic backup could be retrieved and decrypted with a master key when it is restored from user's personal backup.

Since automatic backups are not encrypted with typically weak user passwords, but with keys with at least 128 bits of entropy, it is safe to put such backup anywhere. The only requirement is availability: the service must be available when the user requests the backup.

Overview
--------

1. Wallet app stores all metadata (labels, payment receipts, redeem scripts etc.) in some local storage.
2. Every time app updates that data, it is serialized (e.g. JSON or SQLite file) and encrypted and signed according to this specification.
3. The resulting object is uploaded to several storage providers. E.g. in case of an iOS wallet it could be both iCloud Drive (via CloudKit) and wallet provider's web server.
4. When a person restores his wallet using his master seed, wallet app attempts to download the latest available backup from all storage provider and installs it.
5. The wallet is fully restored with no data loss.
6. Wallet app may periodically request a merkle path for random portions of the backup to verify that the backup is still stored. Since the backup is organized in a merkle tree only the root of which is signed, minimal bandwidth is needed for verification.

Definitions
-----------

`"String"` is a binary string containing ASCII-encoded text.

`a || b` denotes concatenation of two binary strings.

**Plaintext** is a serialized represenation of Wallet's metadata. This could be JSON, XML, SQLite or any other data. This specification does not cover the underlying data format. Each wallet application defines that format for itself.

**Master Key** is a 32-byte raw private key contained in the wallet's master extended private key (per BIP32).

**Backup Key** is a key derived from the *Master Key* from which all other backup-related keys are derived. Wallet should store that key in a secure location that does not require user interaction to be accessed. (Unlike *Master Key* that should be stored in a location that does require interactive authentication like passcode or Touch ID.) All the keys defined below are derived on the fly from the single *Backup Key*. 

    BackupKey = HMAC-SHA256(key: MasterKey, data: "Automatic Backup Key")

**Wallet Identifier** is a 16-byte string used to identify a wallet backup when storing and retrieving it. It is not included in the backup itself and only used to reference the backup in a standard way when communicating with a storage service.

    WalletID = HMAC-SHA256(key: BackupKey, data: "Wallet ID")[0, 16]

**Authentication Key (AK)** is used to verify the integrity of the encrypted backup. This key is computed as follows:

    AK = HMAC-SHA256(key: BackupKey, data: "Authentication Key")

**Encryption Key (EK)** is a 16-byte encryption key for AES-128-CBC algorithm with PKCS7 padding. It is defined as follows:

    EK = HMAC-SHA256(key: BackupKey, data: "Encryption Key")[0, 16]

**Initialization Vector (IV)** is a 16-byte unique string per backup used in AES-128-CBC cipher. It is derived from the *Encryption Key* and the actual metadata *Plaintext*.

    IV = HMAC-SHA256(key: EK, data: Plaintext)[0, 16]

Note that *IV* also acts as a plaintext MAC, a property that we use to protect against mutable *Merkle Tree* (see below).

**Ciphertext** is a AES-128-CBC transformation of the *Plaintext* padded to whole 16-byte blocks according to PKCS7.

    CT = AES-128-CBC-PKCS7(data: Plaintext, key: EK, iv: IV)

**Hash256** is a double-pass SHA-256:

    Hash256(data) = SHA256(SHA256(data))

**Merkle Root** is a 32-byte root of the *Merkle Tree* built from the *Ciphertext*. For simplicity, we use the same merkle tree as used in Bitcoin blocks. Since this merkle tree is vulnerable to mutations (if last node is odd, it can be duplicated without altering the root hash), client will need to recompute and verify *IV* values as they also act as plaintext MAC. The *Ciphertext* is interpreted as a list of 1024-byte chunks that are hashed using *Hash256*. The final chunk length of the *Ciphertext* may be less than 1024. Per Bitcoin Merkle Tree specification, in case any intermediate list of elements (chunks or hashes) is odd, the last element is duplicated. If the entire ciphertext length is 1024 bytes or less, then its *Hash256* is a merkle root.

Example. A text of length 5000 bytes is broken down in 5 chunks:

    Ciphertext = a (1024) || b (1024) || c (1024) || d (1024) || e (904 bytes)

The list has odd number of chunks, so last element is duplicated (*H* = *Hash256*):

    f = H(a || b)
    g = H(c || d)
    h = H(e || e)

The resulting list is of odd length again, so the last element is duplicated:

    p = H(f || g)
    q = H(h || h)

Finally, the last pass produces a single hash which we call a *Merkle Root*:

    MerkleRoot = H(p || q)

**Timestamp** is a 4-byte unsigned integer containing UNIX timestamp of the moment when backup was created. It allows wallet to evaluate how recent the backup is.

**Version Byte** is a byte of value 0x01 indicating the version of this specification.

**Signature** is a 32-byte string authenticating the *Merkle Root* of the *Ciphertext*. It is defined as follows:

    Signature = HMAC-SHA256(key: AK, data: (VersionByte || Timestamp || IV || MerkleRoot))

**Backup Payload** is a variable-length string containing a fully serialized encrypted backup:

    BackupPayload = VersionByte || Timestamp || IV || Ciphertext || Signature

The location and length of the Ciphertext:

    Offset = 1 + 4 + 16 = 21
    Length = Length(BackupPayload) - (1 + 4 + 16 + 32)


Creating Backup
---------------

1. User creates a wallet by generating or importing a *Master Key*.
2. Wallet computes *Backup Key* from *Master Key* and stores it in a secure location that does not require interactive access.
3. Whenever new information is added that must be backed up (e.g. new payment receipt), wallet serializes all the data needs to be backed up and computes *BackupPayload*.
4. Wallet stores *BackupPayload* locally so it can periodically request *Partial Merkle Branch* (per [BIP 37](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki#Partial_Merkle_branch_format)) proofs of storage from the service that supports that.
5. Wallet uploads *BackupPayload* using *Wallet Identifier* to one or more storage services. The service does not need to store the previous versions of backup for that wallet because it must return the latest one when requested.
6. If backup contains critical data for future operation (e.g. P2SH redeem scripts or external public keys), wallet should first ensure the backup is securely uploaded before sending out any transaction that relies on that data.

Restoring Backup
----------------

1. User restores the wallet by importing the *MasterKey*.
2. Wallet computes *Wallet Identifier*, *Authentication Key* and *EncryptionKey*.
3. Wallet downloads the latest copies of the backup from all available services.
4. Wallet verifies *Signature* of each backup using *Authentication Key*.
5. Wallet chooses the latest copy of the backup based on *Timestamp* and imports it.
6. Wallet stores *Backup Key* in a secure location that does not require interactive access to encrypt and sign future backups.

Rationale
---------

**Why 128-bit encryption?**

128-bit AES key is chosen since AES actually has a lousy 256-bit key scheduler which is surprisingly [weaker than the 128-bit one](https://www.schneier.com/blog/archives/2009/07/another_new_aes.html).

**Why not asymmetric encryption and signature scheme (e.g. ECIES)?**

At first sight it seems useful to only keep a public key in a less protected space of the app to encrypt and update the backup, but to require user's authentication (e.g. via TouchID on iOS) to retrieve a private key to decrypt the downloaded backup. However, that does not really buy any additional security. Attacker who can potentially get access to such a public key would not be able to decrypt the backup, but most likely would have access to the same private data stored in the same location as that public key. Also, accessing that public key allows attacker to overwrite backup.

**Why using Bitcoin Merkle Tree instead of a safer one?**

The vulnerability in Bitcoin's Merkle Tree is easy to mitigate and reusing existing implementation greatly reduces complexity of the scheme. Verifying decrypted plaintext using IV-as-MAC has virtually no extra overhead. Alternative Merkle Tree algorithm would require costly review, testing and security analysis and may potentially introduce vulnerabilities of its own.


Incremental updates
-------------------

We expect a modest amount of metadata to be stored (below 1-2 Mb), so all updates can be done by uploading the whole *Backup Payload*. However, if the wallet needs to break down backups in a chain of independent chunks, it is possible to do without changing the scheme. 

Once the backup becomes too big (e.g. approaches 1 Mb), wallet may generate another *Wallet ID* for the second part and store only additional data that is not already stored in the first part. Next parts will identified by linking to exact versions of the previous parts using their *Signatures*:

    WalletID1 = HMAC-SHA256(key: BackupKey, data: "Wallet ID")[0, 16]
    WalletID2 = HMAC-SHA256(key: BackupKey, data: "Wallet ID" || Signature1)[0, 16]
    WalletID3 = HMAC-SHA256(key: BackupKey, data: "Wallet ID" || Signature2)[0, 16]
    ...

Using this scheme, wallet first downloads backup identified by *WalletID1*, then using its *Signature1* computes *WalletID2* and attempts to download second part and so on.

Once the part 2 is uploaded, part 1 should not be updated. Otherwise, wallet may not be able to retrieve exact version identified by *Signature1*.

Protecting against stale backups
--------------------------------

The wallet trusts its service providers to return the latest version of backup identified by *Wallet ID*. To make sure that the latest version is always returned, the wallet needs a trusted reference point. One way to achieve that is to use Blockchain and put a signed timestamp in OP\_RETURN output in every wallet's transaction:

    SignedTimestamp = Timestamp || HMAC-SHA256(key: AK, data: Timestamp)[0, 16]

Wallet adds OP_RETURN output with 20-byte *SignedTimestamp* whenever the backup has changed (backup must be updated before the transaction is sent). If no new data has been added, no additional output is needed.

Total overhead of the *SignedTimestamp* output is 30 bytes (20 bytes for itself and 10 bytes for output amount and script prefix).

For incoming transactions, wallet may use Payment Requests (BIP70) to provide a desired output. It becomes a courtesy of the payer to include that output. This makes sense when the wallet already uses Payment Requests and needs to backup associated notes or invoices.

Alternatively, if the backup is updated not because of the new outgoing transaction, wallet can make a transaction paying to itself. When backup is updated with critical information, such transaction should be sent right away. For less critical data, only periodic timestamping (e.g. once a day/week) may be performed.

To take advantage of this scheme, restore process must be changed: before installing the backup, wallet must first fetch all of its transactions and determine the latest signed *Timestamp*. If blockchain-recorded timestamp equals or below the timestamp in the available backup, then the backup can be applied automatically. If not, then user must be warned that the latest backup is not available. 

For simplicity, *SignedTimestamp* does not sign the transaction itself (which is not possible for incoming transaction anyway). Therefore it becomes possible for someone to reuse previous *SignedTimestamps*. To mitigate that, wallet should not rely on the order of *SignedTimestamps* (as older timestamps may be added by someone in the newer transactions), but merely use the one with the highest timestamp value.

Storage Service Authentication
------------------------------

This specification does not cover how the storage service authenticates access to its resources. Every service may choose its own method. 

For instance, iCloud allows access with a valid Apple ID account and its databases are limited by a per-user quota (for private database) and per-developer quota (for public database). Other services may employ rate limiting, size limiting, require payments or some sort of a proof-of-work scheme.


Implementations
---------------

TODO: list implementations in various languages and libraries.


Test Vectors
------------

TODO: generate test vectors for backups of various lengths (5 bytes, 17 bytes, 2000 bytes, 7000 bytes).


References
----------

* [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) — hierarchical deterministic wallets.
* [BIP 37 Partial Merkle Branch Format](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki#Partial_Merkle_branch_format)
* [AES-256 weakness](https://www.schneier.com/blog/archives/2009/07/another_new_aes.html) explained by Bruce Schneier.

