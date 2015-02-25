
Bitcoin Wallet API Specification (draft)
========================================

Oleg Andreev <oleganza@gmail.com>

Yurii Rashkovskii <yrashk@gmail.com>

Motivation
----------

Apps implementing custom schemes on top of Bitcoin require user to provide bitcoins and signatures in custom transactions. Unified API enables users to keep their bitcoins in a convenient personal wallet while safely allocating desired amounts to custom transactions.

The goal is to avoid 3rd party apps from re-inventing secure key storage and backup. User wallet already keeps the keys, authorizes access to them and implements a backup strategy. Third party app can simply ask the user to sign certain transactions and add their bitcoins in them. The specification is designed to cover many use cases with extremely easy to use APIs.

Setup
-----

**App** is an application that implements some sort of custom transactions. App could be a web page or a native application.

**Wallet** is an application or device that is trusted by user and stores his private keys. Wallet could be web-based (accessible via HTTP), a native app, a hardware device or a browser extension.

Various combinations are possible:

1. App is a web page and Wallet is another web service (e.g. Blockchain.info, Coinbase).
2. App is a web page and the Wallet is a native application. 
3. App is a native application and the Wallet is a web service.
4. Both App and Wallet are native applications.
5. Wallet is a hardware device accompanied with a proxy client or browser extension. 

To allow smooth integration between apps and wallets in all these scenarios, we propose a single core specification and several platform-specific APIs.


Family of APIs
--------------

1\. **Core Specification** covers common binary data structures and secure protocol of exchanging them. It does not specify any specific programming interface for any platform or language. This protocol is the basis for all concrete APIs listed below.

2\. **[Bitcoin Wallet HTTP API](http_api_spec.md)** is an implementation of Core Spec for HTTP services. This is for web wallets like Coinbase and Blockchain.info.

3\. **[Bitcoin URL Protocol](url_spec.md)** is an implementation of Core Spec using *bitcoin:* URL scheme. It allows simple access to wallets for websites and native apps. Results are returned via HTTP URL callbacks.

4\. **[Bitcoin Wallet JavaScript API](js_api_spec.md)** is an implementation of Core Spec with JavaScript API for web browsers. It allows web pages accessing Bitcoin wallets directly from JavaScript without need to handle HTTP callbacks like when using HTTP API or bitcoin: URL.

5\. **[Bitcoin Wallet iOS/OSX API](ios_osx_api_spec.md)** is an implementation of Core Spec with system-provided extensions API. Wallet apps may expose such extensions to allow other native apps to access them.

6\. **[Bitcoin Wallet Android API](android_api_spec.md)** defines a standard way to declare extensions for Android apps.

7\. **[Bitcoin Wallet Windows API](windows_api_spec.md)** defines a standard way to declare extensions for Windows 8 apps.


#### I develop a smart contract scheme, what API should I use?

