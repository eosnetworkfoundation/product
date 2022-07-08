# Undo sessions

This document describes a proposed [change to the EOSIO protocol](../README.md) to introduce a mechanism that allows contracts to establish stackable sessions that can be undone under the direction of the contract code. Undoing a session will undo any side-effects to the persisted state created by the undone session.

## Dependencies on other protocol features

* [Message passing between execution instances](../message_passing/message_passing.md)
* [Provable state domains](../provable_state_domains/provable_state_domains.md)

## Benefits

* Contracts can execute code that can fail in a mini-sandbox but not permit it to abort the transaction. Instead the calling code can resume execution after failure and perhaps even act on the failure reason.
* Contract developers can achieve their goals without having to completely rewrite critical portions of the code in way that it never can create an error.

## Details

TODO: Complete description of this new feature.