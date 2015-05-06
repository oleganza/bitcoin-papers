
Satellite Vault
===============

Overview
--------

Bitcoin storage solutions reduce financial privacy with 3rd party custody and require round-the-clock protection of the trusted computers.

We propose a radically new approach.

Trusted computers deployed on a fleet of satellites. You transact directly with the satellites and retain 100% financial privacy. Withdrawals are delayed and can be canceled immediately. Physical access is impossible, eliminating risk of compromise or destruction of trusted computers.


### Problem

Storing large amounts of bitcoins over long term is a tough problem. Existing solutions use two strategies (sometimes simultaneously): multi-signature transactions (or splitting the secret keys) and secure locations (deposit boxes to store USB drives, high-security data centers).

Two problems exist, however:

Hardware remains always accessible to staff, law enforcement and physical attacks. Employee defection or a single search warrant immediately put at risk every customer and destroy the entire company's reputation.

Customers' privacy is reduced by authentication with 3rd parties. To enforce withdrawal policy a lot of identifying information is gathered that can be linked to actual transactions.

### Solution

Since we need to carefully set up the hardware and initialize it with cryptographic material, lets make that once, and deploy on the satellite to permanently close the window of risk.

Bitcoin withdrawals are time-locked and delayed according to a policy specified by the customer.

When a withdrawal is requested, customer receives a notification and may cancel the withdrawal within 24 hours or more, depending on the amount.

### Security

We not only care about security, but also about convenience. All operations are managed with a simple app on your phone. Communication with the satellite is routed through Space Vault Operation Center.

Every Bitcoin transaction, policy update and even billing are end-to-end encrypted and authenticated between the customer and the satellite. No one can possibly learn the financial details of the customer without their explicit consent. In effect, no one except the customer can initiate transactions.

When withdrawal is requested, customer receives a notification on multiple addresses (SMS, email etc.) Every notification allows immediate cancellation of the transaction. Depending on the amount and the policy, customer has 24 hours or more to cancel the transaction before it is committed to blockchain.

To authenticate the convenient means of notifications (such as email) customers use personal ID. However, ID is never linked to transactions. In other words, “we know that you are our customer, but we never know what you store in your deposit box“.

Completely anonymous operation is also possible (with less convenient notification methods). And for complete autonomy you can order a fully configured suitcase-size radio station.

### Hardware

In the initial phase we will use a constellation of the three 3U CubeSat satellites for optimal redundancy deployed by our partners.

Each satellite is independently configured but enforces the same withdrawal policy. 1-of-3 and 2-of-3 multi-signature setups available.

Secure code and cryptographic identity of the Space Vault are stored in a dedicated write-once Hardware Security Module onboard. BitSat operating system provides the Vault with raw blockchain data that simultaneously acts as a trusted clock to enforce withdrawal delays.

Communication with the satellite is performed via open protocol over CDMA radio link. Your bitcoins remain accessible from anywhere in the world even if Operation Center evaporates. We also offer a portable suitcase-sized radio station for complete peace of mind.


Specifications
--------------

### End-to-end encryption

Full financial privacy is maintained by all communications between the satellite and the user being end-to-end encrypted. Operation Center that handles notifications and general routing does not know the contents of the transactions or even policies.

### Clients store data, not the satellite

Withdrawal policy file (e.g. "1 btc per day") is signed by the satellite and associated with its own oracle signing key. Satellite encrypts the signing key and sends back to client. Each time the client makes a withdrawal request, it must send back the policy file. At the minor cost of addition bandwidth this allows to scale satellite to great number of customers regardless of the amount they store (since satellite does not store any per-client data).

### Notifications double-encrypted for client and Operation Center

When the client is compromised, attacker can issue time-delayed withdrawals, but it may also receive notifications with cancellation data. Thus client may be potentially locked out of access to funds and blackmailed. To avoid that, notifications are encrypted to the Operation Center's keys that provides "out-of-band" authentication to let customers change the communication channel. So if the client's phone number or email is hijacked, client may authenticate with Operation Center and change the contact info in order to cut off attacker's cancellation requests.

### Self-destruction in case trajectory changes

Low-orbit satellites may be harvested by a sufficiently sophisticated and well-funded attacker. To prevent compromise of the secret data, TTS system detects drastic deviation from the planned trajectory and immediately destroys sensitive information (such as master key). It also attempts to inform the Operating Center and other satellites in the fleet about the event.

### Multi-signature key rotation scheme for Operating Center keys

Security of notifications relies on Operation Center's keys being held securely. In case of compromise, the keys must be rotated based on multi-party signature using long-living keys held by Operating Center's managers and board of directors. See [Key Rotation Scheme for 2FA wallet backups](TwoFactorWalletBackup.md) for a draft of key rotation spec.

Note that unlike on-earth signing oracles, compromise of Operating Center's keys coupled with compromise of some client's key does not allow immediate loss of funds, but instead allows attacker to perform denial-of-service attack until key rotation is performed. In the worst case, when meaningful number of long-living rotation keys is compromised (together with customer's key), customer may be blackmailed to enable their own withdrawals. Which means that customer is in a position to negotiate a "tax" on each withdrawal with the attackers (e.g. 50/50 split).






 
