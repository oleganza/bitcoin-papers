Contract extensions to Bitcoin (DRAFT)
======================================

Oleg Andreev <oleganza@gmail.com>

Introduction
------------

Bitcoin's scripting language is simple, but limited. Scripts act as predicates that enable specific input-output connections. Scripts cannot introspect data outside themselves: for example, they cannot access the balance or other transaction parameters, place constraints on the transaction, keep track of additional state across multiple transactions.

We propose an extension to Bitcoin predicate language that incurs minimal cost of execution, but enables much richer variety of contracts. Smart contracts can introspect their transactions and carry the shared state through several transactions. This extension is enabled as a soft-fork migration (similar to P2SH).

Contracts explicitly work in terms of funds denominated in bitcoin without introducing 3rd party assets. Security of contracts for assets with 3rd party liability (e.g. Open Assets protocol) is equivalent to security of contracts validated by that 3rd party themselves (or an equivalent distributed protocol, but detached from Bitcoin consensus). Only bitcoin-denominated funds are free of counterparty risk and therefore actually benefit from network-enforced contracts. That said, network-enforced contracts are still useful when trading 3rd party assets with bitcoins.

Definitions
-----------

A **contract** is a new version of an output script. Each contract has three parts:

1. Initialization script.
2. Entrance script.
3. Transition script.

