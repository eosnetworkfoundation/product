# Generalized on-chain resources

This document describes a proposed [change to the EOSIO protocol](../README.md) to generalize how on-chain resource utilization is tracked and enforced again quotas. Two different resource types are both considered in the changes of this proposal: ephemeral on-chain computational resources (capturing the replacement to CPU/NET); and, persisted on-chain state storage resources (effectively the evolution of the RAM resource).

## Dependencies on other protocol features

* [Message passing between execution instances](../message_passing/message_passing.md)

## Benefits

* More modular contracts.
* Enables better designed contract APIs that can be used by other contracts.
* Lays sensible foundation for good contract standards and greater generality and flexibility in the evolution of the EOSIO protocol.
* Introduces fundamental mechanism of breaking up computation in a manner that can later lead to parallelization of contract code execution.

## Details

TODO: Complete description of this new feature.