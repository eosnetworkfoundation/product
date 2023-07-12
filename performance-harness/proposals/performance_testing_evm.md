# EOS EVM Performance Testing

Roadmap for the development of a performance testing methodology for EOS EVM.

## Extend leap's Test Harness to support EOS EVM Nodes

- Add support for configuring and spinning up an EOS EVM Node
- Add support for initial testing environment requirements
  - contract specifying and loading
    - deploy EOS EVM Contract, swap contract, etc.
  - create swap test tokens
  - account creation
  - initial balance allocation
- Support for standing up an EOS EVM test environment:
  - Single BP Node
  - Http API Node (Optional - can also use P2P)
  - Validation Node (used for querying block and transaction information)

## Extend leap's Performance Harness to support EOS EVM Node testing

- Rework Performance Harness into extensible module
  - Requires moving the bulk of the current performance test and performance test basic into a python module
  - Extend the newly created Performance Harness module for EOS EVM specific configuration and test requirements
  - Create runner script for leap to import the performance harness module and kick off test runs
  - Create similar runner script for EOS EVM to import the extended performance harness module to kick off EOS EVM test runs
- Add support for configuring EOS EVM Operational Mode
  - Add similar configuration controls as already supported similar operational modes
- Transaction Generator Launcher updates
  - Dynamically configure what type of transaction generator to launch
- EVM Transaction Generator
  - Port Transaction Generator framework from leap for reuse in EOS EVM
  - Initial Transaction Generator to create token swap transactions
- Extend Performance Harness Reporting
  - Transaction metrics already analyzed and calculated should hold true for Token Swap Transactions
    - min/max/average transactions (swaps) per second
    - latency measured from time sent to block inclusion time
    - transaction CPU
    - transaction net
    - if applicable:
      - transaction ack response time
  - Peak sustainable swaps per second will be measured through use of the Performance Harness Test
    which iteratively (using binary search) runs performance harness basic tests configured to search
    the window of realistically possible TPS ranges and outputs whether the configured TPS was sustainable
    over a short test duration by comparing average TPS achieved vs. configured TPS target.
  - Peak swaps per second "burst" will be measured by looking at the max TPS achieved in a given successful run.
    This will effectively measure the TPS achievable in short burst without necessarily guaranteeing sustainably
    being able to process that TPS load over a long period of time.
  - Provide updated argument reporting for EOS EVM test run configuration details

## Results Handling

- Single Performance Harness Test Runs
  - Currently a Results Report is provided along with logs of more detailed information from the run as described in the (Result Reports)[https://github.com/AntelopeIO/leap/tree/main/tests/performance_tests#result-reports]
  - This could be extended to provide graph visualizations for histograms on a per block or per transaction basis
    - Transaction latency over time
    - Transactions per block over time
    - This would require additional logging of metrics on a per transaction basis not currently tracked to be able to add the visualization capabilities
- Multi-Run Comparison
  - To be able to see performance statistics and performance analysis/comparison between runs the following would be needed
    - Manual (could be auto) CI workflow run to kick off a preconfigured test scenario on a given commit/branch
    - Location to store logs and reports
      - github repo
      - cloud storage
      - server
    - Location to provide visualizations
      - Needs access to wherever reports and logs are stored
      - Needs capability to update and interact with graphs as new data points are provided (could be static auto generated graphs)
      - Visualizations make the most sense on longer term initiatives where incremental performance increase or decreases are of value.
        For instance:
        - on each commit/PR merge into main to see how main's performance improvement or degradation
        - on each merge into release branches
        - comparing each release
        - Multiple versions in play:
          - leap version
          - EOS EVM contract version
          - EOS EVM Node Version
          - Future: EOS EVM RPC Node Version
      - Visualizations for current development work typically would find the most benefit comparing the current development delta's performance against the current main's performance
        - This could either benefit from 2 data points, main and current dev branch's head OR
        - a histogram of main and then all subsequent commits to the development branch

### Engage with ENF to plot course on the following topics

-   Engage with ENF to identify:
    - Individuals at ENF to engage with
    - How to gather and persist data
    - What data to gather and persist
    - How to incorporate EOS EVM Performance Harness into CI/CD or auto-deployment
    - Initial control surface
      - simple UI
      - Github workflow input params
    - Visualizations desired (tool/technology, graphs, etc.)
      - Static auto-generated graphics/visualizations
      - Dynamic graphs/visualizations

## Documentation

- Provide documentation on how to configure and run the EOS EVM Performance Harness Tests
- Provide overview of components involved
- Provide Test Result Reports documentation

## Opportunities for future development

- Heterogeneous transaction generation, measure performance in a mixed transaction environment
  (Could also be a replay test)
- Measure additional components performance (EOS EVM RPC)
  - Ability to query EVM transactions
  - Time for transactions to be replicated in EVM blocks and accessible through EOS EVM RPC queries
    - Requires additionally running SHIP node and RPC node(s)
- Analysis of performance runs over time
  - Look at developing dashboard or UI for visualization of performance over time
    - Determine cadence of performance runs (i.e. Pull Request, Merge main, release, etc.)
    - What statistics are of interest to compare/track