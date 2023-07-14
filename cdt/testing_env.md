## Testing Environment
The goal of this project is to create a testing environment that leverages and extends `antler-run`.


### `antler-run` and CDT Additions
- Extending/modifying the 'native' testing framework in CDT for this new system.
- Creating a set of options for `antler-run` to load 'test' shared objects.
- A new framework for 'integration' testing will be created.


#### Two Types of Testing
The system will allow for both unit testing of smart contracts and integration testing of smart contracts.  In future releases we might support functional testing and component testing of DApps (hopefully with a cross effort with Eric's team).

#### Unit Testing
CDT has had a unit testing framework available for almost 4 years at this point, it has some limitations and as such that is where this new environment will mitigate those limitations.  Before, host functions were emulated in the framework for some like multi-index, or they were left blank for devs to supply their own stubbed implementations.

Since we will have the host functions from nodeos available to the system we will use those as the default implementations.

The same macros for the native testing framework will exist, with the extension of a few:
- `CALL_ACTION`: This will wrap over the instantiation of the contract class and call into the action.
- `CHECK_ACTION`: This will do the same as above but given a test value to check against the return value of the action.
- `CLEAR_STATE`: This will purge all nodeos state associated with a test run and allow for multiple tests per run without the need to ensure the tests are idempotent.

#### Execution in Nodeos
A no-op contract will be set to a 'testing' account, and then registered like a debug contract.  `antler-run` for running the test will send an action to that account which will cause the environment to direct to the unit test shared object.

For `CLEAR_STATE` we will send a return control flow and allow it to produce the block, since the action is really a no-op and we purge all state from the debug execution, we will be back at a clean state.


### `antler-run` Extensions for Testing
We will need to extend antler-run in a few ways, one is the ability to load a 'test' object, this will be either a unit-test object or an integration test object.  The unit-test object will be loaded in the environment as a smart contract, much the same as regular smart contracts for debugging. For integration test objects we will have a second loader for those to allow for the loading of any smart contract objects that need to be loaded for debugging a smart contract via integration testing.

For `unit-tests` we will compile those with CDT the same as we do for debug smart contracts, for integration tests these will be compiled via GCC or Clang on the users computer.

### Integration Testing
We will supply a set of abstractions for creation of logical 'chains', creating accounts, setting code/abi to accounts, calling actions and creating transactions.

```C++
struct account {
  name acnt_name;
  bytes pubkey;
  bytes privkey;
  bytes contract;
  string abi;
};

struct action {
  name acnt_name;
  name act_name;
  bytes payload;
};

struct transaction {
  std::vector<action> actions;
  std::set<bytes>     signatures;
  bytes               extensions;
};

class chain {
  chain(...); // constructors for options in the chain, or a config.ini
  void create_account(...); // create a single account on the chain with optional keys
  void create_accounts(...); // create a set of accounts
  account get_account(...); // get an account object via a LEAP name
  void set_code(...);
  void set_abi(...);
  template <typename Result>
  Result call_action(...); // call an action on this chain with an account, action name and payload
  action create_action(...); // create an action object via an account name, action name and payload
  transaction create_transaction(...); // create a transaction object to push to the chain from a list of actions and an optional signer
  void push_transaction(...);
  block get_block(...);
  template <typename Result>
  Result get_table(...); // get table data from the chain
  
  void clear(); // clears the state of the chain
};

```

The testing framework will standup nodeos nodes and connect to them for each instantiated 'chain'.  We will treat nodeos as a black box and only use the regular P2P and http RPC and the new debug plugin RPC to facilitate its needs.

Example:

`chain` will generate a node and set its config and data dir to a temporary test directory and start the node.  `create_account` will then call the regular RPC to create a new account on this chain. `set_code` and `set_abi` to set the contract for the new account.  We then call `call_action` on the newly set contract and check the result matches what we expect. We view the tables of the contract account to ensure they match what we expect. `chain` goes out of scope and we destroy the temporary node.

### Testing Frameworks
We will not be prescriptive to what high level framework is used, we will have examples of how to use Catch2 for writing integration tests which could be used as reference for Boost or other testing frameworks.