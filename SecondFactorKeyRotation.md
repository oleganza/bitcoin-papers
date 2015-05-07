Key rotation scheme for second-factor cryptographic keys (DRAFT)
================================================================

Oleg Andreev <oleganza@gmail.com>

Introduction
------------

Here we propose a scheme for additional protection of Bitcoin wallet's backup taking into account possibility to add/remove authentication methods and rotate encryption keys in case 2FA service is compromised.

Overview
--------

To protect user's funds against data loss we create a copy of the master key (see [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), encrypt it and upload to some cloud service (send by email, send to Dropbox, iCloud Drive etc.) The key can be decrypted only when user authenticates with one of the methods devised upfront (email, phone call, sms code, TOTP code).

User generates their own symmetric encryption key *per factor* to encrypt their master key. Then the symmetric key is encrypted with *Authenticator's* service asymmetric key. If multiple authentication methods are used (e.g. email and phone number), then multiple encrypted copies of master key and corresponding number of encrypted encryption keys are stored in the cloud storage service.

Whenever *Authenticator* is compromised, all copies of master key encrypted with compromised keys must be destroyed and re-encrypted with new keys. However it would be more robust to automate that process and perform *key rotation* on a regular basis. Organization may rotate its keys when it changes the staff of admins, location or configuration of the servers. To automate this process, we propose four techniques:

1. *Authenticator Key* is signed by multiple long-living *Rotation Keys* (threshold multi-signature, e.g. 2-of-3). The *Rotation Keys* can be pinned on first use or built in the wallet app (recommended). All subsequent *Authenticator Keys* must be signed by the same *Rotation Keys* (unless they themselves are updated).
2. A rotation mechanism for long-living keys themselves. The same or stricter multi-signature threshold may be used to introduce a new set of long-living keys. This is useful to allow soft migration whenever one of the long-living keys is compromised.
3. Wrapping the existing encrypted master key using the new *Authenticator Key*. This is to enable wallet app to update backup automatically without requiring user to unlock the master key (which may be protected with interactive authentication like TouchID or device passcode). To avoid needless accumulation of wrappers, wallet performs clean encryption of the original master key whenever it is unlocked by the user.
4. Silent push notifications to let wallet app update encrypted backup automatically as soon as possible.


Definitions
----------- 

**Master Key** is wallet's primary secret (e.g. BIP32 seed or BIP39 mnemonic phrase). During normal operation it is directly accessible on the user's device and protected by local passcode or TouchID.

