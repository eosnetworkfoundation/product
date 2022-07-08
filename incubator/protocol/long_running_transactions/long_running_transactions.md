# Long running transactions

This document describes a proposed [change to the EOSIO protocol](../README.md) to introduce the a type of transaction that can extend its execution across several blocks while periodically committing state side-effects as of particular milestones along the execution process at particular points within the blockchain. 

## Dependencies on other protocol features

* [Undo sessions](../undo_sessions/undo_sessions.md)

## Benefits

* Transactions that need to take a long time to execute can be realized in a simpler way for the contract developer. Currently they must painstakingly segment the overall computation into small chunks where each can fit within the short transaction deadlines (typically less than 30 milliseconds). In addition, that more cumbersome current approach has a lot more computational overhead as the interim state must be saved to and restored from persisted state for each chunk of computation.

## Details

TODO: Complete description of this new feature.