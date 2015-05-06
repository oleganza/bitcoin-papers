Turing-complete smart contract engine via Bitcoin soft fork (DRAFT)
===================================================================

Oleg Andreev <oleganza@gmail.com>

Introduction
------------

Bitcoin's scripting language is simple, but limited. Scripts act as predicates that enable specific input-output connections. Scripts cannot introspect data outside themselves: see the balance or other transaction parameters, place constraints on the transaction, keep track of additional state etc.

We propose an extension to Bitcoin predicate language which is still being constrained to incur minimal costs of execution, but allowing much richer variety of contracts. Smart contracts can carry the state through transaction as an explicitly added data to the contract itself. This extension is enabled as a soft-fork P2SH-like migration.

Contracts explicitly work in terms of funds denominated in bitcoin without introducing counterparty assets. Security of contracts for assets with counterparty liability is equivalent to security of contracts validated by counterparty itself (or an equivalent distributed protocol, but detached from Bitcoin consensus). Only bitcoin-denominated funds are free of counterparty risk and therefore actually benefit from on-chain contracts. However, when trading counterparty assets with bitcoins, on-chain smart contracts become useful.

Example
-------

Lets say we want to implement a crowd-funding mechanism with massive control over funds.

N users commit funds invidually to a contract that enables company to unlock funds gradually (e.g. 1/4 every month over a course of 4 months), but a vote M-of-N allows to pull remaining funds proportionally to every participant (bankruptcy procedure). Company management will be able to commit to long-term contracts based on the contract, but if it mismanages the funds, investors can back out and save the remaining assets.

Formally, we will have 4 distinct contracts: first 3 allow withdrawing up to 1/4 of the total amount of funds while the last one allows all remaining funds to be withdrawn. Note that every distinct pledge submits a separate transaction using the same contract. Each of these contracts allows a bancruptcy contract which enables to implement voting (e.g. 2/3 majority vote).

The flow of funds can be described with the following chart (F1 means a funding contract, B1 means a bankruptcy contract):

           Backer
             |
             F1
           / |  \ 
     Backer  |   B1
             |  /  \
             | /    Backer
          F2 + Company
             |  \ 
             |   B2
             |  /  \
             | /    Backer
          F3 + Company
             |  \
             |   B3
             |  /  \
             | /    Backer
          F4 + Company
             |  \
             |   B4
             |  /  \
             | /    Backer
          Company

