# Performance Testing Harness Stage 2 Planning

Roadmap for the development of a **Performance Testing Harness** -- Stage 2.

Requirements are being curated for the development of a performance testing and evaluation harness here: [Performance Testing Harness Requirements](https://github.com/eosnetworkfoundation/product/tree/main/performance-harness/proposals)


## Stage 2


### Expand Transaction generator

-   Configurable transaction type to send via JSON transaction description
-   New Transaction Types supported
    -   Account Creation
    -   `eos-mechanics` transactions (in particular `cpu()` and potentially `ram()`)
    -   EVM Transactions

### Expand Run Harness Config Options

-   Add support for configuring:
    -   chain threads
    -   net threads
    -   producer threads
-   Investigate potential default arguments that are not showing up in configuration reporting
    -   Look at default plugins used (`chain_plugin`, `net_plugin` , `producer_plugin`, `http_plugin`)
    -   What arguments do they support?
    -   Which are pertinent to performance tuning?
-   Update `Cluster.py` to allow running producer nodes in heap mode vs. mapped mode
-   Allow specifying contract to use during testing
    -   Add support for `eos-mechanics` contract

### Multi-producer & "relay" or "canary" node configuration support

-   Add configurable network topology based on public networks
    -   multiple producer nodes, each accessed through a dedicated relay/canary node
    -   all relay nodes connected in mesh
-   Coordinate TPS generation across multi-nodes, directing transaction traffic to relay nodes
-   Dynamic configuration support for multi-node setup
    -   Relay Nodes Specify
        -   Optimized compiler (oc)
    -   Producers specify
        -   Have to specify each of these pairs because software takes more conservative of the pairs.
            -   last-block-cpu-effort-percent
            -   cpu-effort-percent
            -   last-block-time-offset-us
            -   produce-time-offset-us

### Include configurable "relay" nodes

-   Specific configuration per node type

### Expand reporting to include multiple nodes

-   Number of Forks
-   Number of dropped blocks
    -   Missed blocks
    -   Forked blocks
-   Number of dropped transactions
-   Number of complete production windows
-   Create report for per-transaction metrics over time
    -   cpu, latency, sent time, block inclusion time
-   Investigate graphana (or similar tools) for visualizing metrics
-   Investigate/Identify additional metrics that would be useful to publish

### Begin Design Planning for migration to auto deployment

-   Building architectural/design doc
-   Evaluation of technologies
-   Recommendation for technologies to use
-   Investigate/Create plan for exercising Performance Harness tests against another node network (i.e. Jungle test net)
