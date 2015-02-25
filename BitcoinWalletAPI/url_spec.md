
Bitcoin Wallet URL Spec
=======================

This specification extends BIP 21 that defines *bitcoin:* URL format with implementation of the [Bitcoin Wallet Core Specification](core_spec.md). Using this scheme, web pages and natives apps may access bitcoin wallets that handle *bitcoin:* URL scheme.

Pay-to-Transaction URL
----------------------

    bitcoin:[address]?tx=<base58check encoding of a transaction>&amount=<btc>[&req-confs=N][&req-anyonecanpay=1][&message=...]
    
Wallet asks user's permission to spend certain amount in that transaction. Then it adds necessary inputs and outputs, signs transaction and broadcasts it.

If *req-anyonecanpay* is equal to 1, then all inputs must be signed with hashtype flag *ALL | ANYONECANPAY*. Otherwise the hashtype is *ALL*.

If *req-confs* is specified, every unspent outputs must have that many confirmations. Default is 0. If wallet does not have mature enough outputs, it should return generic failure.

Note: if the app supports funding via its own temporary wallet, it may also provide payment address before "?" which will be used by wallets not supporting the API.

Note: use *message* parameter to explain the reason for request to the user.


Inputs Authorization
--------------------

Step 1: authorize inputs.

    bitcoin:[address]?authinputs=<url>&amount=<btc>[&message=...]

Wallet asks user's permission to authorize a given amount. Then it prepares unsigned inputs and change outputs and sends them via the given URL.

App may also provide a bitcoin address as a fallback for wallets that do not support this API.

Step 2: sign a transaction.

    bitcoin:?tx=<base58check encoding of a transaction>&req-token=<base58check encoding of an authorization token>[&req-confs=N][&req-anyonecanpay=1][&req-return=<url where to post the signed transaction>]

Wallet checks if the token is valid and all its inputs and outputs are present in a given transaction. If correct, it inserts the signatures and broadcasts the transaction.

If *req-anyonecanpay* is equal to 1, then all inputs must be signed with hashtype flag *ALL | ANYONECANPAY*. Otherwise the hashtype is *ALL*.

If *req-confs* is specified, every unspent outputs must have that many confirmations. Default is 0. If wallet does not have mature enough outputs, it should return generic failure.

If *req-return* is present, instead of broadcasting, wallet calls the given URL with the base64url-encoded signatures for the transaction.

Note: use *message* parameter to explain the reason for request to the user.

TODO: we need to define a way for app to explain a reason for payment.


Public Key Request API
----------------------

    bitcoin:?&req-pubkey-index=<key index>&id=<base58check compressed pubkey>&t=<timestamp>&return=<url where to return the pubkey>[&message=...]&sig=<signature>


Signature API
-------------

Sometimes in case of web API or native extension API receiver may authenticate the caller using system-provided identifier. E.g. domain name (www.example.com) in JS API or bundle identifier (com.example.myapp) in iOS extensions. 

In cases when secure identification is not possible, App may identify itself simply by a public key and prove itself using a one-time signature of bitcoin: request.

    bitcoin:?req-sign-hash=<hex hash to sign>&i=<key index>&id=<base58check compressed pubkey>&t=<timestamp>&return=<url where to return the signature>[&message=...]&sig=<signature>

*req-sign-hash*: hash to sign. Hash must be minimum 160 bits long and maximum 512 bits long. Only this parameter is marked "req-" to make sure incompatible wallets fail. Other parameters (i, id, t, sig) are required.

*i*: index of a key to be used that is derived from a root key.

*id*: pubkey identifying the app. It is used to derive the root key.

*t*: UNIX timestamp of the current time. Wallet should ignore too old URLs or warn users about them. Recommended to only allow URLs within last 60 seconds and ignore any older or future ones.

*return*: URL where to return the signature in Base58Check format.

Wallet produces a key by mixing the root key with a given pubkey (DH) after validating that the bitcoin: request is not too old (less than a minute since receiving the request) and is properly signed by a given public key.

Source key is supposed to be stored in a secure location.

In some contexts this API may be of a no value. E.g. if the secure storage of an identifying public key may also be used for storing actual keys that user is supposed to use within the App. But in server-client Apps where server keeps the key and client authorizes transactions, this API is useful.


Diffie-Hellman API
------------------

    bitcoin:?req-dh=<base58check pubkey to multiply>&i=<key index>&id=<base58check compressed pubkey>&t=<timestamp>&return=<url where to return the signature>[&message=...]&sig=<signature>

Instead of signing a hash, multiplies a given public key by a derived private key and returns the resulting public key.
