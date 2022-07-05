# Performance Testing Harness Requirements

Gathering requirements for a performance testing and evaluation harness.  Requirements will be evaluated, prioritized, and staged into the [*Performance Testing Harness Delivery Staging*](https://github.com/eosnetworkfoundation/product/tree/main/performance-harness/proposals) document.

Additional requirements may be proposed for inclusion in ongoing or future stages.


## Known Spikes


-   SPIKE: Investigate OpenDDS performance framework for ideas
-   SPIKE: Investigate HTTP performance frameworks
-   SPIKE: Investigate distributed performance tracing/bottlenecks (more
    code/finer grained results) e.g. Zipkin


## Cloud Infrastructure


-   Create Cloud Infrastructure
    -   Hosting 
        -   AWS, Google, etc?
        -   Create web server to host/display performance results
    -   Create pinned environments
    -   Deploy cloud environment
    -   Configure (what) tech?
        -   Helm, Ansible, Puppet, Chef, Gitlab CI, Jenkins, Terraform, etc. 
            -   NOTE: some include deployment as their feature set. 


## Reporting


-   Create reporting template
    -   Customize text and charts for various audiences
    -   Produce output in multiple formats
        -   HTML
        -   PDF
        -   CSV
-   Reporting: Deltas and absolute values
    -   Automatically capture and report hardware, OS, compiler,
        software versions
    -   Create reports based on environments


## Multi and Single Node Testing


-   HTTP API RPC endpoint testing
-   Modify on-chain parameters
    -   User configurable (not just hard-coded run types)
        -   Example: Blocklog size
-   Nodeos configurations (customizable at run time)
    -   User configurable (not just hard-coded run types)
        -   Example: EOS WASM VM: Jit, eos-vm-oc

-   Induce artificial network latency between "machine" and/or container instances (network interfaces)
-   Transaction generation
    -   Multi-node send to multiple nodeos
    -   Send to single nodeos
    -   Generate sequential and unsequenced test data.
    -   User-configurable set-up, and tear-down.
-   Transaction Load testing: P2P ingestion (assumes existing P2P and post P2P code rewrite)
    -   P2P transaction generator
        -   Multi-threaded C++ packed transaction generator
        -   Configurable trxs/second with steady state rate
        -   Support various contract action types (listed below)
-   Transaction Load Testing: API ingestion
    -   Likely to require a large number of API nodes to saturate the block production node (see Spike).
-   Contract action types:
    -   User-provided action types (have the ability for the community
        to add them).
    -   Token transfer
    -   Large action
    -   CPU intense action
    -   DB intense action
    -   EOS mechanics:
        [*Aloha EOS Block Producer Benchmarks*](https://www.alohaeos.com/tools/benchmarks)

-   Contract testing scenarios:
    -   Run on empty state
    -   Run on large backlog
    -   Run with backlog pruning
    -   Run with SHiP enabled
    -   Run with trace_api_plugin
-   Run on different states:
    -   Testing large account creation
    -   Testing: Run on top of EOS Mainnet current state
        -   WAX, TLOS, Jungle, etc.
-   Measure latency:
    -   From creation
    -   From consumption in nodeos
    -   From consumption from SHiP
    -   From consumption from trace_api_plugin
-   Measuring throughput TPS
-   Replay
    -   block log
    -   sync
-   Testing/measuring stability:
    -   Number of Forks
    -   Number of dropped blocks
        -   Missed blocks
        -   Forked blocks
    -   Number of dropped transactions
    -   Number of complete windows
    -   Jitter (transactions/block) unstable
    -   TPS Scoring
        -   Standard deviation, mean, average (basic stats)
-   Creating Metrics: Graphs and Charts
    -   Per run
        -   Block size over time
        -   Transaction throughput over time
        -   Transaction latency over time
        -   Each individual transaction CPU time
    -   Historical metrics over time 
        -   Daily, 
        -   Weekly
        -   Monthly
