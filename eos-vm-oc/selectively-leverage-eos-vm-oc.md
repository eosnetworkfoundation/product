# Selectively leverage EOS VM OC

## Context
The Antelope protocol is known for being high performance blockchain technology. Even with its current status as a top performer, there are still significant optimizations that can be made. One such optimization is to utilize an optimizing compiler to reduce transaction CPU consumption. EOS VM OC is an optimizing compiler which compiles WebAssembly instructions to native machine instructions. Nominally this results in contract execution that is several times faster than baseline EOS VM. In certain cases, contract execution is faster by orders of magnitude (10000x faster or more).

### Key Considerations
While there are compelling performance gains to be had by leveraging EOS VM OC for contracts, a feasible solution must solve for some important considerations.

#### 10000x Transactions
In a network where node usage of EOS VM OC is heterogenous, a block containing a transaction with a significant execution time disparity between EOS VM OC and baseline will force nodes that aren't running EOS VM OC to consume CPU time that is orders of magnitude higher to apply that transaction. This is what we'll refer to as a "10000x Transaction".

As an illustrative trivial example, consider a WASM that loads a value from the stack and forgets about it (“drops it”). OC will just remove that code – literally no machine code will be generated. Yet the baseline EOS VM will dutifully execute that code opcode by opcode.

#### Optimization Permanence
Once a 10000x transaction is in a block that leverages EOS VM OC’s aggressive optimization it’s there forever. Any replays of the chain must have a WASM runtime that performs similar aggressive optimization.


## Proposal Summary

### In Scope

#### Auto OC Mode as default
We propose to add a new Auto OC Mode option (`eos-vm-oc-enable=auto`), and apply it as the default.

#### Baseline / OC selection
We propose that in Auto OC Mode, EOS VM OC will be leveraged selectively, as follows:

| Context                               | OC?                                                             |
|---------------------------------------|-----------------------------------------------------------------|
| Building block                        | baseline, but OC if account is eosio.*                          |
| Applying block                        | OC, _or_<br>if producer: baseline, but OC if account is eosio.* |
| Speculatively executing trx from HTTP | baseline, but OC if account is eosio.*                          |
| Speculatively executing trx from p2p  | OC (but see :bulb: below)                                       |
| Compute transaction                   | baseline, but OC if account is eosio.*                          |
| Read only transaction                 | OC                                                              |

:bulb: There is some uncertainty around leveraging OC for P2P because seed nodes may need to offer some protection against spam. This may need be adjusted to baseline too.

#### Evaluate increasing subjective limits for OC contracts
We propose an analysis to determine the most appropriate subjective limits for contracts that leverage OC. Depending on the results of the analysis, subjective limits may be increased for eosio.* contracts.

### Out of Scope
#### Protocol Feature
In discussion about this proposal, we determined that the addition of a protocol feature would be excessive and unnecessary.

#### OC Contract Whitelist
The possibility of generalizing from eosio.* to any account via a whitelisting system was considered. Ultimately it was decided that any potential defects would need more consideration and that hardcoding to just eosio.* was a sufficient starting point.

## Full exploration of several options
### Background
EOS VM OC is supported with the exception of block production. Prior research exploring the possibility of enabling it for BPs produced results suggesting it is not viable to blanket enable EOS VM OC for block production without substantial protocol changes.

