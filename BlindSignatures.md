Blind Signatures
================

Author: Oleg Andreev <oleganza@gmail.com>

* [Reference Implementation in Objective-C](https://github.com/oleganza/CoreBitcoin/blob/master/CoreBitcoin/BTCBlindSignature.h)
* [Demo Application](https://github.com/oleganza/blindsignaturedemo)

Abstract
--------

Blind signatures allow maintaining privacy while using third-party services such as digital cash server. Existing proposals of blind signatures for ECC lack compatibility with standard ECDSA and thus cannot be used directly in Bitcoin transactions. We propose a scheme that allows generating a blind signature compatible with the existing Bitcoin protocol. The client requests a set of parameters from the custodian and synthesizes a public key to use in a Bitcoin transaction. To redeem the funds, the client transforms the hash of the transaction (“blinds”), sends to the custodian to sign and then transforms the signature (“unblinds”) to arrive at a valid ECDSA signature. The signed transaction is published revealing the synthetic public key and the unblinded signature. Custodian is limited to authenticating the client and his intent, but cannot learn anything about the transaction.

Introduction
------------

A blind signature scheme is a protocol allowing the recipient to obtain a valid signature for a message from the signer without them seeing the message. Blind signature scheme is a digital signature scheme which satisfies non-forgeability and unlinkability properties. Non-forgeability property means that only the signer should be able to generate valid signatures. Unlinkability property means no one can derive a link between a protocol view and a valid blind signature except the author of the message. [[1]](http://ijns.femto.com.tw/contents/ijns-v14-n6/ijns-2012-v14-n6-p316-319.pdf) The concept of blind signatures was introduced by David Chaum in 1982 [[2]](http://www.hit.bme.hu/~buttyan/courses/BMEVIHIM219/2009/Chaum.BlindSigForPayment.1982.PDF) and was extended by multiple authors to the Elliptic Curve Cryptography (ECC) [[3]](http://mshwang.ccs.asia.edu.tw/www/myjournal/P191.pdf) [[4]](http://arxiv.org/pdf/1304.2094.pdf). All works on blind signatures for ECC describe fairly simple methodology to blind messages and unblind signatures, but they all lack compatibility with existing standardized ECDSA scheme.

What is needed is a scheme that produces a compatible ECDSA signature which can be used in existing systems, particularly in Bitcoin protocol. Ability to make blind signatures directly for Bitcoin transactions would allow to separate the signing party (a “custodian”) from knowing anything about the funds they are protecting. This can be viewed as an ultimate solution for secure storage of bitcoins as no individual computer system can be fully trusted (e.g. even RNG may leak information about private keys through ECDSA signatures [[5]](http://www.wired.com/politics/security/commentary/securitymatters/2007/11/securitymatters_1115)). By spreading the trust between several independently operating computers the risk is significantly reduced. The attacker not only has to compromise several computers instead of just one, but beforehand they must find out which computers are involved (which is made much harder when blind signatures are being exchanged). Therefore the scheme enables safer Bitcoin storage on conventional personal computers and smartphones without need for specialized hardware.

Lets say Alice wants to protect her bitcoins against active attackers. All her personal computing devices may be confiscated or secretly compromised in order to access her secrets. Her “paper wallets” may be stolen or become inaccessible. Specialized “hardware wallets” can be badly compromised as well. To protect her funds, Alice may choose to lock them in a “3-of-5” multisignature transaction with 5 of her friends (possibly located in different jurisdictions, using different hardware and software). To unlock her funds, she will need any 3 of her friends to sign the redeeming transaction. Friends are instructed to authenticate Alice either in person, by phone or via other secure channels. As long as any 3 of her friends are available, did not lose their keys, and no one is able to coordinate an attack against the majority of her friends at the same time, her funds are much safer than locked with just her personal keys. The only problem: Alice reveals her funds to all her friends. This reduces security drastically as some friends may attempt to conspire against Alice if she posesses a lucrative amount of bitcoins. With the help of blind signatures, Alice may enjoy security provided by her friends, without revealing transactions she is signing. Assuming her friends are keeping their money safe in a similar way, all participants mutually help each other without revealing sensitive information.


Ordinary ECDSA signature
------------------------

Let **n** be an order of the elliptic curve.

Let **G** be a standard “generator” point on the curve.

Let **t** be a private key (integer in the interval [1, *n* – 1]).

Let **T** be a public key corresponding to *t* (by definition, *T = t·G*).

Let **h** be a cryptographic hash of a message to be signed (integer in the interval [1, *n* – 1]).

Let **k** be a unique random number chosen per signature (integer in the interval [1, *n* – 1]).

Then the ECDSA signature is defined as a pair **(Kx, s)**, where:

* **Kx** is x-coordinate of the point *k·G* on the elliptic curve modulo *n*.
* **s** = *k-1·(h + t·Kx) mod n*
* Signature is invalid if either *Kx* or *s* are zero.
* Note: *k-1* is an inverse of *k* modulo *n* such that *(k^-1·k) = 1 mod n*.
	
The verification requires 3 objects to be present: message hash **h**, public key **T** *(=t·G)* and a signature pair **(Kx, s)**. Verification steps are as follows:

1. Verify that *Kx* and *s* are integers in the interval [1, *n* – 1].
2. Compute *w = s-1 mod n*.
3. Compute *u1 = e·w mod n*.
4. Compute *u2 = Kx·w mod n*.
5. Compute *X = u1·G + u2·T*.
6. If *X = ∞* signature is invalid.
7. Signature is valid if x-coordinate of *X* modulo *n* is equal *Kx*.



Transformations
---------------
 
We observe that the core of the signature is a linear transformation of a parameter h with both factors unknown to the recipient: *s = a·h + b*. We will use this idea to *blind the message*, sign the blinded message and then *unblind the signature*. Note that Bob’s signature will not be standard ECDSA, but after unblinding it, Alice will arrive at a valid ECDSA signature.

Lets imagine Alice wants Bob to sign a message blindly. First, she needs to send him a transformed (“blinded”) hash of the message. Then, Bob transforms (“signs”) the blinded hash and returns the resulting number to Alice. Alice then transforms (“unblinds”) Bob’s number and arrives at a valid signature. This signature can be verified by some synthetic public key that Alice computed in advance from a combination of her secret parameters and Bob’s public parameters.

Let *a*, *b*, *c* and *d* be unique random numbers within [1, *n* – 1] chosen by Alice.

Let *p* and *q* be unique random numbers within [1, *n* – 1] chosen by Bob.

Alice computes the hash of her message *h* and then transforms it as follows:

* *h2 = a·h + b mod n*   (blinding)
	
She sends *h2* to Bob who performs another transformation:

* *s1 = p·h2 + q mod n*		(signing)
	
Bob sends *s1* back to Alice. She performs the final transformation:

* *s2 = c·s1 + d mod n* 		(unblinding)
	
The resulting number *s2* should be a part of the signature *(Kx, s2)* verifiable by a public key *T (= t·G)*. The only question: is it possible to determine *Kx* and *T* without compromising the secrecy of all chosen parameters? Also, *T* must be known in advance, before h is determined.

Lets expand *s2* as transformation of *h*:

* *s2 = c·(p·(a·h + b) + q) + d mod n*
* *s2 = c·p·a·h + c·p·b + c·q + d mod n* **(1)**

At the same time we want s2 to be the second part of the ECDSA signature as a function of h:
 
* *s2 = k-1·(h + t·Kx) = k-1·h + k-1·t·Kx mod n* **(2)**

where

* *k* — unique secret number within [1, *n* – 1]
* *Kx* — x-coordinate of the point *k·G* (this number is the first half of the signature)
* *t* — private key, number within [1, *n* – 1]

Alice needs to know *Kx* and *t·G*. By comparing (1) and (2) we can find the relation between ECDSA parameters with our chosen parameters *a*, *b*, *c*, *d*, *p*, *q*.

From (1) and (2) as equivalent linear transformations of an independent variable h follows:

* *k-1 = c·p·a mod n* **(3)** 
* *k-1·t·Kx = c·p·b + c·q + d mod n* **(4)**

From (3) we can find *k* and *K*:

* *k = (c·p·a)-1 mod n*	 					**(5)** 
* *K = (c·a)-1·p-1·G*   					**(6)**


It’s evident from (6) that Bob can communicate *p-1·G* to Alice without revealing *p*. So Alice can know *K* and *Kx* without knowing *k*.

From (4) we can find *t* and *T*:

* *t = Kx-1·(k·c·p·b + k·c·q + k·d) mod n*		

Expanding *k*:

* *t = Kx-1·((c·p·a)-1·c·p·b + (c·p·a)-1·c·q + (c·p·a)-1·d) mod n* 
* *t = (a·Kx)-1·(b + q·p-1 + d·c-1·p-1) mod n*

Multiplying by *G* to arrive at a target public key:

* *T = t·G = (a·Kx)-1·(b·G + (q·p-1·G) + d·c-1·(p-1·G))*   **(7)**

From (7) we see that Bob can communicate EC points *(p-1·G)* and *(q·p-1·G)* without compromising secrecy of *p* or *q*. That way Alice can determine public key *T* that will verify her signature in the future, without Bob ever knowing *T*.

Now we have everything to present the protocol of constructing a blind signature by Bob for Alice.


Core Protocol
-------------

1. Alice chooses random numbers *a*, *b*, *c*, *d* within [1, *n* – 1].

2. Bob chooses random numbers *p*, *q* within [1, *n* – 1] and sends two EC points to Alice: *P = (p-1·G)* and *Q = (q·p-1·G)*.

3. Alice computes *K = (c·a)-1·P* and public key *T = (a·Kx)-1·(b·G + Q + d·c-1·P)*. Bob cannot know if his parameters were involved in *K* or *T* without the knowledge of *a*, *b*, *c* and *d*. Thus, Alice can safely publish *T* (e.g. in a Bitcoin transaction that locks funds with *T*).

4. When time comes to sign a message (e.g. redeeming funds locked in a Bitcoin transaction), Alice computes the hash *h* of her message.

5. Alice blinds the hash and sends *h2 = a·h + b (mod n)* to Bob.

6. Bob verifies the identity of Alice via separate communications channel.

7. Bob signs the blinded hash and returns the signature to Alice: *s1 = p·h2 + q (mod n)*.

8. Alice unblinds the signature: *s2 = c·s1 + d (mod n)*.

9. Now Alice has *(Kx, s2)* which is a valid ECDSA signature of hash *h* verifiable by public key *T*. If she uses it in a Bitcoin transaction, she will be able to redeem her locked funds without Bob knowing which transaction he just helped to sign.


Security Analysis
-----------------
 
We assume Bob may see the Alice’s message (or its hash *h*), signature *(Kx, s2)* and the public key (*T*).

1. Bob does not know *a* and *b*, therefore he cannot decide if *h2* is a blinded version of *h*.

2. Bob does not know *c* and *d*, therefore he cannot decide if *s2* is unblinded version of *s1*.
 
3. Bob does not know *(c·a)*, and due to difficulty of ECDLP (Elliptic Curve Discrete Logarithm Problem) cannot find them from *K*, therefore he cannot know if *Kx* was produced from his EC point *P*.

4. Similarly, due to difficulty of ECDLP Bob cannot learn if *P* and *Q* were used in constructing public key *T*.

As a result, Alice has completely hidden the fact that Bob helped her sign her message, even if she publishes everything that may need to be published (e.g. in case of Bitcoin, transaction message, signature and the public key need to be public).

Alice must not reuse parameters *a* and *b* for different messages as this would allow Bob to link blinded hashes *h2* to the original hashes *h* by solving two linear equations to find *a* and *b*, and then matching publicly available messages with the blinded hashes he has already signed.

Alice must not reuse *c* and *d*, as Bob can find them from the published signatures in the same way as he can find reused a and b from public and blinded hashes.

Alice cannot learn Bob’s parameters *p* and *q* from *P* and *Q* (due to hardness of ECDLP), so she cannot forge his signature. However, if she makes Bob use the same parameters to sign a different message, she can calculate *p* and *q*. Bob may keep track of hashes he signed to never allow using the same pair (*p*, *q*) to sign different hash, but that’s not really need. The security of the scheme depends on Bob providing secure authentication separately from Alice’s secrets, so only Alice herself (not someone who stole all her secrets) can demand a valid signature from Bob. Since Bob keeps *p* and *q* only for Alice, it’s not important for him if Alice tries to figure them out.

This allows us to put more control over parameters in hands of Alice and less in Bob’s. She will be able to ask Bob for a specific set of parameters matching the message she is going to create. As a result, Bob does not need to keep track of which parameters were already used because Alice will take care of that. Bob only needs a simple mechanism to generate required parameters sequentially on demand.


Generating and exchanging parameters
------------------------------------

Alice will request parameters from Bob more than once to use in multiple messages. In order to simplify the implementation, we propose a standard way to generate secret parameters and exchange public parameters. This is not required for the scheme to work, but is very useful to be implemented in a standard way to have interoperable implementations.

To generate several random numbers we will use a key derivation scheme described in [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) (“Hierarchical Deterministic Wallets”). In that scheme, a sequence of keys is derived from a single extended private or public key using a simple incremented index. Each key pair is accompanied by extra 32 bytes of entropy called “chaincode” used to derive private and public keys. Since we need almost all features of BIP32 (extra entropy, sequential derivation, deriving from private and public keys, a serialization format), it would be unwise to invent similar, but incompatible scheme.

The operation is as follows.

1. Alice creates an extended private key *u*.

2. Bob creates an extended private key *w* and sends the corresponding extended public key *W* to Alice.

3. Alice assigns an index *i* for each “blind” public key *T* to use in the future. Alice must use each value of *i* only for one message.

4. Alice generates *a*, *b*, *c*, *d* using *HD(u, i)*, a “hardened” derivation according to BIP32:

* *a = HD(u, 4·i + 0)*

* *b = HD(u, 4·i + 1)*

* *c = HD(u, 4·i + 2)*

* *d = HD(u, 4·i + 3)*

Alice generates *P* and *Q* via *ND(W, i)*, normal (“non-hardened”) derivation according to BIP32:

* *P = ND(W, 2·i + 0)*

* *Q = ND(W, 2·i + 1)*

Using *a*, *b*, *c*, *d*, *P* and *Q*, Alice computes *T* and *K* and stores tuple *(T, K, i)* for the future, until she needs to request a signature from Bob.

When Alice needs to sign her message, she retrieves *K* and *i* and recovers her secret parameters *a*, *b*, *c*, *d* using *HD(u, i)*.

Alice sends Bob a blinded hash *h2 = a·h + b (mod n)* and index *i*.

Because *P* and *Q* are not simply the public keys of *p* and *q*, Bob needs to do extra operations to derive *p* and *q* from *w* and *i* (following BIP32 would not be enough):


From *P = p-1·G = (w + x)·G* (where *x* is a factor in *ND(W, 2·i + 0)*) follows:
		
* *p = (w + x)-1 mod n*
		
From *Q = q·p-1·G = (w + y)·G* (where *y* is a factor in *ND(W, 2·i + 1)*) follows:
	
* *q = (w + y)·(w + x)-1 mod n*
		
Factors *x* and *y* are produced according to BIP32 as first 32 bytes of HMAC-SHA512. See [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) for details.

Bob computes blinded signature *s1 = p·h2 + q (mod n)* and sends it to Alice (after verifying her identity).

Alice receives the blinded signature, unblinds it and arrives at a final signature *(Kx, s2)*.

Alice does not need to interact with Bob to generate public keys. She only need to receive an extended public key *W* once. To request a signature matching some public key, Alice needs to tell Bob the blinded hash and the index of that public key. Bob will reply with a valid blind signature.


Conclusion
----------

We presented a scheme that enables blind signatures compatible with ECDSA. The signing party can provide a service of storing private keys and signing messages, but never knowing which message they have signed. In the context of Bitcoin, it allows one party to privately lock and redeem funds (which requires publishing the public key and the signature) while the other party blindly authorizes the transaction.

The novelty of the scheme is that unlike the original Chaum blind signature scheme, this one does not allow anyone to prove that the signing party signed a particular message, but instead provides a much stronger privacy: the resulting signature, public key and the message are all completely unlinkable to the signing party. In the context of Bitcoin, the user does not need to redeem funds from a single bank (where Chaum’s blind signatures are used), but defines her own public key against which the redeeming transaction will be validated by the Bitcoin network. Essentially, her synthetic public key will act as her own private bank.


References
----------

[1] Kalyan Chakraborty and Jay Mehta, A Stamped Blind Signature Scheme based on Elliptic Curve Discrete Logarithm Problem (http://ijns.femto.com.tw/contents/ijns-v14-n6/ijns-2012-v14-n6-p316-319.pdf), 2011.

[2] David Chaum, Blind Signatures for Untraceable Payments (http://www.hit.bme.hu/~buttyan/courses/BMEVIHIM219/2009/Chaum.BlindSigForPayment.1982.PDF), 1982

[3] Chwei-Shyong Tsai, Min-Shiang Hwang, Pei-Chen Sung, Blind Signature Scheme Based on Elliptic Curve Cryptography (http://mshwang.ccs.asia.edu.tw/www/myjournal/P191.pdf)

[4] Hossein Hosseini, Behnam Bahrak, Farzad Hessar, A GOST-like Blind Signature Scheme Based on Elliptic Curve Discrete Logarithm Problem (http://arxiv.org/pdf/1304.2094.pdf)

[5] Bruce Schneier, Did NSA Put a Secret Backdoor in New Encryption Standard? (http://www.wired.com/politics/security/commentary/securitymatters/2007/11/securitymatters_1115)

[BIP32] Pieter Wuille, Hierarchical Deterministic Wallets (https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), 2013.