For backwards compatibility and to allow soft-fork upgrade, the contract is encoded as follows (note it's similar to P2SH):

```
<Initialization Script> <Entrance Script> DROP DROP <Transition Script Hash> HASH160 EQUAL
```

First two scripts are provided as-is, in a serialized form. Predicate script is represented by its hash like in P2SH.

**Initialization Script** is executed once per output script and sets the initial state of the *Contract Stack*. Typically it pushes single value 0. The script in the contract is present in a serialized form, as PUSHDATA operation.

**Entrance Script** is executed each time new unspent output is added with the given contract. It may do nothing or modify *Contract Stack*, e.g. by incrementing total balance. Like initialization script, it is present in serialized form, as PUSHDATA operation.

**Transition Script** is executed each time the output is spent. Like in P2SH, the contract contains only a 160-bit hash of a serialized transition script. When the output is spent, actual script is appended to input data (such as signatures).

**Contract Stack** is a stack maintained per contract. It is limited to 1024 bytes and 32 items. When the very first output for the contract is added, *initialization script* is executed to provide initial state. Every time output is created, *entrance script* modifies the stack. Every time output is destroyed (spent), *transition script* reads or modifies that stack. 

In addition to classic operators, we introduce new operators. But before we dive into it, lets see an example.

Example
-------

Lets say we want to implement a crowd-funding mechanism with massive control over funds.

N users commit funds invidually to a contract that enables company to unlock funds gradually (e.g. 25% every month over a course of 4 months), but a vote M-of-N allows to pull remaining funds proportionally to every participant (bankruptcy procedure). Company management will be able to commit to long-term contracts based on the contract, but if it mismanages the funds, investors can back out and save the remaining assets.

Formally, we will have 4 distinct contracts: first 3 allow withdrawing up to 1/4 of the total amount of funds while the last one allows all remaining funds to be withdrawn. Note that every distinct pledge submits a separate transaction using the same contract. Each of these contracts allows a bancruptcy contract which implements voting (2/3 majority vote allows every user to withdraw their remaining funds).

The flow of funds can be described with the following chart (F{1,2,3,4} mean funding contracts, B{1,2,3,4} mean bankruptcy contracts):

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
      initialize {
        0 // add initial value zero to Contract Stack
      }
      enter {
        OUTPUTAMOUNT ADD // add output amount to the contract total balance
      }
      func contractAmountRaised {
        FROMCONTRACTSTACK DUP TOCONTRACTSTACK // get the value from contract stack and put back a copy
      }
      clause "Refund initial pledge if total amount is not raised before the deadline" {
        lockTime >= Deadline &&
        contractAmountRaised() < Target &&
        validSignature(backerPubkey)
      }
      clause "Release up to 25% of the funds" {
        contractAmountRaised() >= Target &&
        outputAmount("Month 2") >= currentAmount()*0.75 &&  // require there be an output with a contract continuation locking up at least 75%.
        validSignature(companyPubkey)
      }
      clause "Bankruptcy 1" {
        contractAmountRaised() >= Target &&
        outputAmount("Bankruptcy 1") == currentAmount() && // we must transfer exactly all funds to a voting procedure
        validSignature(backerPubkey)
      }
    }
    
    Contract Script                                                           Bytes          Total 
    ----------------------------------------------------------------------------------------------
    <OP_0>                                                                    1+1            2
    <OP_OUTPUTAMOUNT OP_ADD>                                                  1+2            5
    DROP DROP                                                                 2              7
    <Transition Script Hash> HASH160 EQUALVERIFY                              20+1+1         29
    
    Encoding as an address:
    <version byte> <length> <OP_0> <length> <OP_OUTPUTAMOUNT OP_ADD> <20-byte hash> <4-byte checksum>
    => 1+1+1+1+2+20+4 = 30 bytes
    
    Transition Script                                                         Bytes          Total 
    ----------------------------------------------------------------------------------------------
    IF // refund                                                              1              1
      Deadline CHECKLOCKTIMEVERIFY DROP                                       1+4+1+1        8
      CONTRACTAMOUNT Target LESSTHAN VERIFY                                   1+1+8+2        20
      BackerPubkey CHECKSIG                                                   1+33+1         55
    ELSE                                                                      1              56
      FROMCONTRACTSTACK DUP TOCONTRACTSTACK Target GREATERTHANOREQUAL VERIFY  3+1+8+1+1      70
      IF                                                                      1              71
        // continue                                                                        
        Month2ScriptHash OUTPUTSCRIPTAMOUNT                                   1+20+1         93
        4 MUL amount MUL 3 GREATERTHANOREQUAL VERIFY                          1+1+8+4        107
        CompanyPubkey CHECKSIG                                                1+33+1         142
      ELSE                                                                    1              143
        // bankrupcy                                                          0             
        Bankruptcy1ScriptHash OUTPUTSCRIPTAMOUNT                              1+1+1+23+1     170
        amount EQUALVERIFY                                                    1+1+1+8+1      182
        backerPubkey CHECKSIG                                                 1+33+1         217
      ENDIF                                                                   1              218
    ENDIF                                                                     1              219
    -----------------------------------------------------------------------------------------------
    
    contract "Bankruptcy 1" {
      initialize {
        0 // add initial value zero to Contract Stack
      }
      enter {
        OUTPUTAMOUNT ADD // add output amount to the contract total balance
      }
      func contractAmountRaised {
        FROMCONTRACTSTACK DUP TOCONTRACTSTACK // get the value from contract stack and put back a copy
      }
      clause "Refund the remaining funds if have enough votes" {
        contractAmountRaised() > Target*0.666 &&
        validSignature(backerPubkey)
      }
      clause "Release up to 25% of the funds" { // allow continuation if was not refunded
        outputAmount("Month 2") >= currentAmount()*0.75 &&  // require there be an output with a contract continuation locking up at least 75%.
        validSignature(companyPubkey)
      }
    }
    
    contract "Month 2" {
      clause "Release up to next 25% of the funds" {
        lockTime >= Deadline + 1 month &&
        outputAmount("Month 3") >= currentAmount()*0.50 &&  // require there be an output with a contract continuation locking up at least 50%.
        validSignature(companyPubkey)
      }
      clause "Bankruptcy 2" {
        outputAmount("Bankruptcy 2") == currentAmount() && // we must transfer exactly all funds to a voting procedure
        validSignature(backerPubkey)
      }
    }
    
    contract "Bankruptcy 2" {
      initialize {
        0 // add initial value zero to Contract Stack
      }
      enter {
        OUTPUTAMOUNT ADD // add output amount to the contract total balance
      }
      func contractAmountRaised {
        FROMCONTRACTSTACK DUP TOCONTRACTSTACK // get the value from contract stack and put back a copy
      }
      clause "Refund the remaining funds if have enough votes relative to remaining funds" {
        contractAmountRaised() > Target*0.75*0.666 &&
        validSignature(backerPubkey)
      }
      clause "Release up to next 25% of the funds" { // allow continuation if was not refunded.
        lockTime >= Deadline + 1 month &&
        outputAmount("Month 3") >= currentAmount()*0.50 &&  // require there be an output with a contract continuation locking up at least 50%.
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
        outputAmount("Bankruptcy 4") == currentAmount() &&
        validSignature(backerPubkey)
      }
    }
    
    contract "Bankruptcy 4" {
      initialize {
        0 // add initial value zero to Contract Stack
      }
      enter {
        OUTPUTAMOUNT ADD // add output amount to the contract total balance
      }
      func contractAmountRaised {
        FROMCONTRACTSTACK DUP TOCONTRACTSTACK // get the value from contract stack and put back a copy
      }
      clause "Refund the remaining funds if have enough votes relative to remaining funds" {
        contractAmountRaised() > Target*0.25*0.666 &&
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

**contractAmountRaised()** is an amount of funds being deposited to this contract (not only in unspent outputs, but also in the spent ones). The validating nodes can increment this balance and keep track of it until there are no unspent outputs for that contract (the balance can be removed from index when no output on this contract remains).

**outputAmount(script)** finds the first output in the spending tx with the matching script and returns its balance. Returns 0 if the output is missing.

**currentAmount()** is the amount of funds on the current output containing contract (see `OP_CURRENTAMOUNT`).

**validSignature** is verification of the signature on the spending transaction with a given public key



Operators
---------

The execution of the new operators is limited to contract scripts only, so we do not need to repurpose existing NOP operators. Instead we allocate unused values starting with 0xC0 ("C" stands for "Contracts").

Name                   | Value       |  Description
:----------------------|:------------|:-----------------------------------
OP_CHECKMULTISIG       | 0xae        | (*signatures* *M* *pubkeys* *N* → true/false) Verifies transaction signatures (with hashtype byte) against the given public keys. Signatures must be provided in the same order as corresponding public keys. This fixes "one too many" bug.
OP_CHECKLOCKTIMEVERIFY | 0xb1        | (former OP_NOP2) verifies that nTimeLock matches the expected time according to BIP65.
OP_TOCONTRACTSTACK     | 0xc0        | Moves item from the stack to contract stack. Available only in transition script.
OP_FROMCONTRACTSTACK   | 0xc1        | Moves item from the contract stack to stack. Available only in transition script.
OP_CURRENTAMOUNT       | 0xc2        | Pushes amount being transferred in the current context. Entrance script receives current output amount. Transition script receives current input amount. Initialization script fails if this opcode is executed.
OP_OUTPUTSCRIPT        | 0xc3        | (index → script) Pops index of an output from the stack and pushes corresponding output script to the stack. Available only in transition script.
OP_OUTPUTAMOUNT        | 0xc4        | (index → amount) Pops index of an output from the stack and pushes corresponding output amount to the stack. Available only in transition script.
OP_OUTPUTSCRIPTAMOUNT  | 0xc5        | (hash160 → amount) Pops Hash160 of an output script and pushes sum of amounts on all outputs with scripts matching the hash. If no output found, pushes 0. Available only in transition script.

**Expansion Opcodes:**

Name                   | Value       |  Description
:----------------------|:------------|:-----------------------------------
OP_NOP11               | 0xc6        | Does nothing. Used for future expansion.
OP_NOP12               | 0xc7        | Does nothing. Used for future expansion.
OP_NOP13               | 0xc8        | Does nothing. Used for future expansion.
OP_DROP1               | 0xc9        | Drops the top value. Used for future expansion.
OP_DROP2               | 0xca        | Drops the top value. Used for future expansion.
OP_DROP3               | 0xcb        | Drops the top value. Used for future expansion.
OP_RESERVED3           | 0xcc        | Disabled opcode. If executed, transaction is invalid. Used for future expansion.
OP_RESERVED4           | 0xcd        | Disabled opcode. If executed, transaction is invalid. Used for future expansion.
OP_RESERVED5           | 0xce        | Disabled opcode. If executed, transaction is invalid. Used for future expansion.
OP_RESERVED6           | 0xcf        | Disabled opcode. If executed, transaction is invalid. Used for future expansion.

Notes
-----

* Note that this architecture allows "selling contracts". The contract can ensure that it is transferred into itself or compatible contract.
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



