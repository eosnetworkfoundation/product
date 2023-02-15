# Performance Testing Harness Stage 3 Planning

Roadmap for the development of a **Performance Testing Harness** -- Stage 3.

Requirements are being curated for the development of a performance testing and evaluation harness here: [Performance Testing Harness Requirements & Planning & Staging](https://github.com/eosnetworkfoundation/product/tree/main/performance-harness/proposals)


## Stage 3

###   Multi-producer & "relay" or "canary" node configuration support (Carried over from Stage 2)
    -   Add configurable network topology based on public networks
        -   multiple producer nodes, each accessed through a dedicated relay/canary node
        -   all relay nodes connected in mesh
    -   Include configurable "relay" nodes
        -   Specific configuration per node type
    -   Expand reporting to include multiple nodes

### Work with ENF to identify additional metrics that would be useful to publish (Carried over from Stage 2)
    -   Support additional metrics that would be useful to publish

### Expand Transaction generator test scenarios

-   EVM Transactions (Carried over from Stage 2)
-   Large action
    -   Large action payload (~200KB string payload) that is not used
-   DB intense actions
    -   Lots of reads with no writes
    -   Lots of writes with no reads
    -   Mix of reads/writes
-   Kitchen Sink (mix of all transaction types)?

### Additional Latency Measuremeants

-   Measure latency:
    -   From creation
    -   From consumption in nodeos
    -   From consumption from SHiP
    -   From consumption from trace_api_plugin

### Reporting

-   Work with ENF to create reporting template
    -   Customize text and charts for various audiences
    -   Produce output in multiple formats
        -   HTML
        -   PDF
        -   CSV

### Migrate from manual cli deployment to auto deployment

-   Begin Design Planning for migration to auto deployment (From Stage 2)
    -   Work with ENF to building architectural/design doc
    -   SPIKE: Evaluation of technologies
    -   Recommendation for technologies to use

-   Begin Migration (based on outcome of planning)

    -   Begin migrating to use of selected technology (i.e. Ansible and Terraform or similar)
    -   Work with ENF to design gui
        -   Preferred technology/language
        -   UI/UX Design
        -   Where to deploy
    -   Provide barebones "gui" for changing configuration
    -   Work with ENF to determine what configurations to run?
    -   Work with ENF to determine how frequently to run?

### Increase output evaluation statistics

-   Work with ENF to determine which statistics should be built into output or graphed
    -   Graphical representation of results
    -   Plotting of latency and other stats

### Storage of historical data for comparison

-   Work with ENF to determine:
    -   What data to store
    -   Where to store data
-   Availability of data from web?
-   Github pages? GCP?






