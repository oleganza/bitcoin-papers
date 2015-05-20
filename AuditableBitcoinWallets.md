BIP Draft: Auditable Bitcoin Wallets
====================================

    Date: May 17, 2015.
    Author: Oleg Andreev <oleganza@gmail.com>
    Status: Draft

Hardware Bitcoin wallets are obvious targets for backdoors. We propose a unified method of auditing any wallet for presence of potential backdoors or bugs. The specification also applies to software wallets executed by general-purpose computers, although these may be more challenging to audit.

Motivation
----------

Many cryptographic primitives (hash functions, symmetric ciphers, RSA and ECC) generate pseudo-random (unpredictable) sequences absolutely deterministically. This means that it is very easy to hide backdoors behind seemingly "random" data. The goal of this document is to specify how the wallet's operation can be audited without wallet's knowledge to prove that wallet does not and cannot communicate unauthorized information.

This specification applies equally to specialized hardware and general-purpose computers running wallet software. Access to source code is not required. Published source code is of no guarantee that exactly same code is being executed on any given device.

Specification
-------------

#### Practical Cryptographic Assumptions

We assume that wallets implement correctly cryptographic standards and underlying cryptographic primitives are not breakable in practice. Protection against theoretical and practical weaknesses in hash functions, elliptic curve algorithms and block ciphers are beyond the scope of this paper.

#### Monitored Output

The wallet's output should be easy to monitor. It must not be opaquely encrypted (e.g. going through TLS connection to wallet provider's server). Communication should be either in plain text, or encrypted deterministically based on master secret and user-provided data. If the device has multiple methods of output (wired, bluetooth, wifi etc), all of them must be documented and equally auditable. 

Note that data stored on disk must also be auditable and therefore compliant with this specification.

#### Master Seed

Throughout this document we assume that wallet generates only one unpredictable secret ("master seed") and all subsequent output is strictly determined by this secret and/or user-provided data. For wallets based purely on user-provided data all requirements are reduced to having a constant seed equal to some pre-defined value (e.g. empty string). Wallet that use multiple generated secrets have to apply the same procedure to every secret.

Process of creating the master seed should be functionally equivalent to the following steps:

