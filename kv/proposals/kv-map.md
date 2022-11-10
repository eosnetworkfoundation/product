# Proposal for new map data types for smart contract developers

## Problem

### Opportunity
Currently we do not have simple key value structure for managing database in smart contracts. At the moment this can be achieved with multi_index but with some level of verbocity. This proposal is to create new ordered and unordered map containers with interface similar to corresponding std containers.
### Target audience
- Contract developers
### Strategic alignment
There is a demand for simple key value structure in smart contracts. We can simplyfy smart contract code access to database mimicing std containers interface and step away from clanky lambdas that we need for setting values in `multi_index`. We also expect better performance for those new containers.

## Solution

### Solution name
KV map
### Purpose
Set of map containers for managing chainbase database tables using standard std syntax and improved performance.
### Success definition
Ordered and unordered map containers that implement basic std methods like begin/end, find, insert, square brackets operator, range based loop, etc.
### Implementation details
We are planning to use same database host functions as `multi_index` does (`db_store_i64`, `db_update_i64`, etc.). Internally `unordered_map` won't be traditional hash map but some hybrid of ordered and unordered maps. We are planning to use `uint64` hashes as a keys for entries. Hash will be cross platform solution for prospective usage in other than c++ languages. The purpose behind this is performance since `uint64` index is faster than byte array. There won't be buckets structure where traditional hash maps use more memory. For X enties there will be X table rows. No rehashes. Collision chance expected to be nearly zero since we will use origin 8 byte hashes that are not converted to array index. For ordered map we will introduce new host functions to use byte array as an index. Ordered map will have CPU tradeoff due to byte array index so unordered map is suggested to be used where order is not important.


## Assumptions

### Risks
We are going to implement from scratch map containers so need to have extensive unit tests and ensure performance is adequate

## Functionality

```c++
using my_map_t = eosio::unordered_map<"kvmap"_n, std::string, std::string>;

my_map_t kvmap;
kvmap["key1"] = "val1";
kvmap["key2"] = "val2";
kvmap["key3"] = "val3";

assert(!kvmap.empty());
assert(kvmap.size() == 3);
assert(kvmap.find("key1")->value == "val1");
assert(kvmap.find("key2")->value == "val2");
assert(kvmap.find("key3")->value == "val3");

for (const auto& [key, value] : kvmap) {
    eosio::print_f("% = %", k, v);
}

kvmap.clear();
assert(kvmap.empty());

```