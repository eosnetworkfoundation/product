# Proposal for new map data types for smart contract developers

## Problem

### Opportunity
Currently we do not have simple key value structure for managing database in smart contracts. At the moment this can be achieved with multi_index but with some level of verbocity. This proposal is to create new ordered and unordered map containers with interface similar to corresponding std containers.
### Target audience
- Contract developers
### Strategic alignment
There is a demand for simple key value structure in smart contracts. We can simplyfy smart contract code access to database mimicing std containers interface and step away from clanky lambdas that we need for setting values in multi_index

## Solution

### Solution name
Kv map
### Purpose
Set of map containers for managing chainbase database tables using standard std syntax
### Success definition
ordered and unordered map containers that implement basic std methods like begin/end, find, insert, square brackets operator, range based loop, etc.

## Assumptions

### Risks
We are going to implement from scratch map container and hash functions so need to have extensive unit tests and ensure performance is adequate

## Functionality

```
using my_map_t = eosio::kv::unordered_map<"kvmap"_n, int, std::string>;

my_map_t kvmap;
kvmap[1] = "str1";
kvmap[2] = "str2";
kvmap[3] = "str3";

assert(!kvmap.empty());
assert(kvmap.size() == 3);
assert(kvmap.find(1)->value == "str1");
assert(kvmap.find(2)->value == "str2");
assert(kvmap.find(3)->value == "str3");

for (const auto& [key, value] : kvmap) {
    eosio::print_f("% = %", k, v);
}

kvmap.clear();
assert(kvmap.empty());

```