1. Wallet generates a pseudo-random number (128 bits or more) that we call **app entropy**.
2. Wallet unconditionally displays app entropy to the user (in binary, hex, as a bit pattern etc).
3. User is then asked for some data to be entered that allows to improve security in case app entropy generator has a backdoor (see [Dual EC DRBG](http://en.wikipedia.org/wiki/Dual_EC_DRBG)). This could be a one-time text entered by the user only for this purpose, or a passphrase containing enough entropy that is used later for other purposes.
4. Wallet uses computationally expensive KDF to deterministically mix app entropy and user-entered text to produce the final **master seed**. There is no need to display the resulting seed as auditor can arrive at the same value independently.

Important considerations:

* App entropy must be displayed unconditionally so that wallet cannot know if user is going to audit it or not.
* User-entered data should be visible and reproducible by the user to be able to arrive to the same data in an auditing software. Good: text or pattern of button presses. Bad: accelerometer data, mouse movements (hard to record and reproduce).
* User-entered data should contain enough entropy. If the wallet has a backdoored RNG which generates sequence of numbers predictable to attackers, it becomes trivial to use public blockchain data to bruteforce user input (especially when it is a short passphrase). Wallet should be capable of accepting input with at least 128 bits of entropy. E.g. wallet that uses 4-digit code to mix with app entropy does not comply with this specification.
* Unless necessary, wallet should not rely solely on user input to generate master seed because it may have inadequate entropy. In such case, honest wallet with correct RNG implementation would generate secure master seed.
* Process of mixing *app entropy* and *user input* must use computationally expensive KDF to compensate for low entropy in user input.

#### Coin Selection

When composing transactions, wallet must use a deterministic algorithm to select unspent transaction outputs (or accept user-provided unspent outputs). Wallets are free to implement any algorithm depending on privacy or efficiency requirements as long as it is completely reproducible by the auditor. Pseudo-random shuffling of unspents can be done using HMAC with Master Seed.

#### Canonical Transaction Encoding

Wallet generally should produce canonical encoding of all data structures within a transaction (see [BIP 62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki), [BIP 66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki)). If not possible or not relevant, encoding should still be deterministic and well-documented by the wallet developer.

#### Deterministic ECDSA Signatures

ECDSA standard is notable for use of unpredictable nonce per signature that must not only be reused, but should be completely unpredictable to observers. This specification requires that ECDSA signature uses a deterministically generated nonce. Many libraries in different programming languages already support [RFC 6979](https://tools.ietf.org/html/rfc6979) which defines a standard way to do that.

#### Deterministic Address Generation

All public keys and addresses generated by wallet must be deterministic. The most commonly used algorithm is defined by [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki). If the wallet needs to generate a truly random address, key or other data, it should follow the same procedure as with Master Seed generation to make the process auditable. Since it is a cumbersome process for the user, it is recommended to have only one random value (Master Seed) and derive the rest from it using cryptographic hash functions.

#### Deterministic Input and Output Shuffling

It is a common practice to shuffle outputs to hide information about which output is payment and which one is change. Whenever wallet needs to shuffle either input or output, shuffling must be done deterministically. For instance, using result of HMAC(master seed, serialized data) as a sort descriptor.

#### Preventing Timing Attacks

This specification does not address timing attacks (when the wallet communicates secret bits via delays in emitting the data). We assume that the wallet emits transaction triggered by an external event outside of attacker's control (e.g. when user makes a payment). This specification will be extended with concrete scenarios of timing attacks and methods to prevent them when such scenarios are identified and explored.

Suggested Algorithms
--------------------

This is a non-normative list of suggested algorithms to ensure best interoperability of auditing tools. Wallet developer may choose different algorithms as long as they provide complete documentation and tools enough to audit their wallet.

#### Master Seed Generation

    MasterSeed = Scrypt(password: User Data, salt: App Entropy), where other parameters are wallet-specific

#### Coin Selection

Wallet selects unspent outputs in order of age. Unspent outputs sharing the same block are sorted by their order in transactions list and index within the transaction. Unconfirmed outputs are sorted by the binary hash of their transaction and index. Note: while simple, this algorithm does not provide optimal privacy (does not try to keep one unspent output per transaction).

#### Transaction Encoding

Use [BIP 62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki) and [BIP 66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) to ensure deterministic and canonical encoding of the transactions.

#### ECDSA Signatures

Use [RFC 6979](https://tools.ietf.org/html/rfc6979) to generate a deterministic nonce. Then use [BIP 66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) to ensure canonical form of the S value.

#### Address Generation

Use [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) or more specific [BIP 44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki).

#### Sorting Outputs

Let `SortingKey = HMAC-SHA256(data: "Deterministic Sorting Key", key: MasterSeed)` which can be stored separately from a Master Seed, potentially in a less secure location on a device.

Use `HMAC-SHA256(key: SortingKey, data: SerializedOutput)` as a sorting descriptor (in ascending order).

#### Sorting Inputs

Let `SortingKey = HMAC-SHA256(data: "Deterministic Sorting Key", key: MasterSeed)` which can be stored separately from a Master Seed, potentially in a less secure location on a device.

Use `HMAC-SHA256(key: SortingKey, data: SerializedInput)` as a sorting descriptor (in ascending order). Inputs may be signed, half-signed or completely unsigned, but that must be deterministic.


Applicability
-------------

This document primarily applies to wallets managing a single user-stored key and issuing transactions to externally specified addresses and amounts. 

* Hardware wallets like Trezor, Ledger and Mycelium Card.
* Software wallets like Bitcoin Core (Bitcoin-QT), Multibit, Breadwallet, Mycelium, Hive.
* Toolkits and libraries like CoreBitcoin, btcd, Bitcoin-Ruby, BitcoinJ that allow creating aforementioned wallets.

#### Exemptions

In some cases it is impossible or impractical to apply this specification fully or at all. For instance, a multisig wallet may have all transactions generated by a third party that cannot reveal the private keys for full audit. However, all pieces of information controllable by the keys held by the user could still be audited according to present specification. Obviously, systems that completely isolate end user from cryptographic keys (e.g. banks and exchanges) cannot be made compliant to this specification.

Deployment
----------

This specification should be reviewed for correctness and completeness by independent security experts, hardware and software wallet developers.

After **November 30, 2015** all applicable hardware and software wallets and toolkits (libraries) shall be considered **broken** unless they conform to this specification. Authors of these products have over 6 months to add support for auditablity on API and UI level. If they decide not to, they are guilty of **intentionally putting their customers under risk**.

Applicable products released after **May 17, 2016** (12 months since publication) not conforming to this specification shall be considered **intentionally malicious and backdoored**.

Current Status
--------------

* None of software or hardware wallets fully conform to this specification.
* TREZOR hardware wallet implements auditable entropy, canonical encoding and even constant-time signing. However, entropy is not displayed unconditionally (yet).
* Mycelium iOS wallet v1.2 uses CoreBitcoin library that implements determinstic address generation (BIP32 and BIP44), deterministic signatures (RFC6979) and deterministic shuffling of outputs (non-standardized).
* Many popular Bitcoin libraries implement canonical encoding of transactions and RFC6979.

If you believe your wallet is fully auditable according to this specification, please add it to this list.

If you propose a different algorithm to be used as a recommendation, please file a pull request.

