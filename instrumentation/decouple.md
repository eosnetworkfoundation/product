# Decouple instrumentation feasiblity
https://github.com/AntelopeIO/leap/issues/1047

## Proposed new integration points
- Event system
    - Generate and signal event if any listeners configured
        - Avoid overhead of creating event if no listeners
    - Avoid additional dynamic memory when generating events
        - Use const char* or enum or pre-allocated descriptors
- ram usage
    - Refactor all chainbase access through a wrapper class
        - Provide each modifier method with meta info needed to create event
        - Side benefit of easier integration of new database
    - Distinguish between user contract data and native data
- Generate an event for every host function call
    - Include all data passed to host function
    - Include all data returned from host function
    - Exclude low-level
        - memcpy, memmove, memcmp, memset
            - CDT is likely to not call these anyway
        - softfloat, compiler_builtins
            - CDT might provide these directly as well
- Lifecycle events
    - Application start/(clean)stop
    - block start/stop
    - Fork switch
        - Include range info
    - [deferred|implicit|input] trx start/stop
        - Emit trace
    - input [cf]_action start/stop
        - Emit trace (debug purposes)
    - inline [cf]_action start/stop
        - Emit trace (debug purposes)
    - notify start/stop
    - create account
    - update permissions
    - feature activation
    - set abi/code
    - eosio::transfer
        - report offical transfers
        - Registered by system contract, use eosio.token to begin with


## Existing DeepMind integration points
- begin/end action `apply_context::exec_one`
- require_recipient `apply_context::require_recipient`
- send[_context_free]_inline `apply_context::execute[_context_free]_inline`
- ram modification (see on_ram_trace section below)
    - deferred trx creation 
        - `apply_context::schedule_deferred_transaction`
        - `transaction_context::schedule_transaction`
    - deferred trx del `apply_context::cancel_deferred_transaction`
    - deferred trx rm (execute) `controller remove_scheduled_transaction`
    - deferred trx ram correction `on_activation<builtin_protocol_feature_t::replace_deferred>`
    - create table `apply_context::find_or_create_table`
    - del table `apply_context::remove_table`
    - db store `apply_context::db_store_i64`
    - db update `apply_context::db_update_i64`
    - db remove `apply_context::db_remove_i64`
    - secondary index add 
        - `apply_context generic_index store`
        - `apply_context generic_index update`
    - secondary index remove 
        - `apply_context generic_index remove`
        - `apply_context generic_index update`
    - secondary index `apply_context generic_index update`
    - charge/refund payer `apply_context::db_update_i64`
    - create native account `controller create_native_account`
    - create account `apply_eosio_newaccount`
    - setcode `apply_eosio_setcode`
    - setabi `apply_eosio_setabi`
    - udpate auth `apply_eosio_updateauth`
    - del auth `apply_eosio_deleteauth`
    - link auth `apply_eosio_linkauth`
    - unlink auth `apply_eosio_unlinkauth`
    - ram usage `resource_limits_manager::add_pending_ram_usage`
- create table `apply_context::find_or_create_table`
- del table `apply_context::remove_table`
- db store `apply_context::db_store_i64`
- db update `apply_context::db_update_i64`
- db remove `apply_context::db_remove_i64`
- create deferred trx `transaction_context::schedule_transaction`
- cancel deferred trx 
    - `apply_context::schedule_deferred_transaction`
    - `cancel_deferred_transaction`
- send deferred trx
    - `apply_context::schedule_deferred_transaction`
- fail deferred trx
    - `controller push_scheduled_transaction`
- onerror - `controller apply_onerror`
- onblock creation `controller get_on_block_transaction`
    - This seems incorrect as this is at the creation of the onblock. The onblock can fail in which case this onblock should not be reported.
- create permission `authorization_manager::create_permission`
- modify permission `authorization_manager::modify_permission`
- remove permission `authorization_manager::remove_permission`
- application start `controller init`
- switch forks `controller maybe_switch_forks`
- applied trx - same as signal `applied_transaction`
- start block - same as signal `block_start`
- accepted block - same as signal `accepted_block`
- trx start `transaction_context::transaction_context`
- trx end `transaction_context::~transaction_context`
- input action `transaction_context::execute_action`
- preactivate feature `controller preactivate_feature`
- activate feature `protocol_feature_manager::activate_feature`
- add_ram_correction `controller add_to_ram_correction`
- init resources `resource_limits_manager::initialize_database`
- new account resource limits `resource_limits_manager::initialize_account`
- update resource limits `resource_limits_manager::set_block_parameters`
- update resource limits state 
    - `resource_limits_manager::process_account_limit_updates`
    - `resource_limits_manager::process_block_usage`
- update account usage `resource_limits_manager::add_transaction_usage`
- set account limits `resource_limits_manager::set_account_limits`


### on_ram_trace
```
apply_context.cpp
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", ptr->id)), "deferred_trx", "cancel", "deferred_trx_cancel");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", gtx.id)), "deferred_trx", "update", "deferred_trx_add");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", gtx.id)), "deferred_trx", "add", "deferred_trx_add");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", gto->id)), "deferred_trx", "cancel", "deferred_trx_cancel");
   dm_logger->on_ram_trace(std::move(event_id), "table", "add", "create_table");
   dm_logger->on_ram_trace(std::move(event_id), "table", "remove", "remove_table");
   dm_logger->on_ram_trace(std::move(event_id), "table_row", "add", "primary_index_add");
   dm_logger->on_ram_trace(std::string(event_id), "table_row", "remove", "primary_index_update_remove_old_payer");
   dm_logger->on_ram_trace(std::move(event_id), "table_row", "add", "primary_index_update_add_new_payer");
   dm_logger->on_ram_trace(std::move(event_id) , "table_row", "update", "primary_index_update");
   dm_logger->on_ram_trace(std::move(event_id), "table_row", "remove", "primary_index_remove");
controller.cpp
   dm_logger->on_ram_trace(RAM_EVENT_ID("${name}", ("name", name)), "account", "add", "newaccount");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", gto.id)), "deferred_trx", "remove", "deferred_trx_removed");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", itr->id._id)), "deferred_trx", "correction", "deferred_trx_ram_correction");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${name}", ("name", create.name)), "account", "add", "newaccount");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${account}", ("account", act.account)), "code", operation, "setcode");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${account}", ("account", act.account)), "abi", operation, "setabi");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", permission->id)), "auth", "update", "updateauth_update");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", p.id)), "auth", "add", "updateauth_create");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", permission.id)), "auth", "remove", "deleteauth");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", l.id)), "auth_link", "add", "linkauth");
   dm_logger->on_ram_trace(RAM_EVENT_ID("${id}", ("id", link->id)), "auth_link", "remove", "unlinkauth");
transaction_context.cpp
   dm_logger->on_ram_trace(std::move(event_id), "deferred_trx", "push", "deferred_trx_pushed");
apply_context.hpp
   dm_logger->on_ram_trace(std::move(event_id), "secondary_index", "add", "secondary_index_add");
   dm_logger->on_ram_trace(std::move(event_id), "secondary_index", "remove", "secondary_index_remove");
   dm_logger->on_ram_trace(std::string(event_id), "secondary_index", "remove", "secondary_index_remove");
   dm_logger->on_ram_trace(std::move(event_id), "secondary_index", "add", "secondary_index_update_add_new_payer");
```                     
