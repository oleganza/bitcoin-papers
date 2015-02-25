
Bitcoin Wallet JavaScript API Spec
==================================

This specification defines JS API for web browsers that implements the [Bitcoin Wallet Core Specification](core_spec.md). Using this API any web page may request payment from user's wallet using JavaScript interfaces provided by the web browser. 


TODO: write up on JS functions, structures and callbacks for each of the APIs.

Note 1: Wallet must return a list of signatures, but JS API should put them automatically in user's transaction. So the app may simply trust the transaction but not trust the wallet.

Note 2: Key API should support 3 functions: pubkey request for a given index, ECDSA signature for a given hash and DH pubkey for a given pubkey.

