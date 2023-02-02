# Prometheus Plugin Architecture

This document describes the architecture for the Prometheus Plugin (eosio::prometheus_plugin).  The Prometheus Plugin instruments nodes in order to allow
data to be collected for [Prometheus](https://prometheus.io/).  The requirements for this plugin are extracted from this [Github issue](https://github.com/eosnetworkfoundation/mandel/issues/67):

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

# Prometheus Plugin

A new prometheus_plugin will be created.  This plugin will interface with instrumented plugins (chain_plugin, producer_plugin, http_plugin, net_plugin, api plugins...) to supply the data to be exposed.  The various plugins will be need to be modified to capture some of these statistics, aka "instrumented". 
The prometheus_plugin will receive data in a message-based format consisting of a collection of current metrics for the instrumented plugin.  The collection of metrics for a particular plugin may change over time and it will be up to the prometheus_plugin to manage retiring metrics that are no longer relevant.  Each instrumented
plugin will have it's own particular update cycle based on what makes sense for the particular plugin.  During each update cycle, the instrumented plugins will post the collection of metrics to the prometheus_plugin (if enabled).  The prometheus_plugin will
update its internal data model as these messages are received.

## Prometheus Endpoint
The prometheus_plugin will have one endpoint (/metrics) to support the pull functionality. The pull endpoint will be implemented on top of nodeos http_plugin, rather than civetweb.
Data will be exported using the prometheus text-based exposition format.
The http endpoint for the prometheus_plugin will utilize the prometheus strand to send client updates.  The plugin will receive requests on an http thread, which will immediately post a lambda to the prometheus strand.  The request will pull the
updated metric data from the prometheus data model and use the prometheus-cpp library to perform the serialization.  Requests will be bound by a timeout of 30ms.

## Instrumentation of plugins

Each of the instrumented plugins will have a register_metrics_listener() method to register a std::function that takes a vector of runtime_metrics.  Instrumented plugins will each maintain a specialized plugin_metrics structure (see below).  The plugin will pass the register_listener() method to the plugin_metrics struct. The plugin_metrics struct will manage the update cycle
to enforce a clean 'snapshot' view of all the metric data for a plugin.

A plugin_metrics base class will be provided for plugins to support collecting/publishing metrics to the prometheus_plugin (or potentially some other metrics publisher in the future). Each plugin will extend the base class and include runtime_metrics as members of the base class.
Each plugin implements a virtual function metrics(), which will collect all relevant metrics and insert them into a vector for processing by the prometheus_plugin.

The register_listener() method on plugin_metrics will be used by the prometheus plugin to register a lambda to perform the metrics update.
The lambda will post to a strand, backed by a named_thread pool (consisting of a single thread).  The lambda posted to the strand will perform the metrics update inside the prometheus_plugin.
Each plugin will have its own update cycle - for instance the producer_plugin will update on each block.  The instrumented plugins will report their metrics update via the post_metrics() method on the plugin_metrics object.
The post_metrics method will check the should_post() method to verify that it should actually update the prometheus_plugin by calling should_post().  The should_post method will check to see if there is a registered listener and verify that enough time has elapsed since
the last post.  The default throttling will be to enforce a maximum of 4Hz (250ms between calls).  Based on settings, an update rate of up to 10Hz (100ms between calls) will be supported.

```C++
struct plugin_metrics {
   virtual vector<runtime_metric> metrics()=0;
   bool should_post();
   bool post_metrics();
   void register_listener(metrics_listener listener);
};
```

# Initial Intrumentation
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
 
# Subsequent Instrumentation
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
* gauge - number of transactions in process 