If you develop a **web application**, you should consider three APIs: [Bitcoin JS API](js_api_spec.md) (when available), [Bitcoin HTTP API](http_api_spec.md) and [Bitcoin URL](url_spec.md). Bitcoin JS API provides smoother experience for user, but requires coordination between their web browser and their wallet. Bitcoin HTTP API is also smooth and works similarly to OAuth: user has to authenticate transaction on a separate web page (but it works only with web-based wallets). Bitcoin URL may be used directly without special support from the browser and is used for communicating with native wallet apps. Bitcoin URL API also allows easy fallback to an intermediate wallet that can be used to fund special transactions (in case user's wallet does not support any form of Wallet API).

If you develop a **native application**, e.g. for iOS, consider using native [Bitcoin extension API](ios_osx_api_spec.md), [Bitcoin HTTP API](http_api_spec.md) or [Bitcoin URL](js_api_spec.md). Same reasoning applies as to web apps. When native extensions are not available, use Bitcoin HTTP for web wallets and URL scheme for native apps. You can use custom URL schemes in URL callbacks for inter-app communication.


#### I am a wallet developer, which API should I implement?

If your wallet is **web-based**, consider implementing [Bitcoin HTTP API](http_api_spec.md). Also consider developing a *satellite native app* to ease integration with native apps. 

If your wallet is a **native app**, it is recommended that you implement these three APIs: [Bitcoin URL](url_spec.md), native extension for your platform and integrate with browsers of your platforms so they can provide [JS API](js_api_spec.md). If the browser on your platform already knows how to speak with native extensions, you do not need to specifically integrate with JS API.

If your wallet is a **separate device**, consider implementing both [JS API](js_api_spec.md) via browser extension and a native app with a native extension. If the browser on your supported platform already supports JS API by communicating with native extensions, you may simply develop an app with native extension API.


#### I am a developer of a web browser, which API should I implement?

If you develop a web browser, consider these possibilities: 

1\. Implement [JavaScript API](js_api_spec.md) that talks to extensions provided by native wallet apps.

2\. Allow your own in-browser extensions to connect to JS API (so instead of overriding the entire JS API they can act as proxies to it). This is for wallets that are implemented as in-browser extensions or for in-browser extensions that talk to external hardware or software wallets.


Core Protocols
--------------

Bitcoin Wallet API is based on five protocols:

1. **Pay-to-Transaction Protocol**. App sends an incomplete transaction to Wallet and asks to add a certain amount of coins in it. Wallet asks user's permission and adds signed inputs to the transaction.

2. **Inputs Authorization Protocol**. This is a two-step process. App requests from Wallet an authorization to spend certain amount of coins. Wallet asks user's permission and sends back to the app a list of unsigned inputs and change outputs. App uses them to compose a complete transaction that is sent back to Wallet for signing. Wallet signs the transaction if it correctly uses all authorized inputs and outputs.

3. **Public Key Protocol**. App may outsource key storage to Wallet because it already implements security and safety measures such as encryption, authorization and backups. App can only access public keys specific to itself and request signatures of arbitrary data using these keys. Wallet never stores its own coins using these keys and only provides storage service to the app. 

4. **Signature Protocol**. Using the public key exposed in the previous API, App may request arbitrary data to be signed by the given key.

5. **Diffie-Hellman Protocol**. Using the public key exposed in the previous API, App may ask Wallet to multiply an arbitrary public key by a private key allocated for this App.

Wallet developers may implement all of these or only a subset within one of the concrete APIs for their platform.


### 1. Pay-to-Transaction Protocol

One request: app sends an incomplete transaction and asks wallet to add inputs and outputs and sign inputs. Wallet returns inputs, outputs and signatures.

Signatures may be done with hashtype ALL, SINGLE and ANYONECANPAY flags. Hashtype NONE is not supported until we find a good use case for them and figure how to deal with change outputs.

TODO: how to specify the fee. Should it be explicit, or only per inputs? Wallet may have many inputs and must pay for them. Also it should not be obliged to figure the input amounts to see how many fees are left. 

It's easier if the app mandates the fees (user will see authorized amount as amount + fee) or leaves it up to wallet that provides the fee only in proportion to size of its own inputs / outputs (in such case wallet may or may not show the fee as a spendable amount, just like it does with regular payments).


### 2. Inputs Authorization Protocol

Two requests: 

1. Send desired amount and a message, receive inputs and outputs. 
2. Send complete transaction. Wallet signs the inputs if all inputs and outputs are in place and returns signatures.

App is free to supply incomplete transaction and ask wallet to sign with ANYONECANPAY or SINGLE hashtype flags.

Signatures may be done with hashtype ALL, SINGLE and ANYONECANPAY flags. Hashtype NONE is not supported until we find a good use case for them and figure how to deal with change outputs.

TODO: specify the timeout for authorization. We need the wallet to "lock" inputs until they are actually signed. We have several options:

1) Lock forever. So even if they are not signed, they are considered "spent" right away. App is supposed to spend or refund them. Problem: sometimes app may want to simply cancel/abort the protocol.

2) Lock for a very long time (e.g. 1 year). So coins are effectively "locked" but can be used later. This is UX nightmare.

3) Lock for a time that wallet decides. App should spend inputs within a given reasonable timeframe or coins can be spent by the user. The problem is to explain to the user in simple terms that in case of cancellation, funds will become available only after X minutes/days.

4) Lock for a time that app decides (with wallet's cap). App may request inputs for shorter amount than default as a courtesy to the user. E.g. for a minute instead of one hour.

5) Lock for a time with explicit cancellation. App may try to tell the wallet to cancel the lock before timeout. The problem is how to communicate the cancellation. In some contexts (e.g. bitcoin: URL) it may be too explicit.

Other questions: is it desireable to allow very long timeouts, e.g. days? Maybe some contracts really require that.