There are three sets of users that don't use EOS VM OC:
1. BPs (because they can't)
2. RPC nodes (to estimate transaction CPU cost)
3. Users who simply don't know to enable OC (because it is currently off by default)

RPC nodes run baseline EOS VM to avoid accepting transactions that will ultimately be dropped on BP nodes due to execution time.

With the introduction of the parallel read only transaction window an RPC node may actually serve two edge execution purposes: as a way to submit transactions, and as a way to submit read only transactions executed in parallel. The read only transactions aren't concerned with estimating time accurately; they just want to get done _fast_. A simple blanket OC on/off knob as exists today definitely doesn't work well in this use case (though, an RPC provider could likely work around this mismatch by proxying read only queries to a separate node vs the one used to accept transactions; but that may mean double the nodes, and not _everyone_ is operating a public RPC node farm -- some users  are just running one private node for their dapp).

## "Auto" OC Mode

### Simple

Consider instead a new `eos-vm-oc-enable` option in nodeos: not just on/off, but let's say "auto" (other names certainly welcome!) When using the new default `eos-vm-oc-enable=auto` usage of baseline or OC will be decided on a more granular basis.

| Context                               | OC?      |
|---------------------------------------|----------|
| Building block                        | baseline |
| Applying block                        | OC       |
| Speculatively executing trx from HTTP | baseline |
| Speculatively executing trx from p2p  | OC       |
| Compute transaction                   | baseline |
| Read only transaction                 | OC       |

:::info
Originally 4.0 was to have the speculative block removed which makes these classifications above considerably easier to handle. Speculative block has been restored though, so transaction origin will need to be tracked likely as part of the `transaction_metadata` -- still not too hard.
:::

This new default decision matrix on when to use OC will align a lot better with both what users want and what OC is supported for.

:bulb: In review of the V1 document there was some uncertainty on if p2p should be OC or baseline because seed nodes may need to offer some protection against spam. This may need be adjusted to baseline too.

### Exclude Producers From OC Block Application

There is something that some may consider a little risky about the above though. Even though a producer is still building blocks with the baseline, it's building _on top of a block applied via OC_. This definitely means "the chain is being built with a mix of two WASM runtimes" so to speak, which may be considered risky. Another possible matrix of `eos-vm-oc-enable=auto` might instead be

| Context                               | OC?                                 |
|---------------------------------------|-------------------------------------|
| Building block                        | baseline                            |
| Applying block                        | OC, _unless configured as producer_ |
| Speculatively executing trx from HTTP | baseline                            |
| Speculatively executing trx from p2p  | OC (but see :bulb: above)                                  |
| Compute transaction                   | baseline                            |
| Read only transaction                 | OC                                  |

In this matrix, blocks are only ever being built with baseline on top of blocks applied also only by baseline. It should alleviate the possibility of an execution difference introduced in one block causing a future poison block, for example.

With nodeos' default being to use OC to apply blocks, that means all or nearly all nodes will be using OC to apply blocks, which opens up the possibility for another improvement...

## Enable OC On "System" Contracts

The primary problem with the safety of EOS VM OC is that a contract could abuse the optimizer to have OC execute transactions 10000x or faster than baseline. This issue, and others, are discussed in depth in the previously referenced documents.

But not all contracts are under control of randos. There are a set of contracts that are in control of BPs, and the BPs are unlikely to put OC-abusing code on these accounts (BPs have far easier ways to attack the network!). So instead we could reasonably safely modify the matrix above to state that for all contracts on eosio.* accounts OC should be used. This would include contracts such as,
* eosio.token
* eosio.system
* eosio.ibc
* eosio.evm

So a node configured with the default `eos-vm-oc-enable=auto` would have its decision matrix modified accordingly

| Context                               | OC?                                                           |
|---------------------------------------|---------------------------------------------------------------|
| Building block                        | baseline, but OC if account is eosio.*     |
| Applying block                        | OC, _or_<br>if producer: baseline, but OC if account is eosio.* |
| Speculatively executing trx from HTTP | baseline, but OC if account is eosio.*     |
| Speculatively executing trx from p2p  | OC (but see :bulb: above)                                                           |
| Compute transaction                   | baseline, but OC if account is eosio.*     |
| Read only transaction                 | OC                                                            |

The main OC specific risk to the above (other then building blocks on top of multiple runtimes) is that one of the system contracts may end up being too close to the subjective tier up limit and some nodes will fail to tier up. But since BPs generally only deploy known contracts that the ENF develops and/or reviews, we can easily make some measurements to gauge if we're in a safe ballpark for existing contracts, and preemptively measure for new contracts.

We may want to slightly tewak OC's tier up behavior to prioritize these system contracts to protect against "tier up spam" blocking system contract tier up, and we likely want to "pin" these OC compilation in OC's code cache.

We will need to look closely at EOS VM OC tech debt to determine if there are any major or blocking concerns that need to be addressed before allowing this new behavior. [#673](https://github.com/AntelopeIO/leap/issues/673), [#692](https://github.com/AntelopeIO/leap/issues/692), and [mandel#641](https://github.com/eosnetworkfoundation/mandel/issues/641) are all large tech debt items that could be considered; how we handle LLVM versioning is also an important discussion topic.
