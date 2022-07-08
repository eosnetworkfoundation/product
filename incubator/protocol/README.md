# Changes to EOSIO protocol

This section organizes various ideas regarding enhancements and other significant changes to the EOSIO protocol that are being explored. 

Here, EOSIO protocol encompasses all of the following:
* the protocol that drives validation of a given blockchain;
* the protocol that any node/client follows to determine whether blocks are considered final;
* the (consensus) protocol for how to reach consensus on the production and finalization of blocks among block finalizers and block proposers;
* and, the network protocol(s) used by nodes in the network to send free transactions upstream for inclusion in a block, receive completed blocks downstream for processing and adding to their local chain, and receive state from peers when automatically bootstrapping a node from a trusted block in the blockchain. 

The changes are organized into the one of the following categories based on the area for which the protocol change primarily adds value:

* [Development utility](#development-utility)
* [Operational efficiency](#operational-efficiency)
* [User convenience](#user-convenience)
* [Complexity reduction](#complexity-reduction)
* [Economics and governance](#economics-and-governance)

## Development utility (DU)

This sub-section includes protocol changes that primarily serve to enable new features for contract/application developers which allow them to achieve more within their applications with less of an implementation and/or maintenance burden imposed on them.

* [BLS12-381 cryptography](bls12/bls12.md)
* [Message passing between execution instances](message_passing/message_passing.md)
* [Provable state domains](provable_state_domains/provable_state_domains.md)
* [Undo sessions](undo_sessions/undo_sessions.md)
* [Long running transactions](long_running_transactions/long_running_transactions.md)

## Operational efficiency (OE)

This sub-section includes protocol changes that primarily attempt to reduce the costs of operators running EOSIO nodes regardless of whether these operators are running the nodes for the purpose of block production, providing APIs, enabling a specific application, or simply to contribute to the security and availability of the network. This includes changes to enable more efficient use of computation and memory as well as sharding / scaling solutions which allow for scaling the network to higher levels of throughput with putting excessive hardware demands on any one node.

* [Distributed atomic transactions](distributed_atomic_transactions/distributed_atomic_transactions.md)

## User convenience (UC)

This sub-section includes protocol changes that primarily simplify some on-chain model (e.g. resources) that are not easily abstracted away at the higher layers from the users of the blockchain or of applications built on an EOSIO blockchain. It also includes changes that can make a significant impact to certain performance metrics that are highly visible and impactful to users, e.g. time to finality.

* [Improved consensus algorithm](improved_consensus_algorithm/improved_consensus_algorithm.md)
* [Generalized on-chain resources](generalized_onchain_resources/generalized_onchain_resources.md)

## Complexity reduction (CR)

This sub-section includes protocol changes that primarily serve the purpose of simplifying the EOSIO protocol to reduce maintenance burden for developers of the protocol and make room for safer and faster development of other valuable enhancements to the EOSIO protocol.

* [Remove deferred transactions](remove_deferred_trx/remove_deferred_trx.md)

## Economics and governance (EG)

This sub-section includes protocols changes that are prerequisites to enabling new behavior at higher layers in the stack (e.g. in the system contracts) that is concerned with the economics or governance of the network and which also do not better fit in one of the other categories.

* [Protocol change events](protocol_change_events/protocol_change_events.md)