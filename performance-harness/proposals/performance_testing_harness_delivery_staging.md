# Performance Testing Harness Delivery Staging

Staged roadmap for the development of a **Performance Testing Harness**.

Requirements are being curated for the development of a performance testing and evaluation harness here: [Performace Testing Harness Requirements](https://github.com/eosnetworkfoundation/product/tree/main/performance-harness/proposals)


---
## MVP
---

### Transaction Source

-   Transaction generator

    -   minimal

    -   most likely just transfers

    -   p2p format using net plugin

    -   configurable

        -   Token transfers per second (tps) for generating
        -   number of threads

    -   tracks generation rate

        -   exit on poor performance
        -   report if Source or Provider is at fault

-   Transaction Provider

    -   establishes p2p connection to net_plugin

    -   configurable

        -   number of connections (to the same nodeos)
        -   configurable number of threads

### Run Harness

-   python script

    -   setup environment
    -   launch nodeos node(s)
    -   run Transaction Sources
    -   end test
    -   run output evaluation
    -   Create CSV results for ingestion in the future

### Output Evaluation

-   python script

    -   scrape nodeos logs

    -   evaluate

        -   min
        -   max
        -   standard deviation

    -   Capture of logs

    -   Reporting of configurations and run env to CSV report

        -   Nodeos version, configuration options, env, etc

    -   TPS Scoring

        -   Standard deviation, mean, average (basic stats)

### User Documentation

---
## Stage 2
---

### Expand Transaction generator

-   configurable

    -   number of accounts

### Expand MVP to support multi-producer setup

-   Coordinate TPS generation across multi-nodes
-   Possibly multiple generators across nodes

### Include configurable "relay" nodes

-   Specific configuration per node type

### Expand reporting to include multiple nodes

-   Number of Forks

-   Number of dropped blocks

    -   Missed blocks
    -   Forked blocks

-   Number of dropped transactions

-   Number of complete windows

---
## Stage 3
---

### HTTP harness testing

-   Setup HTTP performance testing across single node

### Additional contract type

-   EOS mechanics

### Manual cli deployment to auto deployment

-   Use of Ansible or similar
-   Provide barebones "gui" for changing configuration

### Increase output evaluation statistics

-   Graphical representation of results
-   Plotting of latency and other stats

### Storage of historical data for comparison

-   Availability of data from web

---
## Additional Planned Development (To Prioritize and Stage)
---
-   *Work in progress*