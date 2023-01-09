# Prometheus Plugin Architecture

This document describes the architecture for the Prometheus Plugin (eosio::prometheus_plugin).  The Prometheus Plugin instruments nodes in order to allow
data to be collected from [Prometheus](https://prometheus.io/).  The requirements for this plugin are extracted from this [Github issue](https://github.com/eosnetworkfoundation/mandel/issues/67):

> For monitoring, nodeos should have a prometheus exporter plugin (that runs on a separate port), which exposes key metrics that would be useful for monitoring.
>
> Metrics such as:
>
> - what is returned from /v1/chain/get_info (head block number, lib)
> - unapplied transaction queue sizes
> - blacklisted transactions size
> - subjective billing sizes
> - scheduled transaction size
> - number of forks by producer
> - number of unapplied blocks by producer
> - number of dropped blocks by producer
> - number of missed blocks (missed in a round) by producer
> - number of missing producers (missed 12 blocks in a round) by producer
> - number of double production (more than 12 blocks in a round) by producer
> - average (and last) block arrival time by producer
> - number of transactions per block by producer
> - amount of blockchain cpu used per block by producer
> - number of bytes per block by producer
> - number of bad actors which have exceeded their transaction limit (as per the changes introduced in nodeos 2.0.9)
> - number of clients connected (inbound, outbound, failed)
> - number of api (failed) requests/sec (by request type)
> - uptime
> - cpu usage by thread
> - disk space used, by volume (blocks, ship, state, trace)
> - disk space available
> - rocksdb related
> - blockvault related (produced block, vs did not produce because other nodeos "won")
> - replay status (when starting from snapshot and replaying blocks)


## Client Library

A C++ client library exists to facilitate instrumenting a program to export data point to Prometheus called [prometheus-cpp](https://github.com/jupp0r/prometheus-cpp).

The library is somewhat heavy weight and contains dependencies that are not desirable to bring into the leap codebase, the most problematic being Civetweb:

* `civetweb` for [Civetweb](https://github.com/civetweb/civetweb)
* `com_google_googletest` for [Google Test](https://github.com/google/googletest)
* `com_github_google_benchmark` for [Google Benchmark](https://github.com/google/benchmark)
* `com_github_curl` for [curl](https://curl.haxx.se/)
* `net_zlib_zlib` for [zlib](http://www.zlib.net/)

The prometheus-cpp library will be brought in as a submodule of leap, but the Prometheus plugin will only use the core part of the library, specifically the part that implements the serialization. 

## Plugin

A new Prometheus plugin will be created.  This plugin will interface with other plugins (chain_plugin, producer_plugin, http_plugin, net_plugin) to supply the data to be exposed.  Instrumentation will be need to be added in the plugin to capture some of these statistics. 
A metrics() call will be added to plugins to get their specific stats (in a non-prometheus dependent fashion).

## Instrumentation of plugins

Each of the various plugins will be instrumented to collect various runtime metrics.  Each of the plugins will have a a plugin_metrics struct that is accessed via a metrics() call which returns a reference to the individualized plugin_metrics. 
The plugin_metrics will contain a set of member runtime_metrics which consist of a type {gauge or counter}, a name and a label.  In addition to the member metrics, a vector of metrics will be maintained for the use of the prometheus plugin. None of the non-prometheus plugins will have any dependency on the prometheus-cpp library. 
Synchronization will be managed through the use of std::atomic.  This will ensure that all metric values are in a coherent state.  It does not ensure that all of the metric values represent a coherent 'snapshot' of a point (eg. it is possible that not all producer plugin values represent the state at a specific block).

## Initial Intrumentation
An initial phase of instrumentation will allow for the evaluation of the usability of the instrumentation when viewed from Prometheus.  The initial instrumentation will not include any per account or per contract metrics.  Additionally, the ability to expire metrics will not be included in the initial phase. To support dynamic metrics, the metric object for each plugin will include an indicator of when the set of metrics has last changed.  The following metrics will be collected for the initial phase of instrumentation:

## Net Plugin
* gauge - number of clients
* gauge - number of peers
 
## Producer Plugin

Producer plugin metrics will be gathered at the start of each block.

* gauge - unapplied transaction queue sizes (number of transactions)
* counter - blacklisted transactions count (total)
* counter - blocks produced
* counter - transactions produced
* gauge - last irreversible
* gauge - current block num
* gauge - subjective billing number of accounts
* gauge - subjective billing number of blocks
* gauge - scheduled transactions
 
## Subsequent Instrumentation
After the initial evaluation the following metrics will be added:

* gauge - unapplied transaction queue
  * byte size
  * speculative CPU time
* counter - number of forks
* counter - number of unapplied blocks
* counter - number of transactions received by peer (including dupes) 
* counter - number of transactions received by api (including dupes)
* counter - Number of unique transactions received by peer 
* counter - Number of unique transactions received by api 
* gauge - number of new connections 
* gauge - number of connections on wrong chain
* counter - uptime in seconds
* gauge - cpu usage by thread
* gauge - disk space in bytes used by volume (blocks, ship, state, trace)
* gauge - disk space in bytes available
* gauge - number of dropped blocks
 
## Prometheus Endpoint
The prometheus_plugin will have one endpoint (/metrics) to support the pull functionality. The pull endpoint will be implemented on top of nodeos http_plugin, rather than civetweb.
Data will be exported using the text-based exposition format.
