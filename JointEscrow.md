Joint Escrow
============

Oleg Andreev <oleganza@gmail.com>



Introduction
------------

Typically trust is required when two parties enter a contract. This could be trust in each other's performance, or trust in 3rd party that acts as an insurer or arbiter. We propose a Bitcoin scheme that allows to establish trust between any two parties (humans or autonomous programs) on a purely game-theoretic basis. Before executing a contract valued at X amount of money, both parties simultaneously lock up 2*X amounts in a single 2-of-2 multi-signature output. They can only unlock those funds if they both agree on conditions. This puts them in the Nash equilibrium: neither of them is interested in cheating since the value to gain is noticebly lower than the value to lose. Therefore they both find a way to complete or cancel contract to mutual satisfaction in order to get back their locked funds. The scheme removes the need for reputation or trusted 3rd parties and enables fully private and autonomous operation for humans and machines.

Example 1 
---------

Alice from Australia wants to sell her old rusty iPod for $100. Bob from Brazil finds her listing and wants to purchase the iPod. Someone must act first: either Alice sends the iPod, or Bob sends the payment. Neither party trusts each other and it's too expensive or impossible to employ a trusted 3rd party to observe the process.

They can solve the problem by using Join Escrow: each of them locks up $200 in a single 2-of-2 multisig output where signatures from both Alice and Bob are required to unlock total $400. 

Lets say Alice sends first. When Bob receives the iPod, he gets what he asked for worth $100, but he has $200 still locked up. He composes and half-signs a transaction that unlocks $400 in a way that $300 goes back to Alice and $100 goes back to Bob. If Alice agrees with that proportion, she signs and publishes that transaction. Proportion could be different if both Alice and Bob agree on a partial or complete refund.

Example 2
---------

Alice is a designer who agreed to create a 10-page website for Bob. The total amount of order is $10000. To avoid locking up $20000 from each side, parties may break down the contract in 10 steps (one per website's page) and only lock up $2000 from each side. After completion of each step they exchange the product and the payment of $1000. Then they decide to move to next step or terminate the contract early. This way, the Join Escrow becomes a kind of a "trust channel" that can be reused indefinitely as long as both parties are satisfied.


Illustration
------------

#### Joint Escrow Transaction

Alice promises something worth $100 to Bob. Both lock up twice as much for protection.

Inputs             |      | Output                          |      |
-------------------|------| --------------------------------|------| 
Alice's signature  | $200 | 2 Alice Bob 2 OP_CHECKMULTISIG  | $400 |
Bob's signature    | $200 |                                 |      |



#### Unlock+Payment Transaction

Bob has received a good or service from Alice. They both sign and unlock funds adjusted for the payment amount (Alice earns $100).

Inputs                              |      | Output           |      |
------------------------------------|------| -----------------|------| 
Alice's signature, Bob's signature  | $400 | Alice's Address  | $300 |
                                    |      | Bob's Address    | $100 |

#### Withholding-Prevention Transaction

(see below for details)

Inputs                              |      | Output           |      |
------------------------------------|------| -----------------|------| 
Alice's signature, Bob's signature  | $400 | Null Address     | $400 |
Lock time: 30 days in the future.   |      |                  |      |



Escrow Factor
-------------

Escrow factor could be specified depending on application. E.g. first sender (merchant) may lock up `1*X` while the payer (customer) may lock up `2*X`. It ultimately depends on how both parties perceive *fairness* of the escrow. 


Withholding Unlocked Funds
--------------------------

It is possible that once Bob half-signs transaction that unlocks funds, Alice keeps her copy of that transaction indefinitely, as a part of her "pension fund". She technically can spend her portion of funds any time, but Bob must wait for her to do so to access his funds. To prevent this attack, after locking up their funds, but before executing the contract, both parties may sign a time-locked transaction that spends both their funds to an unspendable address (e.g. all-zero). This way, if Alice tries to withhold a half-signed transaction, she risks having all funds permanently destroyed by Bob after a certain point in time. So she has additional incentive to play nice and finalize the transaction as soon as possible.


Features
--------

* Unlock process may include payment for a good or service in a single transaction.
* Join Escrow enables markets that otherwise are impossible: either when transacting parties cannot have prior reputation (e.g. one-time Ebay sellers/buyers), or trusted 3rd party would be too expensive or impossible to use (e.g. both parties trade inexpensive goods across the globe).
* Absolute privacy is enabled since transacting parties do not need to rely on reputations or any other persistent identification.
* Bots can transact on the internet autonomously. They essentially "buy trust" from each other and then provide necessary services. This makes Joint Escrow an important building block in the next generation of distributed markets and networks.

Drawbacks
---------

* Relatively significant capital investment is required which limits it to relatively inexpensive contracts. However, expensive contracts can be broken down into less expensive steps (see Example 2).
* Nash equilibrium works if both actors behave rationally. However, one party may hurt another at significant expense to itself. However, in some applications that risk could be considered tolerable (e.g. when alternatives to conduct trade are simply not available or much more expensive).
* Both parties must stay alive till their contract is fulfilled. Otherwise funds will remain locked forever. Using time-locked refund transaction is useless as it defeats the purpose of the scheme. In the Example 1, when Alice sends the iPod, a time-locked refund will allow Bob to never pay back and safely receive his money some time later.



References
----------

* [Video of my talk about Joint Escrow on February 13, 2014](http://www.bitcoinomie.fr/2014/02/18/compte-rendu-paris-bitcoin-startups-1/) ([slides](http://oleganza.com/bitcoin-epita-2014.pdf))

* [Bitmarkets](http://voluntary.net/bitmarkets/) app, a working implementation of a private market using Joint Escrow.

* [NashX market](http://nashx.com) that inspired me to design a scheme without 3rd party.

* [Nash Equilibrium](http://en.wikipedia.org/wiki/Nash_equilibrium) on Wikipedia.