Research and Development in Bitcoin and Crypto
==============================================

The following papers are result of solving real problems in real applications. Most of them are fully invented by Oleg Andreev, some improve the existing ideas. Papers will be updated when better ideas come along. Feedback is welcome.

#### [Bitcoin Glossary](BitcoinGlossary.md)

The most complete collection of Bitcoin technical terms.

#### [Bitcoin Blind Signatures](BitcoinBlindSignatures.md)

Distribute control over bitcoins between multiple custodians while maintaining absolute privacy. Custodians only authenticate you in person and allow or deny a transaction, but never know the contents of the transaction.

#### [Automatic Encrypted Wallet Backups](AutomaticEncryptedWalletBackups.md) (BIP proposal)

Bitcoin wallets normally require careful backup for only its master seed. The scheme allows to encrypt all external information (payment receipts, notes etc.) using that master seed and automatically upload to one or more hosting services (iCloud, Dropbox, Google Drive) to be automatically downloaded and installed during restore process.

#### [Joint Escrow](JointEscrow.md)

A simple multi-signature scheme for establishing game-theoretic trust between two otherwise untrusted entitites (persons or computer agents). This is a basic building block for many decentralized markets and applications such as [Bitmarkets app](http://voluntary.net/bitmarkets/).

#### [Bitcoin Wallet API](BitcoinWalletAPI/core_spec.md)

Many end-user Bitcoin applications (e.g. [Lighthouse](https://www.vinumeris.com/lighthouse), [Bitmarkets app](http://voluntary.net/bitmarkets/)) have to implement a wallet functionality just to use some bitcoins for their operation. This is typically less safe for the user than the wallet he is already familiar with. We propose a set of secure APIs that allow apps to get some bitcoins from the user's wallet in a secure and flexible manner.

#### [Two-Factor Wallet Backup](TwoFactorWalletBackup.md)

Bitcoin wallet's master key must be protected against compromise and data loss. We propose a solution that does not require user to remember or write down any additional secret information, yet provides two-factor protection against both theft and loss.

#### [Decentralized Payment Network](PaymentNetwork.md)

Blockchain is great for ultimate settlement of ownership, but it does not efficiently scale to billions of regular transactions. We propose an overlay protocol that enables clearing small payments via a decentralized infinitely scalable network of IOU swaps backed by actual bitcoins locked up using [Joint Escrow](JointEscrow.md).

#### [Smart contracts extension in Bitcoin](SmartContractsSoftFork.md)

Proposal to extend scripting capabilities in Bitcoin via a soft fork that allows access to wider amount of information about transactions and persistent state across contracts to enable massive multi-party execution at low cost.


About
-----

Oleg Andreev is an entrepreneur and software designer who brings order and sanity to Bitcoin world.

Email: oleganza@gmail.com

Twitter: @oleganza

Github: [github.com/oleganza](https://github.com/oleganza)


License
-------

[Creative Commons Attribution License v4.0](http://creativecommons.org/licenses/by/4.0/)

You can freely copy, share and build things based on these ideas, but must give appropriate credit to Oleg Andreev as an author. You don't get to restrict others from doing the same thing.

Have a nice day!
