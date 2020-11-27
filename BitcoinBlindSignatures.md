# Schnorr Blinded Custody

**Note: the previous version of this paper proposes an unsafe protocol for ECDSA from 2014. You can find it here: [BitcoinBlindSignatures2014.md](BitcoinBlindSignatures2014.md).**

Author: Oleg Andreev <oleganza@gmail.com>

## Goal

Protect personal wallet by outsourcing signing to multiple trusted parties, while hiding the public key, signature and the message from them.

Most blind signature protocols focus on making signatures unidentifiable for a given public key, but do not protect public key itself. This is a problem for cryptocurrency transactions where the public key is either unique (one-time keys in Bitcoin) or identifies user's account (Ethereum, Stellar).

The following protocol uses each party as a "trusted custodian", while keeping them unaware about the published transaction, where both the signature and the public key are not linked to the signer's data. The only job of a custodian is to correctly authenticate the user and deny signatures to imposters.

## Compatibility

The scheme produces signatures compatible with Ed25519 (used in Stellar and some other protocols) and BIP340 (Schnorr signatures proposal for Bitcoin).

## Definitions

**Principle** is the user holding funds who outsources control to multiple **agents** (with m-of-n multisig).

**Agent** is a user providing signing service to the **principle**.

Together, **principle** and **agent** perform the role of a **signer** in the Schnorr signature protocol

## Protocol 1: key generation

Agent generates a key pair `x, X = x·G` and sends `X` to Principle. `G` is a common base point.

Principle chooses a blinding factor `b` as a uniformly random scalar.

Principle computes the public verification key `X' = X + b·G` and uses it for locking funds on a blockchain.

## Protocol 2: signing

Principle authenticates itself to its Agent to initiate a signature protocol.

Agent chooses a random nonce scalar `r` and sends commitment `R = r·G` to the principle.

Principle chooses two random blinding factors `p` and `q` and generates the blinded nonce commitment `R' = p·R + q`.

Principle sends blinded nonce commitment to the Verifier who responds with the challenge `c'` (in a non-interactive scheme, it is a hash of the protocol transcript).

Principle reverse-blinds challenge `c'` with previously generated blinding factor `p` to produce Agent's challenge `c = p^-1·c'`.

Principle sends reverse-blinded challenge `c` to Agent.

Agent computes the Schnorr signature `s = r + c·x` and returns `s` to Principle.

Principle blinds `s` as follows: `s' = p·s + q + c'·b`. 

Now the signature `(s',R')` is a valid Schnorr signature for challenge `c'` and verification key `X'`:

```
s' = p·s + q + c'·b
   = p·(r + c·x) + q + c'·b
   = p·r + q + p·c·x + c'·b
   = p·r + q + p·p^-1·c'·x + c'·b
   = p·r + q + c'·x + c'·b
   = p·r + q + c'·(x + b)
   = r' + c'·x'
```

## Practical considerations

The blinding factor `b` can be generated through a scheme similar to BIP32. The agent generates a single public key `X`, while the principle generates unique `b` for each "address" they need to send funds to.

Blinding factors `p` and `q` could be generated through the similar procedure as agent's nonce `r`: a mix of deterministic (based on transcript and blinding factor `b`) and auxiliary randomness.


## Compatibility with MuSig and threshold signing

TBD: present how MuSig must be adjusted to permit reconstruction of the blinded multikey, where the Principle acts like a Dealer, coordinating the protocol between multiple Agents.


## What blinding factors do we need?

Suppose we blind `X` with two factors `a, b`. Which factors could we omit, and would it be safe to do so?

```
x' = a·x + b
c' = a^-1·p·c
R' = p·R + q·G
s' = p·s + q + c'·b
   = p·(r + c·x) + q + c'·b
   = p·r + q + p·c·x + c'·b
   = p·r + q + p·a·p^-1·c'·x + c'·b
   = p·r + q + a·c'·x + c'·b
   = p·r + q + c'·(a·x + b)
   = r' + c'·x'
```

#### b = 0

```
x' = a·x
c = a·p^-1·c'
s' = p·s + q
   = p·(r + c·x) + q
   = p·r + q + p·c·x
   = p·r + q + p·a·p^-1·c'·x
   = p·r + q + a·c'·x
   = p·r + q + c'·(a·x)
   = r' + c'·x'
```

Seems to be ok, but not compatible with BIP32 derivation.


#### b = 0 & q = 0

```
x' = a·x
c = a·p^-1·c'
s' = p·s
   = p·(r + c·x)
   = p·r + p·c·x
   = p·r + p·a·p^-1·c'·x
   = p·r + a·c'·x
   = p·r + c'·(a·x)
   = r' + c'·x'
```

Insecure: can learn `a/p = c/c'`, then mult it by `s'` and get `a·s`, where `a = (c/c')*s'/s`.
If `a` matches, tx is deanonymized.


#### b = 0 & p = 1

```
x' = a·x
c = a·c'
s' = s + q
   = (r + c·x) + q
   = r + q + c·x
   = r + q + a·c'·x
   = r + q + c'·(a·x)
   = r' + c'·x'
```

Insecure: can compute `a` out of `c/c'` and match against X'.


#### a = 1

```
x' = x + b
c  = p^-1·c'
R' = p·R + q·G
s' = p·s + q + c'·b
   = p·(r + c·x) + q + c'·b
   = p·r + q + p·c·x + c'·b
   = p·r + q + p·p^-1·c'·x + c'·b
   = p·r + q + c'·x + c'·b
   = p·r + q + c'·(x + b)
   = r' + c'·x'
```

Can compute candidate `p` as `c'/c`, but `R'` is still protected by `q·G`, where `q` is protected by ECDLP.

Seems to be ok. Nice thing is - additive blinding is compatible with BIP32.

#### a = 1 & p = 1

In this case `c == c'` which identifies the transaction.

#### a = 1 & q = 0

Tx is identified by matching `(c'/c)R` with `R'`.
