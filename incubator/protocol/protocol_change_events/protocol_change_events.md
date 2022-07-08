# Protocol change events

This document describes a proposed [change to the EOSIO protocol](../README.md) to introduce a mechanism that can be used by privilege smart contracts to emit an event capturing a protocol change which allows nodes to opt into follow the chain across that protocol change. This allows changes that the EOSIO protocol provides the flexibility to not be hard forking changes to be optionally (by the discretion of the system contracts) considered hard forking changes.

## Dependencies on other protocol features

None

## Benefits

* Especially when combined with the [message passing between execution instances](../message_passing/message_passing.md) protocol change to allow for more modular system contracts, certain critical functions managed by the system contract (such as enforcement of the maximum inflation rate of the core token) can be protected as hard forks to optionally provide users of EOSIO blockchains greater immutability guarantees that they may expect from other blockchains.

## Details

See this [forum post](https://forums.eoscommunity.org/t/idea-mechanisms-to-address-concerns-regarding-ease-with-which-changes-can-be-made-on-general-programmable-blockchains-like-eosio/2587) for more details.

TODO: Complete description of this new feature.