6) Best-effort locking. Wallet does not promise anything to the app regarding locking inputs, but tries not to use them if possible. This requires no cancellation. If app requires temporary storage for funds (e.g. for long-term crowdfunding transaction), it should move the funds right away (e.g. to 2-of-2 multisig with user's key). This is in some sense a variation of option 3/4.




TODO: think about explicit revocation of authorization so app may retry.

TODO: require certain number of confirmations on an output from wallet. Rationale: while technically, app can check confirmations of the inputs itself, it will do that *after* getting user's permission to spend funds. If it wants to retry, it would be very confusing for the user to authorize again the same payment. Also, retrying is not a guarantee to receive confirmed unspent outputs again. To simplify things, wallet takes care of it if it can. If app does not care, it specifies number of confirmations equal to 0.


### 3. Public Key Protocol

Public key is linked to identity of the requestor. Different APIs do it differently. 

Public key is indexed by BIP32 non-hardened derivation index. This is to simplify key management for the wallets so they do not need to store any app-specific data at all.

Public keys must be presented in compact format (32 bytes).


### 4. Signature Protocol

Standard ECDSA signature and CompactSignature. 

Require signature to be canonical (lower S, DER encoding).

Maybe require it to be deterministic according to RFC6979?


### 5. Diffie-Hellman Protocol

Wallet should check if the given public key is correct and multiply it with app's private key. Returns a compact public key as a result of multiplication.


Proxying
--------

If the implementation of this spec acts as a bridge between the app and the wallet (e.g. a web browser or a native client to a hardware wallet), then it may proxy requests directly to the recipient without asking user's permission too often. 

#### JS API proxying by web browser

Browser may ask user to select a wallet of his choice or detect such wallet automatically (using bitcoin: scheme for native apps). 

Browser may ask user to authorize certain website's access to the wallet and remember that choice. This is to limit spam attempts to connect to the wallets and possibly exploit vulnerabilities in them. Once allowed, browser will never ask user's permission and direct all request to the wallet. It's now wallet's job to get user's permission to give app access to the wallet.

#### Native app proxy to hardware wallet

Similarly to web browser, native app may ask a permission to access hardware wallet just once. Then authorization happens on hardware wallet directly while native app simply transfers data to and from the requesting app.

Combination of these APIs enables a sophisticated chain of communication (web page with a hardware wallet) without any extra effort on anyone's part:

    Web page <-> JS API in browser <-> native API of a wallet client app <-> custom API of a hardware wallet

In this chain, web page accesses generic JS API. Web browser implements generic native extensions API (platform-dependent). Native app implements its own custom protocol to talk to its hardware wallet that implements Core Spec and authorizes transactions.


Error Handling
--------------

Error handling highly depends on actual implementation of this spec. But the general rule is: *Wallet should never leak private information* by offering a variety of error messages. In case of any error, App should receive a generic "failure" response while Wallet is free to tell the user actual reasons for error. 

Kinds of errors:

1. Wallet is not available. App receives "not available" error. App may provide user with alternative options to provide Bitcoins. Example: if [JS API](js_api_spec.md) cannot connect to any wallet, App may provide a [bitcoin: URL](url_spec.md) so user can go directly to his wallet app.

2. Not enough funds. Wallet may explain to user that App asks for more funds than user has. App sees generic "failure" error.

3. Authorization failed. Wallet may explain to user that his password is incorrect, fingerprint did not match, daily spending limit reached etc. App sees generic "failure" error.

4. No confirmed outputs available yet. Wallet may tell the user that his coins are not "mature" enough. App sees generic "failure" error.


Examples
--------

Here are some examples of interesting schemes that can use wallet API to allow user direct participation without intermediate deposits.

#### 1. Two party escrow

App (first party) prepares a transaction with 1 btc on the input and 2 btc on the 2-of-2 multisignature output. It then uses "pay-to-transaction" API to ask wallet to add another 1 btc input to the transaction. Wallet provides one or more inputs with total amount 1.2341 btc and one change output with 0.234 btc (0.0001 btc waived as a mining fee). Wallet appends its inputs and change output to the transaction and signs the inputs. It then returns signed inputs and output to the app (or broadcasts the transaction right away if API says so).


#### 2. Multiparty escrow

App is an intermediary between multiple people willing to enter in a multisig contract (for simplicity, lets say 2-of-3 scheme). Alice and Bob are supposed to contribute 1 btc each. Third party Carl acts as an arbitrator. App prepares a transaction with two outputs: one multisig with 1.99 btc and one output with Carl's address and 0.01 btc as his fee. App then uses 2-step Inputs Authorization API to gather inputs and change outputs for a total amount of 2 btc. Wallets of both parties ask permission of their owners and prepare some inputs and change outputs as in previous example. Inputs are not signed yet. Each wallet attaches a unique authorization token to the inputs and output. App receives inputs and outputs from each wallet and completes the transaction. Then it sends the complete transaction to each wallet to sign their inputs. Each wallet checks that the inputs/outputs were recently authorized and all of them are correctly present in the transaction. Each wallet signs its inputs and returns signatures back to the app. App waits for all wallets to provide signatures, adds them in a pending transaction and releases it.

In both examples App may use Public Key API and Signature API to get public keys for multisignature script and when redeeming it ask the Wallet to sign a target transaction. Since these public keys are outside the scope of Wallet's funds, Wallet allows signing arbitrary hashes asked by the App. The app thus completely controls signing process, but does not need to store sensitive private keys anywhere and worry about backup. Wallet provides secure and safe storage for keys, but does not concern itself with how they are used. It only guarantees the app that the keys are isolated from the user's funds and other applications.






