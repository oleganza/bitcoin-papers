Decentralized Payment Network
=============================

Oleg Andreev <oleganza@gmail.com>


Introduction
------------

Bitcoin blockchain establishes a global consensus on distribution of property titles. However, the need for global synchronization puts a limit on number and speed of transactions. Blockchain is great as a platform for contracts and "digital gold", but probably will never be able to scale to billions of daily transactions. Luckily, most transactions are not expensive or important enough to require global acknowledgement. The most obvious solution is a *clearing house*: an organization that has accounts with multiple customers, stores actual bitcoins in custody and allows customers to exchange IOUs frequently and efficiently, while clearing debts only so often.

The problem with the centralized clearing house is the risk of immediate loss of funds due to compromise or data loss which puts a limit on how much can it scale. Complex hierarchies of clearing houses need to be built to connect all such institutes in a single global network.

We propose a *decentralized clearing protocol* that effectively allows anyone to be their own clearing house, and uses [Joint Escrow](JointEscrow.md) to establish mutual trust between debtor and creditor who can potentially be completely anonymous (e.g. bots). IOUs propagate from node to node and are trusted because of a Joint Escrow transaction between each pair of nodes (locking up certain amount on both ends in 2-of-2 multisig). Total amount of debt from one node to another is limited to 50% of the locked amount (e.g. if both nodes lock up $20 each, they allow debt up to $10 in each direction). When debt is reaching its limit, it's being "cleared" by debtor via a real Bitcoin transaction or simply by "closing" the contract transaction with correct proportion on outputs to pay off the debt.

Every node may require an arbitrary fee for a service of providing his funds to back IOUs, when making a payment, merchant/customer may find several possible "paths" and choose the quickest/cheapest one to use. Centralization is possible at a proportional capital expense. If some node wants to be Visa-scale with millions of contracts and a lot of fees to earn, they'll have to lock up huge amount of money. This puts natural limit on centralization or associated risk. 

### Example

I pay $10. The following path is discovered and signed off by the Merchant who accepts an ad-hoc 0.3% fee:

Me: $10 -> $9.99 (Alice) -> $9.98 (Bob) -> $9.97 (Merchant).

Now I owe $10 to Alice, Alice owes $9.98 to Bob, Bob owes $9.97 to Merchant. Clearing of debts happens independently between each participant based on their debt-to-capital ratio or whenever any party wishes to exit the contract. Of course, if several paths are discovered within a reasonable timeframe, Merchant will choose the cheapest one. Merchant may abort the transaction if the proposed path is too expensive (e.g. total fee is >1%).

### Pros

* Decentralized.
* Mere seconds to settle a payment.
* Infinite scalability (no global consensus). Each payment is routed through 5 to 10 nodes.
* No trusted parties or federation (trust is "purchased" using "joint escrow" txs on blockchain)
* No funny currency, IOUs denominated in BTC.
* No global consensus or protocol. Nodes can be semi-compatible, upgrade independently. All risks are local.

Political problems solved:

* No need to debate zeroconf transactions. We don't *need* them anymore to buy a coffee.
* No need to debate block size limit. It'd still be nice to raise it when needed, but for 99% of transactions we will have a good decentralized solution off-chain, so the issue is less pressing.

### Cons

* Some amount of cash needs to be locked up with random nodes most of the time. If one of the nodes is offline, payments can't be cleared through that node. Although, it could not be a big problem as the network is useful for small-ish payments and every node will have 10-15 contracts, so it will tolerate occasional unavailability of some of them. 


Prior art
---------

### Straw Pay and Stroem protocol

Unfortunately, not much is published yet. Most interesting info is in Reddit comments.
 
* [Design info](http://www.reddit.com/r/Bitcoin/comments/2r3ri7/strawpay_cheap_and_secure_micropayments/)
* [Github Account](https://github.com/strawpay)


### Lightning Network

* [Web page](http://lightning.network)
* [Slides](http://lightning.network/lightning-network.pdf)
* [Paper Draft v0.5](http://lightning.network/lightning-network-paper-DRAFT-0.5.pdf)
* [Mike Hearn's analysis](https://medium.com/@octskyward/the-capacity-cliff-586d1bf7715e)


Compatibility
-------------

There are 2 common ways to request a bitcoin payment: a bitcoin URL (BIP21) and a BIP70 Payment Request. (Payments can also be sent non-interactively to a bare address, but for our scheme we need a method of communication with the recipient.) We propose extensions to both of those to ensure the following scenarios:

1. Payer may support both regular Bitcoin payments and the Payment Network.
2. Payer supports exclusively Payment Network.
3. Recipient supports both regular Bitcoin payments (e.g. accepts 0-confirmation transactions) and Payment Network.
4. Recipient requires only Payment Network (e.g. does not want to support 0-confirmation transactions or wait for confirmations).

New MIME type `application/x-paymentnetworkrequest` is used in `Accept` and `Content-type` headers to declare optional or required support for the Payment Network. Both payers and payees may thus negotiate commonly acceptable payment method.

Discovery Protocol
------------------

Discovery may be performed from either end (sender or recipient) or simultaneously from both ends. Note that for transaction to complete, all parties must be *active* and online: they should acknowledge willingness to receive or send IOUs.

TBD.

Message Protocol
----------------

Payment message is an atomic transaction swapping IOUs between all nodes in a given path. When all nodes have placed a signature on the message, it becomes finalized and acts as a receipt that every creditor demonstrates to its debtor.

TBD.






