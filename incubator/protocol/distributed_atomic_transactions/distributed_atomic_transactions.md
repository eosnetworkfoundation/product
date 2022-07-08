# Distributed atomic transactions

This document describes a proposed [change to the EOSIO protocol](../README.md) to introduce distributed atomic transactions that can efficiently work across multiple EOSIO blockchains.

## Dependencies on other protocol features

* [Long running transactions](../long_running_transactions/long_running_transactions.md)

## Benefits

* Allows application logic to be shared across many different blockchains with minimal changes required to the application logic by the application developer.
* A more convenient and efficient form of IBC for any applications where the communication between two blockchains requires messages to be passed back and forth more than once or where the communication must occur across three or more blockchains that are collectively carrying out an atomic operation.

## Details

TODO: Complete description of this new feature.