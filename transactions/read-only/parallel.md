# Parallelize Read-only Transaction Execution

Continuing PR (https://github.com/AntelopeIO/leap/pull/558) (on branch https://github.com/AntelopeIO/leap/tree/send_read_only_trx), this document describes an approach to parallelize read-only transaction execution.

## Existing Requests Handling

### HTTP RPC Requests

#### Chain APIs
Chain APIs can be classified into reads and writes.
- Reads are those whose names start with `get_`, like `get_info`, `get_activated_protocol_features`, `get_block`, `get_block_info` ... They do not modify states.
- Writes are the rest of requests: `compute_transaction`, `push_transaction`, `push_transactions`, `send_transaction`, `send_transaction2`, and `push_block`.  They may modify states.

Chain APIs are received on the HTTP thread, processed on the main thread (and producer thread for non-get requests), and responses are sent on the HTTP thread.

| API                             |  Data modified                | read-only thread safe  |
|---------------------------------|-------------------------------|------------------------|
| get_info                        |  none                         | yes                    |
| get_activated_protocol_features |  none                         | yes                    |
| get_block                       |  none                         | yes                    |
| get_block_info                  |  none                         | yes                    |
| get_block_header_state          |  none                         | yes                    |
| get_account                     |  none                         | yes                    |
| get_code                        |  none                         | yes                    |
| get_code_hash                   |  none                         | yes                    |
| get_abi                         |  none                         | yes                    |
| get_raw_code_and_abi            |  none                         | yes                    |
| get_raw_abi                     |  none                         | yes                    |
| get_table_rows                  |  none                         | yes                    |
| get_table_by_scope              |  none                         | yes                    |
| get_currency_balance            |  none                         | yes                    |
| get_currency_stats              |  none                         | yes                    |
| get_producers                   |  none                         | yes                    |
| get_producer_schedule           |  none                         | yes                    |
| get_scheduled_transactions      |  none                         | yes                    |
| abi_json_to_bin                 |  none                         | yes                    |
| abi_bin_to_json                 |  none                         | yes                    |
| get_required_keys               |  none                         | yes                    |
| get_transaction_id              |  none                         | yes                    |
| get_consensus_parameters        |  none                         | yes                    |
| get_accounts_by_authorizers     |  none                         | yes                    |
| get_transaction_status          |  none                         | yes                    |
| send_read_only_transaction      |  none                         | yes                    |
| compute_transaction             |  main thread: temp change chainbase      | no          |
| push_block                      |  main thread: chainbase, forkdb, producer, controller | no |
| push_transaction                |  main thread: chainbase, forkdb, producer, controller | no |
| push_transactions               |  main thread: chainbase, forkdb, producer, controller | no |
| send_transaction                |  main thread: chainbase, forkdb, producer, controller | no |
| send_transaction2               |  main thread: chainbase, forkdb, producer, controller | no |


#### Producer APIs
Producer APIs do not mutate states. They are received on the HTTP thread, processed on the main thread, and responses are sent on the HTTP thread.

| API                         |  Data modified                | read-only thread safe  |
|-----------------------------|-------------------------------|------------------------|
| pause                       |  main thread: producer's \_pause_production  | yes     |
| resume                      |  main thread: producer's \_pause_production  | yes     |
| paused                      |                               | yes                    |
| get_runtime_options         |                               | yes                    |
| update_runtime_options      |  main thread: producer plugin and controller configs | no (configs used by trx processing|
| add_greylist_accounts       |  main thread: controller resource_greylist | yes (not used in read-only trx handling. only used by max_bandwidth_billed_accounts_can_pay) |
| remove_greylist_accounts    |  main thread: controller resource_greylist | yes       |
| get_greylist                |                               | yes                    |
| get_whitelist_blacklist     |                               | yes                    |
| set_whitelist_blacklist     |  main thread:controller blacklist, whitelist|  no      |
| get_integrity_hash          |                               | yes                    |
| create_snapshot             |                               | yes                    |
| get_scheduled_protocol_feature_activations |                | yes                    |
| schedule_protocol_feature_activations |                     | no                     |
| get_supported_protocol_features |                           | yes                    |
| get_account_ram_corrections |                               | yes                    |
| get_unapplied_transactions  |                               | yes                    |

#### Net APIs
Net APIs do not mutate states. They are received on the HTTP thread, processed on the main thread, and responses are sent on the HTTP thread.

| API                         |  Data modified                  | read-only thread safe  |
|-----------------------------|---------------------------------|------------------------|
| connect                     |  main thread: net::connections  | yes                    |
| disconnect                  |  main thread: net::connections  | yes                    |
| status                      |  none                           | yes                    |
| connections                 |  none                           | yes                    |


#### Trace APIs
 API                          |  Data modified                  | read-only thread safe  |
|-----------------------------|---------------------------------|------------------------|
| get_block                   |  none                           | yes                    |
| get_transaction_trace       |  none                           | yes                    |

#### DB Size API
| API                         |  Data modified                  | read-only thread safe  |
|-----------------------------|---------------------------------|------------------------|
| get                         |  none  | yes                    |                        |

### Net Messages

Net messages can be classified into sync and non-sync:
- Non-sync messages do not modify states.
- Sync messages are signed_block, packed_transaction. They may modify states.

| Message                     | Data modified                   | threads involved | read-only thread safe |
|-----------------------------|---------------------------------|-----------------------------|-----------------------| 
| handshake                   | none                            | net, main (handshake check) | yes        |
| go_away                     | none                            | net              | yes                   |
| time                        | none                            | net              | yes                   |
| notice                      | none                            | net              | yes                   |
| request                     | none                            | net, main (controller::fetch_block_by_id()) | yes   |
| sync_request                | none                            | net, main (controller::fetch_block_by_number()) | yes |
| packed_transaction          | chainbase, forkdb, producer, net, controller | net, producer, main | no    |
| signed_block                | chainbase, forkdb, producer, net, controller | net, producer, main | no    |

## Design Decisions
The node toggles between `write` and `read` windows. In `write` window, the node handles requests normally, except queuing read-only transactions . In `read` window, the node runs read-only transactions in the read-only thread pool, and runs read-only thread safe requests in other threads.

### When to Switch Windows?
To facilitate window switching, configurable options write window time, and read window time, and number of read-only transaction threshold are provided. 
- From `write` to `read`: Switch at the end of the `write` window and read-only transaction queue has entries, or whenever the number of outstanding read-only transactions reaches the threshold.
- From `read` to `write`: Switch at the end of read-only window or whenever read-only queue becomes empty. The threshold option helps to make sure not too many read-only transactions are held if last read-only window exits before its end time.

### How to Handle Write and sync Requests in `read-only` Window?
During `read` window, new read-only thread non-safe requests keep coming in. How to handle them?
- Drop the requests. This in not acceptable as it changes the behavior of the node.
- Queue the requests. Modify appbase priority queue by adding a request type `thread-safe` and `not-thread-safe`. In `write` window, everything works as it is now. In `read` window, only `thread-safe` requests are dequeued and processed. This avoids introducing new queues and keeps the order of the request.
- Repost to the main thread. All write requests are handled by the main thread for some period of time. In the functor of the post to the main thread, if node is in `read-only` window, re-post it to the main thread. Care must be taken to prevent infinite loops. Should the order of write requests be kept?


### Read-only Transaction Queue
```c++
struct read_only_transaction {
   send_read_only_transaction_params params;
   next_func_t  next;
};

std::queue <read_only_transaction> read_only_trx_queue;
```
Protected by a mutex. A transaction failed due to read window deadline but not transaction deadline will be put back into the queue for next round.

### Configuration Options

- `read-only-transaction-num-threads`: the number of threads in read-only transaction thread pool. Default to `0`. If `0`, multi-threaded execution is not used; read-only transactions are executed single threaded
- `read-write-window-time`: time in milliseconds the read-write window runs. Default to 500 milliseconds
- `read-only-window-time`: time in milliseconds the read-only window runs. Must be equal to or greater than `max-read-only-transaction-time`. Default to 200 milliseconds
- `read-only-transaction-threshold`: when the number of queued read-only transactions reaches the threshold, node switches to read-only mode, even it is before the end of read-write window. Default to 0. If 0, this option is not used.
- `read-only-window-margin`: when the time remains in the read-only window is less than the margin, no new transactions are scheduled in read-only threads. Default to 5 milliseconds.
- `max-read-only-transaction-time`: time in milliseconds a read-only transaction can execute before being considered invalid. Default to 150 milliseconds. This option has already been implemented by #558


## Thread Safety

- Safety between read-only transaction threads and other `nodeos` threads
   - _main_ thread: The `main` thread only performs read-only requests. It does not have any conflicts with read-only threads.
   - _chain_ thread: `chain` threads are used in `apply_block`, `log_irreversible`, `finalize_block`,  `create_block_state_future`. Those do not run while in read-only window.
   - _net_ thread: Non-read requests are reposted to (held by) the main thread. No conflicts with read-only transaction execution.
   - _http_ thread: It is used to receive requests and send back responses. No conflicts with read-only transaction execution.
   - _prod_ thread: It is used in on_incoming_transaction_async, which is not running in read-only window
   - _resource monitor_ thread: Resource monitor does not have any conflicts with any transaction execution.
- Safety between read-only transaction threads: no writes are made into `chainbase` and global states when a read-only transaction is executed. This is achieved by PR #558

## Tests

- number of read-only threads is 0
- number of read-only threads is 1
- number of read-only transactions greater than number of threads
- `read-write-window-time` test
- `read-only-transaction-window-time` test
- `read-only-transaction-threshold` test
- `read-only-window-margin` test
- read-only transactions are processed within one read window
- read-only transactions are processed in multiple read windows
- make other RPC requests while read-only transactions are executed
