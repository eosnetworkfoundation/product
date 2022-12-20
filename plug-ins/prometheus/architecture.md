# Prometheus Plugin Architecture

This document describe the architecture for the Prometheus Plugin (eosio::prometheus_plugin).  The Prometheus Plugin instruments nodes in order to allow
data to be collected from [Prometheus](https://prometheus.io/).  The requirements for this plugin are described in this [Github issue](https://github.com/eosnetworkfoundation/mandel/issues/67).

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

A new Prometheus plugin will be created.  This plugin will interface with other plugins (chain_plugin, producer_plugin, http_plugin) to supply the data to be exposed.  Instrumentation will be need to be added in the plugin to capture some of these statistics. 
A metrics() call will be added to plugins to get their specific stats (in a non-prometheus dependent fashion).

## Instrumentation of plugins

Each of the various plugins will be instrumented to collect various runtime metrics.  Each of the plugins will have a a plugin_metrics struct that is accessed via a metrics() call which returns a reference to the individualized plugin_metrics. 
The plugin_metrics will contain a set of member runtime_metrics which consist of a type {gauge or counter}, a name and a label.  In addition to the member metrics, a vector of metrics will be maintained for the use of the prometheus plugin. None of the non-prometheus plugins will have any dependency on the prometheus-cpp library. 
Synchronization will be managed through the use of std::atomic.  This will ensure that all metric values are in a coherent state.  It does not ensure that all of the metric values represent a coherent 'snapshot' of a point (eg. it is possible that not all producer plugin values represent the state at a specific block).

An initial phase of instrumentation will allow for the evaluation of the usability of the instrumentation when viewed from Prometheus.  The initial instrumentation will not include any per account or per contract metrics.  Additionally, the ability to expire metrics will not be included in the initial phase.  The following metrics will be collected for the initial phase of instrumentation

## Net Plugin
* gauge - number of clients
* gauge - number of peers
 
## Producer Plugin

Producer plugin metrics will be gathered at the start of each block.

* gauge - unapplied transaction queue sizes
* counter - blacklisted transactions count
* counter - blocks produced
* counter - transactions produced
* gauge - last irreversible
* gauge - current block num
* gauge - subjective billing account size
* gauge - subjective billing block size
* gauge - scheduled transactions
 
## Chain Plugin
* counter - snapshot rows processed
* counter - replay block status 

## Subsequent Instrumentation
After the initial evaluation the following metrics will be added:

* gauge - unapplied transaction queue
  * byte size
  * speculative CPU time
* counter - blacklisted transactions
  * by account
  * by key count
  * by contract count
* counter - number of forks
* counter - number of unapplied blocks
* counter - number of transactions received by peer (including dupes) 
* counter - number of transactions received by api (including dupes)
* counter - Number of unique transactions received by peer 
* counter - Number of unique transactions received by api 
* gauge - number of new connections 
* gauge - number of connections on wrong chain
* counter - uptime
* gauge - cpu usage by thread
* gauge - disk space used by volume (blocks, ship, state, trace)
* gauge - disk space available

## Prometheus Endpoint
The prometheus_plugin will have one endpoint (/metrics) to support the pull functionality. The pull endpoint will be implemented on top of nodeos http_plugin, rather than civetweb.
Data will be exported using the text-based exposition format.
