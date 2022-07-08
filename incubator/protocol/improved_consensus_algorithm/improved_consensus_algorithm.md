# Improved consensus algorithm

This document describes a proposed [change to the EOSIO protocol](../README.md) to transition to a new consensus algorithm that replaces the existing one used in the EOSIO protocol. The new consensus algorithm would have many desireable properties that improves upon the existing algorithm, including but not limited to faster finality and mathematical proofs of accountable safety.

## Dependencies on other protocol features

* [BLS12-381 cryptography](../bls12/bls12.md)

## Benefits

* Faster finality.
* Greater confidence in security of the consensus algorithm through mathematical proofs of accountable safety and plausible liveness.
* Smaller sized proofs of block finalization. This reduces space used in blockchain but it also reduces transactions costs in providing finality of blocks of a source blockchain within a contract executing on a destination chain (which is a significant factor in the economics of IBC).
* Enable a larger number of block finalizers without sacrificing time-to-finality.
* Cheap cryptographic proofs of slashable behavior (which can lead to a finality violation if enough block finalizers participate) allows introducing slashing time-locked funds as an enhancement to the consensus mechanism to further align economic incentives of block finalizers and increase security.

## Details

Details regarding this feature has already been captured in an EOSIO+ Coalition [RFP](https://github.com/eosnetworkfoundation/Coalition-RFPs/blob/main/2022%2004%20RFP%20-%20Faster%20Finality.pdf).