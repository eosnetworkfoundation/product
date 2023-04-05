# Performance Testing Harness Phase 3 Planning

Roadmap for the development of a **Performance Testing Harness** -- Phase 3.

Requirements are being curated for the development of a performance testing and evaluation harness here: [Performance Testing Harness Requirements & Planning & Staging](https://github.com/eosnetworkfoundation/product/tree/main/performance-harness/proposals)


## Phase 3

### Operational Modes

-   Develop documentation around operational modes
-   For each operational mode identified:
    -   Create reproducible test setup
        -   Feature set enabled/disabled for specific operational mode
        -   Type of operations required to exercise features for operational mode
        -   Easy to run test of operational mode
    -   Define performance measurement criteria
    -   Measure performance of operational mode
    -   Report on performance for operational mode
    -   Isolate operations in operational mode for finest granularity testing and performance measurement
-   Initial Operational Modes to support
    -   Block Producer
        -   How many transactions processed into a block
        -   What is the latency for transactions getting into a block
        -   Time to produce a block (wall clock time)
    -   API Node (HTTP Node)
        -   How many transactions can be pushed
        -   How many read-only transactions can be processed
        -   How many reads are possible (i.e. get table calls)
    -   Relay Node
        -   Running with optimized compiler
        -   How many transactions can it relay
        -       May require new instrumentation as this measurement doesn't currently exist
        -   Validation time for a block of given size (compared to production time of that or similarly sized block)
    -   Canary Node
        -   Similar to Relay Node, but not running optimized compiler
    -   SHiP Node
        -   Not lagging behind, steady state behind head (~1 block)
        -   Are they able to keep up with X number of clients
    -   Trace API Node
        -   Not lagging behind, steady state behind head (~1 block)
        -   Are they able to keep up with X number of clients


### Engage with ENF to plot course on the following topics

-   Engage with ENF to identify:
    -   Individuals at ENF to engage with
    -   How to gather and persist data
    -   What data to gather and persist
    -   How to incorporate Performance Harness into CI/CD or auto-deployment
    -   Initial control surface (simple UI?)
    -   Visualizations desired (tool/technology, graphs, etc.)






