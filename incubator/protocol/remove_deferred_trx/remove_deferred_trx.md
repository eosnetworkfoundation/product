# Remove deferred transactions

This document describes a proposed [change to the EOSIO protocol](../README.md) to remove deferred transactions.

## Dependencies on other protocol features

None

## Benefits

* Deferred transactions have a way of making certain consensus bugs that are unfortunate but ultimately easy to fix into undesired behavior that is forever captured in the blockchain through a poison block which forces us to keep the undesired behavior in the protocol long after fixing the bug for the sake of allowing replayability. Even worse, if the consensus bug is non-deterministic, the poison block may require custom variants of the EOSIO protocol on different EOSIO blockchain instances that encountered the bug. While removing deferred transactions does not prevent such scenarios from occurring, it does lower the probability that they will occur.
* Deferred transactions are unreliable so contract developers already need to put effort into creating workarounds that do not depend on them. Since they already must do this work, having deferred transactions available in the protocol not only does not add meaningful value but it actually adds cognitive burden that they need to consider to ensure it cannot be used by attackers to work around the intention of their contract. Removing deferred transaction would reduce the cognitive burden for contract developers without taking away from any functions they should already be relying on (especially since deferred transactions have been marked deprecated for a long time at this point).
* Deferred transactions slow down development of some EOSIO protocol features by EOSIO blockchain developers as they need to consider how the new feature or changed protocol behavior may interact with deferred transactions in unexpected ways. Removing deferred transaction would improve productivity of EOSIO blockchain developers.

## Details

TODO: Complete description of how deferred transactions can be removed from the protocol.