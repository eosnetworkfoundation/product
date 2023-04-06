# Proposal for Nodeos Endpoint IP and Port Configuration
This is a proposal to allow setting the IP and port for each endpoint provided by nodeos independently, i.e. address [IS #965](https://github.com/AntelopeIO/leap/issues/965).
## API category classification

Although [IS #965](https://github.com/AntelopeIO/leap/issues/965) mentioned the *endpoint* as the unit of address configuration; however, if we allow each endpoint to be configurable, it
would be too finer grained and verbose to use. Yet we feel the granularity is on the plugin level might not meet the user expectation. Therefore, we introduce a category level as the unit of server address configuration. The following are our proposed category classification.

### get_info
  - v1/chain/get_info

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
  - v1/chain/compute_transaction
  - v1/chain/get_accounts_by_authorizers
  - v1/chain/get_raw_block
  - v1/chain/get_block
  - v1/chain/get_block_header
  - v1/chain/get_transaction_status

### chain_rw
  - v1/chain/push_transaction
  - v1/chain/push_transactions
  - v1/chain/send_transaction
  - v1/chain/send_transaction2

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
  - v1/producer/get_snapshot_requests
  - v1/producer/get_integrity_hash

### producer_rw
  - v1/producer/pause
  - v1/producer/resume
  - v1/producer/update_runtime_options
  - v1/producer/add_greylist_accounts
  - v1/producer/remove_greylist_accounts
  - v1/producer/set_whitelist_blacklist
  - v1/producer/schedule_protocol_feature_activations

### snapshot
  - v1/producer/create_snapshot
  - v1/producer/schedule_snapshot
  - v1/producer/unschedule_snapshot

### trace_api
  - v1/trace_api/get_block
  - v1/trace_api/get_transaction_trace

### wallet
  - v1/wallet_mgr/set_timeout
  - v1/wallet_mgr/sign_transaction
  - v1/wallet_mgr/sign_digest
  - v1/wallet_mgr/create
  - v1/wallet_mgr/open
  - v1/wallet_mgr/lock_all
  - v1/wallet_mgr/lock
  - v1/wallet_mgr/unlock
  - v1/wallet_mgr/import_key
  - v1/wallet_mgr/remove_key
  - v1/wallet_mgr/create_key
  - v1/wallet_mgr/list_wallets
  - v1/wallet_mgr/list_keys
  - v1/wallet_mgr/get_public_keys

### prometheus
  - v1/prometheus/metrics

### node
  - v1/node/get_supported_apis

## User configuration knob
A single `http-category-address` can be used to configure all addressesTo in command line and ini file. The option can used multiple times as needed.

```config.ini
http-server-address   = 0.0.0.0:8080
http-category-address = chain_ro,127.0.0.1:8081
http-category-address = chain_ro,localhost:9090
http-category-address = chain_rw,[::1]:8082
http-category-address = net_ro,[::1]:8081
http-category-address = producer_ro,/tmp/absolute_unix_path.sock
http-category-address = wallet,./relative/unix_path.sock
http-category-address = snapshot_rw,./relative/unix_path.sock
http-category-address = get_info,http-server-address             # means 0.0.0.0:8080
http-category-address = trace_api,                               # disable
````

The corresponding environment variables to configure the addresses will be something like HTTP_CATEGORY_SERVER_CHAIN_RO, HTTP_CATEGORY_SERVER_NET_RO

For backward compatibility, the default addresses would be "http-server-address" for all catagories which are not explicitly configured.

All existing configuration options in http plugin apply to all handlers for all address; this include `http-threads` option. This means all handlers share the same thread pool.


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
     wallet,
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
* server address can be in one of the following format
  - ipv4_address:port
  - ipv6_address:port
  - hostname:port
  - unix socket path (must starts with '/' or './')
  - http-server-address

* In the case of hostname, the IP address is resolved by ARP, *ALL* resolved address will be listened.
* Whether an IPv4-mapped IPv6 address can be used to handle both IPv4 connections and IPv6 connections is determined by system configuration. Some tool like Apache server has the option `-—enable-v4-mapped` to change the behavior, should we do the same?
* To listen to all interfaces, just use 0.0.0.0:8008 or [::1]:8080, alternative syntax just adds unnecessary complexity.
