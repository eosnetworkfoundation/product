# Introduction
[IS 299](https://github.com/AntelopeIO/leap/issues/299) suggested using `boost JSON` to replace our own parser in `fc/io/json.cpp`; however, we believe it requires more discussion about our end goal before working on it.
In the leap code base, `fc::variant` acts as the conduit for converting between JOSN, serialized binary format, and in-memory C++ struct object representation. For example, if we want the binary representation of an object in JSON format, we need to convert it to fc::variant and then use abi serializer to get the end result. Nevertheless, `fc::variant` representation is rarely what is actually needed; it is just an intermediary for the conversion we actually want. In our opinion, the long-term goal should be to remove the usage of `fc::variant` from our code base. If that is the case, we should adopt `abioes` in our codebase rather than simply replace the JSON parser for `fc::variant`.

# Preliminary Performance Analysis between `boost JSON` and `RapidJSON` used by `abieos`:
In the official Boost JSON documentation, it claims to be more than twice as fast as `RapidJSON` in certain cases, but there are still some cases where `RapidJSON` is faster than `boost JSON`. However, it is based only on the DOM parser because boost JSON only provides the DOM parser. In abieos, it only uses the SAX parser from `RapidJSON` which is certainly faster than the DOM parser. Given that `boost JSON` cannot beat RapidJSON's DOM parser in all cases, it is doubtful that it can be better than `abieos` which uses the SAX parser.
Furthermore, to use `boost JSON`, JSON strings must be parsed into boost::json::value objects and then converted into `fc::variant` objects because `boost::json::value` objects are not useful for other parts of the system. That means we need to take the conversion from `boost::json::value` to `fc::variant` into account. It is doubtful how much performance can be improved with this effort.

## Pros and Cons for abieos
### Pros
- `abieos` supports direct conversion between JOSN, serialized binary format, and in-memory C++ struct object representation without intermediaries.
- JSON parser may be better than `boost JSON`.
- Further reducing fc library dependency.
- Macros used by abieos for reflection are very similar to the existing FC reflection facility, it should be straightforward to change.
### Cons
- It could cause the change of chainbase memory layout; therefore, this should only be done at the time we decide to break the chainbase memory layout compatibility.