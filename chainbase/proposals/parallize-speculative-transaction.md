# Exploring Linux mmap() Copy-on-Write feature to Parallelize the Speculative Transaction Execution

## Context
https://github.com/eosnetworkfoundation/product/issues/95 described the possibility to achieve speculative transaction execution by rapidly cycling  `nodeos` through exclusive write mode and read-only modes. In read-only mode, block syncing would be paused to ensure nothing is written to chainbase. Transactions in the transaction queue are dispatched to a thread pool to access chainbase in parallel. Nevertheless, we propose to explore the the  `mmap()` copy-on-write feature on Linux to allow even better parallelization.

## Possible Solution

On Linux, the `mmap()` API allows to create a private copy-on-write mapping with `MAP_PRIVATE` flag. Updates to the mapping are not visible to other processes mapping the same file, and are not carried through to the underlying file.  It is unspecified whether changes made to the file after the `mmap()` call are visible in the mapped region.

To make this work in the context of chainbase, we need to create a new private mapping every time a new block is about to be constructed, and the state change can only be applied to the new mapping, the old mapping can then be used for read-only transactions or writes back to disk on scheduled interval. With this approach, it can also reduce possibility of corrupted `shared_memory.bin` on the disk.

However, it is unknown to us for the efficiency in the Linux kernel to copy memory pages and how it affects the regular non-ready-only transaction execution; therefore, it may require some prototyping and performance evaluation before we are ready to commit on this approach.