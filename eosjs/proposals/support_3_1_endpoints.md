# EOS JS Support for 3.1 Endpoints

Looking through the [release notes for 3.1](https://github.com/eosnetworkfoundation/mandel/releases/tag/v3.1.0-rc1) there is support for some APIs which need to be added.

## Send Transaction 2

Need to verify `receipt` and `except` fields in the returned trace.

Need to support same flags as `cleos`
* Use old send RPC
* Option to suppress Return Failure Trace
* Retry irreversible
* Retry num blocks

## Transaction Status

Support this new endpoint
* transaction status command

## Compute Transaction

Support this new endpoint
* read only flag on submissions 
