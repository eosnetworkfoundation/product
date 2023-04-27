# Replace some JSON parser usages in leap with abieos

## Introduction
[IS 299](https://github.com/AntelopeIO/leap/issues/299) suggested using `boost JSON` to replace our own parser in `fc/io/json.cpp`; however, we believe `abieos` is a more appropriate alternative.

In the leap codebase, there are a lot of places which convert JSON into their C++ aggregate representation; more specifically, we refer to the code like [`fc::json::from_string(body).as<T>()`](https://github.com/AntelopeIO/leap/blob/83be35f7131cad8e4fcaa5faad707b07d5804ea6/plugins/http_plugin/include/eosio/http_plugin/http_plugin.hpp#L259) in chain, net, producer and http plugins. The code parses a JSON string into `fc::variant` and then converts it to C++ aggregate type via `fc` reflection facilities. However, because the `fc` JSON is known to be inefficient, there is a desire to replace it with a better implementation. `boost JSON` was proposed as the replacement in [IS 299](https://github.com/AntelopeIO/leap/issues/299). We don't think it is a good choice because that involves parsing JSON into `boost::json::value` and then converting the `boost::json::value` to C++ aggregates while there's no existing code to do this kind conversion out of the box. 

On the other hand, `abieos` already provides the capability to convert JSON into C++ aggregate out of the box. Therefore, we think it's better to adopt `abieos`.

## Some other considerations

### ABIEOS  Refactor
* During our discussion, there is a desire to separate some auxiliary types, such as `eosio::asset` or `eosio::signature`, out of the core library. We believe it is doable as part of the effort. Due to the consideration that `abieos` has been used by some third party developers, we propose to create a new library called `abieos-core` which contains only the core serialization part without those auxiliary types and the `abieos` library will subsequently depend on the new `abieos-core` library.

* Change the `eosio` namespace in `abieos`: the usage of `eosio` namespace in `abieos` can cause name collisions once it integrates with leap. We propose to change it to `abieos` instead. For backward compatibility, third party developers may need to use a namespace alias explicitly or use some MACRO to turn off the namespace alias on demand. 

### Entirely removing the fc::variant in leap codebase
There are discussions to remove `fc::variant` from our codebase. We believe it is our long term goal, yet premature to do that at the moment.
Based on our analysis, `fc::variant` is used in the following areas of our codebase:

* logging: there are discussions to replace fc logging library with `fmt` or `spdlog`; therefore, `fc::variant` removal in this context should be part of that effort.
* abi_serializer: `abieos` also provides abi serialization functionality which we believe can be a follow-on task after the replacement of `fc::json::from_string(body).as<T>()` pattern.

* unit tests: there are a lot of places in our unit tests which use `fc::mutable_variant_object` in conjunction with `abi_serializer`. We believe most of them can be replaced with C++ aggregate types with abieos ABI serialization. However, we consider this is a very low priority task because performance is not an issue here.

* `cleos` HTTP request parsing and response formatting: due to the way `cleos` is written, removing `fc::variant` in `cleos` is not trivial because there is a direct counterpart of `fc::variant` in `abieos`. We still think this is doable, the question is whether it is worthwhile to spend time on the part where performance is not critical.