Note that in case of failure to vote for bankrupcy, funds can proceed towards the next contract. The next attempt to bankrupcy must use a different bankrupcy contract to avoid counting deposits made on the first one.

    contract "Month 1" {
      clause "Refund initial pledge if total amount is not raised before the deadline" {
        lockTime >= Deadline &&
        contractAmount < Target &&
        validSignature(backerPubkey)
      }
      clause "Release up to 25% of the funds" {
        contractAmount >= Target &&
        outputAmount("Month 2") >= amount*0.75 &&  // require there be an output with a contract continuation locking up at least 75%.
        validSignature(companyPubkey)
      }
      clause "Bankruptcy 1" {
        contractAmount >= Target &&
        outputAmount("Bankruptcy 1") == amount && // we must transfer exactly all funds to a voting procedure
        validSignature(backerPubkey)
      }
    }
    
    Opcodes:                                                               Bytes          Total 
    ----------------------------------------------------------------------------------------------
    IF // refund                                                           1              1
      Deadline CHECKLOCKTIMEVERIFY                                         1+4+1          7
      CONTRACTAMOUNT Target LESSTHAN VERIFY                                1+1+8+2        19
      backerPubkey CHECKSIG                                                1+33+1         54
    ELSE                                                                   1              55
      CONTRACTAMOUNT Target GREATERTHANOREQUAL VERIFY                      1+1+8+1+1      67 
      IF                                                                   1              68
        // continue                                                                     
        0 OUTPUTSCRIPT Month2Script EQUALVERIFY                            1+1+1+23+1     95
        0 OUTPUTAMOUNT 4 MUL amount MUL 3 GREATERTHANOREQUAL VERIFY        4+1+8+4        112             
        companyPubkey CHECKSIG                                             1+33+1         147             
      ELSE                                                                 1              148         
        // bankrupcy                                                       0             
        0 OUTPUTSCRIPT Bankruptcy1Script EQUALVERIFY                       1+1+1+23+1     175             
        0 OUTPUTAMOUNT amount EQUALVERIFY                                  1+1+1+8+1      187             
        backerPubkey CHECKSIG                                              1+33+1         222             
      ENDIF                                                                1              223
    ENDIF                                                                  1              224
    -----------------------------------------------------------------------------------------------
    
    contract "Bankruptcy 1" {
      clause "Refund the remaining funds if have enough votes" {
        contractAmount > Target*0.666 &&
        validSignature(backerPubkey)
      }
      clause "Release up to 25% of the funds" { // allow continuation if was not refunded
        outputAmount("Month 2") >= amount*0.75 &&  // require there be an output with a contract continuation locking up at least 75%.
        validSignature(companyPubkey)
      }
    }
    
    contract "Month 2" {
      clause "Release up to next 25% of the funds" {
        lockTime >= Deadline + 1 month &&
        outputAmount("Month 3") >= amount*0.50 &&  // require there be an output with a contract continuation locking up at least 50%.
        validSignature(companyPubkey)
      }
      clause "Bankruptcy 2" {
        outputAmount("Bankruptcy 2") == amount && // we must transfer exactly all funds to a voting procedure
        validSignature(backerPubkey)
      }
    }
    
    contract "Bankruptcy 2" {
      clause "Refund the remaining funds if have enough votes relative to remaining funds" {
        contractAmount > Target*0.75*0.666 &&
        validSignature(backerPubkey)
      }
      clause "Release up to next 25% of the funds" { // allow continuation if was not refunded.
        lockTime >= Deadline + 1 month &&
        outputAmount("Month 3") >= amount*0.50 &&  // require there be an output with a contract continuation locking up at least 50%.
        validSignature(companyPubkey)
      }
    }
    
    ...
    
    contract "Month 4" {
      clause "Release remaining funds" {
        lockTime >= Deadline + 3 months &&
        validSignature(companyPubkey)
      }
      clause "Bankruptcy 4" {
        outputAmount("Bankruptcy 4") == amount &&
        validSignature(backerPubkey)
      }
    }
    
    contract "Bankruptcy 4" {
      clause "Refund the remaining funds if have enough votes relative to remaining funds" {
        contractAmount > Target*0.25*0.666 &&
        validSignature(backerPubkey)
      }
      clause "Release remaining funds" {
        lockTime >= Deadline + 3 months &&
        validSignature(companyPubkey)
      }
    }

Where:

**clause** is part of the contract triggered by the spending transaction (transaction specifies explicitly which clause to trigger).

**lockTime** is a lock time specified in the spending transaction.

**Deadline** is a constant value in terms of time or block height specifying time limit for fundraising. Clause cannot be triggered before the deadline or if the *Target* amount is reached.

**Target** is an amount of funds being raised within a contract.

**contractAmount** is an amount of funds being deposited to this contract (not only in unspent outputs, but also in the spent ones). The validating nodes can increment this balance and keep track of it until there are no unspent outputs for that contract (the balance can be removed from index when no output on this contract remains).

**outputAmount(script)** finds the first output in the spending tx with the matching script and returns its balance. Returns 0 if the output is missing.

**amount** is the amount of funds on the current output containing contract.

**validSignature** is verification of the signature on the spending transaction with a given public key


Notes
-----

* What happens when you add up some balance later? 
* Inaccuracy of fractions can be mitigated by rational numbers (e.g. instead of `a > 0.75*b` we could do `4*a >= 3*b` or `a DUP ADD DUP ADD b DUP DUP >=`)
* What if someone unrelated to original funding process deposits enough funds to enable bankcruptcy? Those who wish to continue funding may do so, others can exit.
* Note that we refer amounts to a constant Target instead of an actual currently locked amount (imagine the target is 1000, but actually 4000 were provided). Additional funds can be deposited to a contract any time. The difficulty is to synchronize on which amount is considered latest, so each individual transaction refers to the same amount. One possibility is to provide a barrier up-front ("we count only funds raised before date X"). This slightly complicates calculation and storage of the deposits and does not solve a problem of influx of extra funds to affect voting process.
* Alternative way to do multi-party computation is serial: Bob "spends" Alice's contract under condition of preserving funds and contract (but can add more).
  A 100 -> A 120 -> A 170 -> A 270 (but then we lose knowledge of individual shares)
* Idea: instead of rather ad hoc "contractAmount" use actions and persistant per-contract stack (with limits) and "actions" triggered on deposits.

Design Considerations
---------------------

To support truly massive contracts (millions of participants), we must support independent transactions on the blockchain, to be able to spread the commitments to the contract across multiple transactions.


Design Considerations
---------------------

To support truly massive contracts (millions of participants), we must support independent transactions on the blockchain, to be able to spread the commitments to the contract across multiple transactions.

