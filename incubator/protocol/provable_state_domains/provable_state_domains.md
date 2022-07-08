# Provable state domains

This document describes a proposed [change to the EOSIO protocol](../README.md) to introduce the concept of provable state domains.

## Dependencies on other protocol features

None

## Benefits

* Contract developers can persist within state indices that use arbitrary-length keys. This enables more powerful table abstractions than what is currently possible with the EOSIO database API exposed to contracts. 
* Enables further improvements to allow state snapshots to be automatically validated by nodes while only requiring in the two-thirds supermajority of block finalizers.
* Lays foundation for stateless (or more likely partial state) validation by full validation nodes. This enable a mechanism to scale on-chain state storage beyond what can fit in physical RAM on a typical server hardware.
* It becomes possible for API providers to prove the validity of state queries to client without requiring that the client trust the API provider.

## Details

TODO: Complete description of this new feature.