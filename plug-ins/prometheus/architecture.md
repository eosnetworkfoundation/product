# Prometheus Plugin Architecture

This document describe the architecture for the Prometheus Plugin (eosio::prometheus_plugin).  The Prometheus Plugin instruments nodes in order to allow
data to be collected from [Prometheus](https://prometheus.io/).  The requirements for this plugin are described in this [Github issue](https://github.com/eosnetworkfoundation/mandel/issues/67).

## Client Library

A C++ client library exists to facilitate instrumenting a program to export data point to Prometheus called [prometheus-cpp](https://github.com/jupp0r/prometheus-cpp).

Unfortunately, that library is somewhat heavy weight and contains dependencies that are not desirable to bring into the leap codebase, the most problematic being Civetweb:


* `civetweb` for [Civetweb](https://github.com/civetweb/civetweb)
* `com_google_googletest` for [Google Test](https://github.com/google/googletest)
* `com_github_google_benchmark` for [Google Benchmark](https://github.com/google/benchmark)
* `com_github_curl` for [curl](https://curl.haxx.se/)
* `net_zlib_zlib` for [zlib](http://www.zlib.net/)

Instead of making prometheus-cpp a submodule of leap, a new library named prometheus will be created to support exporting of data points from nodeos.
Code will be ported over from prometheus-cpp, but converted to be more in keeping with the leap coding style (snake case, .hpp & .cpp).  Substantially all of "core" will be ported 

Instead of relying on civetweb to implement the pull functionality, the Prometheus plugin will implement those functions on top of built-in http
support in nodeos.  

The pull and push functionality will not be pulled into this library, but will be implemented directly in the plugin, due to dependencies on other plugins. 

## Plugin

A new Prometheus plugin will be created.  This plugin will interface with other plugins (chain_plugin, producer_plugin, http_plugin) to supply the data to be exposed.  Instrumentation will be need to be added in the plugin to capture some of these statistics. 
A call will be added to plugins to get their specific stats (in a non-prometheus dependent fashion).

Data from the original github issue (these data points will require curation/prioritization):

Chain Plugin
* what is returned from /v1/chain/get_info (head block number, lib)
* replay status (when starting from snapshot and replaying blocks)

Producer Plugin
* unapplied transaction queue sizes
* blacklisted transactions size
* subjective billing sizes
* scheduled transaction size
* number of forks by producer
* number of unapplied blocks by producer
* number of dropped blocks by producer
* number of missed blocks (missed in a round) by producer
* number of missing producers (missed 12 blocks in a round) by producer
* number of double production (more than 12 blocks in a round) by producer
* average (and last) block arrival time by producer
* number of transactions per block by producer
* amount of blockchain cpu used per block by producer
* number of bad actors which have exceeded their transaction limit (as per the changes introduced in nodeos 2.0.9)

Net Plugin
* number of clients connected (inbound, outbound, failed)

Other
* uptime
* cpu usage by thread
* disk space used, by volume (blocks, ship, state, trace)
* disk space available

All API plugins
* number of api (failed) requests/sec (by request type)
 
Unknown
* number of bytes per block by producer

The plugin will have two endpoints, one for pull 

The pull endpoint will be re-implemented on top of nodeos http_plugin, rather than civetweb and will scrape data as defined in the prometheus_plugin configuration.

The push endpoint will initiate a push to Prometheus Pushgateway as defined in the prometheus_plugin configuration.  The implementation will continue to use libcurl, as that is already a dependency for leap. 
The value of the push endpoint would be for a nodeos instance behind a firewall to be able to send data out. 

Data will be exported using the text-based exposition format.