**Authenticator** is a service providing authentication capability via decryption based on [ECIES](http://en.wikipedia.org/wiki/Integrated_Encryption_Scheme).

**Authentication Factor** is one of the methods of authentication provided by the *Authenticator*: e.g. confirmation code by phone call, SMS or email.

**Authentication Key (Apub, Aprv)** is an elliptic curve key (private or public) used for ECIES encryption and decryption of the **Encryption Key**.

**Encryption Key** is a unique unpredictable AES-128 key used to encrypt a copy of *Master Key* for each *Authentication Factor*. It is recommended to generate it using `HMAC-SHA256(key: Timestamp, data: MasterKey)` to limit dependency on random number generators and enable blackbox aduitability of the wallet.

**Encrypted Master Key (EMK)** is a AES-128-encrypted *Master Key*. For uncompressible binary keys it is sufficient to use AES-128-ECB. For human-readable passphrases AES-128-GCM (or CBC) must be used to avoid leaking the penguin.

**Encrypted Encryption Key (EEK)** is ECIES-encrypted *Encryption Key*.

**Master Key Envelope** is an object describing *Authenticator*, *Factor*, *Authentication Key Envelope* and either *Encrypted Master Key* or *Encrypted Master Key Envelope*.

In case of encrypting original *Master Key*:

```
{
  "service": Authenticator,
  "factor":  "phone" | "sms" | "email" | "totp" | etc,
  "account": PhoneNumber | Email | TOTP ID | etc,
  "ake":     AuthenticationKeyEnvelope,
  "eek":     EEK,
  "emk":     EMK
}
```

In case of wrapping an existing envelope:

```
{
  "service": Authenticator,
  "factor":  "phone" | "sms" | "email" | "totp" | etc,
  "account": PhoneNumber | Email | TOTP ID | etc,
  "ake":     AuthenticationKeyEnvelope,
  "eek":     EEK,
  "emke":    EMKE
}
```

**Encrypted Master Key Envelope (EMKE)** is a AES-128-GCM-encrypted *Master Key Envelope* which is being wrapped by an outer *Master Key Envelope* (see above).

**Authentication Key Envelope (AKE)** is an object containing *Authentication Public Key (Apub)*, *Rotation Keys Envelope* and corresponding signatures of these two objects.

```
{
  "apub": Apub,
  "rke": RKE,
  "sigs": [
    ECDSA-DER(RotationKey1, Apub || RKE),
    ECDSA-DER(RotationKey2, Apub || RKE),
    ...
  ]
}
```

**Rotation Keys** is a collection of EC keys belonging to *Authenticator* administrators. These keys are used to create threshold multi-signatures to allow wallets verify *Authentication Keys* (and *Rotation Keys* themselves) during key rotation.

**Rotation Keys Envelope (RKE)** is an object containing *Rotation Keys*, threshold for signing *Authentication Key Envelope*, threshold for signing *Rotation Keys Envelope* itself, hash of the previous envelope and signatures for this envelope according to previous envelope's threshold and keys. 

```
{
  "keys": [
    Rpub1, 
    Rpub2, 
    Rpub3, 
    ...
  ],
  "ak-threshold": N, // minimum number of Rotation signatures to update AKE.
  "rk-threshold": M, // minimum number of Rotation signatures to update RKE. If missing, rotation is not allowed.
  "prev": SHA-256(SHA-256(PreviousRKE)),
  "sigs": [
    ECDSA-DER(RotationKey1, keys || ak-threshold || rk-threshold || prev),
    ECDSA-DER(RotationKey2, keys || ak-threshold || rk-threshold || prev),
    ...
  ]
}
```

Usage Scenarios
---------------

### Adding a factor

When user adds a security factor (say, a phone number), he receives an *Authentication Key Envelope* after verifying that the chosen factor actually works. Using the received *Apub* wallet can produce *Master Key Envelope* which is uploaded to a cloud storage service (e.g. iCloud/Dropbox/etc). 

### Restoring from backup

When restoring wallet on another device, one of the *factors* must be used to decrypt *Master Key*. User chooses one of the available factors, wallet makes a necessary requests to *Authenticator* ultimately sending an *Encrypted Encryption Key* and a corresponding *Apub*. If authentication succeeds, *Authenticator* uses private key corresponding to a given *Apub* to decrypt the *Encryption Key*. The decrypted *Encryption Key* is returned to the wallet app. Wallet app is now able to decrypt its *Master Key* and must re-encrypt it with the new *Encryption Key* since the previous one is now compromised.

###  Removing a factor

To remove a factor, wallet simply destroys a copy of *Master Key Envelope* corresponding to that factor.

### Changing a factor

To change parameters of a factor (e.g. change the phone number), wallet adds new *Master Key Envelope* for a new value and then removes the previous envelope.


Attack Scenarios
----------------

### User's device is stolen

Funds are protected by on-device encryption and passcode/TouchID authentication.
User can restore the wallet on another device using his cloud account and one of the authentication factors.

### User's cloud account is compromised

Funds are protected by Authenticator's key. User should migrate funds to a new wallet.

### Authenticator is compromised

Funds are protected by user's cloud account access policy and eventually 2-factor protection is restored when keys are rotated.


Data Loss Scenarios
-------------------

### User's device is lost

Funds are protected by on-device encryption and passcode/TouchID authentication.
User can restore the wallet on another device using his cloud account and one of the authentication factors.

### User's cloud account is not available

Funds are still available on the current device. Wallet must be backed up in another location ASAP.

### Authenticator is not available

Funds are still available on the current device. User may use another authenticator or add one ASAP.


Rationale
---------

**Why 128-bit encryption?**

128-bit AES key is chosen since AES actually has a lousy 256-bit key scheduler which is surprisingly [weaker than the 128-bit one](https://www.schneier.com/blog/archives/2009/07/another_new_aes.html).


Reference Implementations
-------------------------

TBD.


Test Vectors
------------

TBD.







