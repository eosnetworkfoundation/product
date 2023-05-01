# Proposal for Nodeos Endpoint IP and Port Configuration
This is a proposal to allow setting the IP and port for each endpoint provided by nodeos independently, i.e. address [IS #965](https://github.com/AntelopeIO/leap/issues/965).
## API category classification

Although [IS #965] (https://github.com/AntelopeIO/leap/issues/965) mentioned the *endpoint* as the unit of address configuration, if we allow each endpoint to be configurable, it
would be too fine-grained and verbose to use. Yet we feel that granularity at the plugin level might not meet user expectations. Therefore, we introduce a category level as the unit of server address configuration. The following are our proposed category classifications.

### chain_ro
  - v1/chain/get_activated_protocol_features
  - v1/chain/get_block_info
  - v1/chain/get_block_header_state
  - v1/chain/get_account
  - v1/chain/get_code
  - v1/chain/get_code_hash
  - v1/chain/get_consensus_parameters
  - v1/chain/get_abi
  - v1/chain/get_raw_code_and_abi
  - v1/chain/get_raw_abi
  - v1/chain/get_table_rows
  - v1/chain/get_table_by_scope
  - v1/chain/get_currency_balance
  - v1/chain/get_currency_stats
  - v1/chain/get_producers
  - v1/chain/get_producer_schedule
  - v1/chain/get_scheduled_transactions
  - v1/chain/get_required_keys
  - v1/chain/get_transaction_id
  - v1/chain/send_read_only_transaction
  - v1/chain/get_accounts_by_authorizers
  - v1/chain/get_raw_block
  - v1/chain/get_block
  - v1/chain/get_block_header
  - v1/chain/get_transaction_status

### chain_rw
  - v1/chain/compute_transaction
  - v1/chain/push_transaction
  - v1/chain/push_transactions
  - v1/chain/send_transaction
  - v1/chain/send_transaction2

### db_size
  - v1/db_size/get
### net_ro
  - v1/net/status
  - v1/net/connections

### net_rw
  - v1/net/connect
  - v1/net/disconnect

### producer_ro
  - v1/producer/paused
  - v1/producer/get_runtime_options
  - v1/producer/get_greylist
  - v1/producer/get_whitelist_blacklist
  - v1/producer/get_scheduled_protocol_feature_activations
  - v1/producer/get_supported_protocol_features
  - v1/producer/get_account_ram_corrections
  - v1/producer/get_unapplied_transactions

### producer_rw
  - v1/producer/pause
  - v1/producer/resume
  - v1/producer/update_runtime_options
  - v1/producer/add_greylist_accounts
  - v1/producer/remove_greylist_accounts
  - v1/producer/set_whitelist_blacklist
  - v1/producer/schedule_protocol_feature_activations
  - v1/producer/get_integrity_hash

### snapshot
  - v1/producer/get_snapshot_requests
  - v1/producer/create_snapshot
  - v1/producer/schedule_snapshot
  - v1/producer/unschedule_snapshot

### trace_api
  - v1/trace_api/get_block
  - v1/trace_api/get_transaction_trace

### prometheus
  - v1/prometheus/metrics

### node
  - v1/node/get_supported_apis
  - v1/chain/get_info

## User configuration knob
A single `http-category-address` can be used to configure all addresses in command line and ini file. The option can be used multiple times as needed.

```config.ini
http-server-address   = http-category-address
http-category-address = chain_ro,127.0.0.1:8081
http-category-address = chain_ro,localhost:9090
http-category-address = chain_rw,[::1]:8082
http-category-address = net_ro,127.0.0.1:8081
http-category-address = net_rw,[::]:8083 
http-category-address = producer_ro,/tmp/absolute_unix_path.sock
http-category-address = producer_rw,./relative/unix_path.sock
http-category-address = snapshot,./relative/unix_path.sock
```

A single `hostname:port` specification can be used by multiple categories, as of the case for `127.0.0.1:8081` in above example. However, two specifications having the same port with different hostname strings are always considered as configuration error regardless whether they can be resolved into the same set of IP addresses. For example, the following configuration are invalid.

```
http-category-address = chain_ro,127.0.0.1:8081
http-category-address = chain_rw,localhost:8081 # error, 127.0.0.1 and localhost are different
```

For the node category, it is NOT user configurable because it will be available for all listened http addresses.

For backward compatibility, the HTTP category facility has to be opted in by specifying `http-server-address = http-category-address` explicitly. 
When `http-server-address = http-category-address`, all the unspecified categories are considered disabled.  In addition, `http-server-address = http-category-address`
connot be specified together with a non-empty `unix-socket-path`. 


All existing configuration options in the http plugin apply to all handlers for all addresses; this includes the `http-threads` option. This means all handlers share the same thread pool.

## API registration from other plugins

Existing API registration method in http_plugin
```
using url_path_t = std::string;
using request_body_t = std::string;
using url_handler = std::function<void(url_path_t&&, request_body_t&&, url_response_callback&&)>;
using api_entry = std::pair<url_path_t, url_handler>;
class http_plugin : public appbase::plugin<http_plugin>
 {
    public:

      void add_handler(const string& url, const url_handler&, appbase::exec_queue q, int priority = appbase::priority::medium_low, http_content_type content_type = http_content_type::json);
      void add_api(const api_description& api, appbase::exec_queue q, int priority = appbase::priority::medium_low, http_content_type content_type = http_content_type::json) {
         for (const auto& call : api)
            add_handler(call.first, call.second, q, priority, content_type);
      }

      void add_async_handler(const string& url, const url_handler& handler, http_content_type content_type = http_content_type::json);
      void add_async_api(const api_description& api, http_content_type content_type = http_content_type::json) {
         for (const auto& call : api)
            add_async_handler(call.first, call.second, content_type);
      }
}
```

It will be converted as follows in this proposal
```
   using server_address_t = std::string;
   using url_path_t = std::string;
   using request_body_t = std::string;
   using url_handler = std::function<void(const server_address_t&, url_path_t&&, request_body_t&&, url_response_callback&&)>;

   enum class api_category {
     node,
     get_info,
     chain_ro,
     chain_rw,
     net_ro,
     net_rw,
     producer_ro,
     producer_rw,
     snapshot,
     trace_api,
     prometheus,
   };

   struct api_entry {
     api_category        category;
     std::string         server_address; // empty value mean the API is available for all listened addresses, NOT DISABLE
     std::string         path;
     url_handler         handler;
     appbase::exec_queue q;
     int                 priority = appbase::priority::medium_low
     http_content_type   content_type = http_content_type::json;
   };

   class http_plugin : public appbase::plugin<http_plugin> {
     public:
       void add_sync_api(api_entry&& api);
       void add_async_api(api_entry&& api);
   };
```

## server address format
* server address can be in one of the following formats
  - ipv4_address:port like 127.0.0.1:8080
  - ipv6_address:port like [2001:db8:3c4d:15::1a2f:1a2b]:8080
  - hostname:port like my.domain.com:8080
  - unix socket path (must starts with '/' or './')

* In the case of the hostname, the IP address is resolved by ARP, *ALL* resolved addresses will be listened to.
* To listen to all interfaces, just use `0.0.0.0:8080`, `[::]:8080` or `:8080`.

### How to handle IPv4-mapped IPv6?
There are a couple of options:
  - Always use system default. That is, on Linux where IPv4-mapped IPv6 is enabled by default, `[::]:8080` will listen to both ipv4 and ipv6. 
  - Always disable IPv4-mapped IPv6. That is, `[::]:8080` will only listen to ipv6; `:8080` will listen on both ipv4 and ipv6.
  - Use system default and provide option to be more specific, like `[::],ip-v6-only=on` or `[::],ip-v6-only=off`.

Apache server adopts the first option and provides a global "--enable-v4-mapped" option such that the behavior on platform where IPv4-mapped IPv6 is disabled by default can match others.

Ngnix, on the other hand, provides `ip-v6-only=on` and `ip-v6-only=off` to be specific, yet it uses `ip-v6-only=off` by default regardless of the platform it runs on since version 1.3.4.  
  