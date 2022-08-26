# Regression Test Harness High-level Review

The regression test harness is a framework written in Python currently in use
to run many of the tests in the `tests` directory.  It is used in tests
which require one or more running instances of `nodeos` and `keosd`.

The current test harness evolved over time.  As developers realized some
particular action was becoming common across tests, it was simply tacked
on to the existing classes wherever was convenient at the time.  No effort
was made to then adopt the new method across all tests.  Subsequently
authored tests may or may not use the common functionality provided by the
test harness.  It is also dependent on `eosio-launcher` for the most
convenient path for bringing up a network of `nodeos` instances.

This document outlines some steps needed to improve the maintainability,
utility, and reliability of the test harness.

## First Step: Replace eosio-launcher

The first major step required is to reimplement `eosio-launcher` in Python.
The `eosio-launcher` is a non-trivial program written in C++ which accepts
counts of nodes and their types as well as (optionally) a network topology,
and launches all of the requested nodeos instances with the required network
and storage configurations.  It has some additional commands for managing
running `nodeos` instances such as `--down`, `--kill` (which is distinct from
`--down`), and `--bounce`, some of which depend on a set of shell scripts
to find and kill processes using traditional UNIX command line tools.

The launcher and its accompanying scripts contain hardcoded assumptions
about the filesystem and about network ports, further exacerbated by those
same assumptions mirrored in the Python code.  The result is brittle, making
it impossible to run regression tests in parallel.  Because `nodeos` is
tied to real time, this limitation imposes a substantial cost in the time
required to run all of the existing regression tests sequentially.

With three layers of software in three languages, process control is a
problem in the current test harness.  Some control paths track process ids
in files while others do not.  The common and general usage of the test
harness will indiscriminately issue kill signals to every `nodeos` instance
running on the system, including those which it did not launch.

Reimplementing `eosio-launcher` in Python will allow reliable full lifecycle
control of processes and their associated storage.  It is required in order
to introduce a much better separation of concerns which is currently lacking.

## Improve flexibility

Once `eosio-launcher` is replaced and reliable process control is centralized
in the Python code, the next step is to improve the flexibility of the
test harness by removing all hardcoded assumptions about filesystem
paths and network ports.  Much of this comes as a side effect of replacing
`eosio-launcher`.  That replacement work is not simply a line-by-line
transcription of the existing C++ into similar Python.  Instead, the
functionality of `eosio-launcher` will be broken up into cluster and node
control modules, concepts which currently exist in the Python code but
which are much abbreviated and, in the case of node control, badly stunted.

After the migration, the Python node will be responsible for the filesystem
paths and network ports used by the `nodeos` instance it is managing.  This
allows for automated and coordinated collision avoidance.

### Improve transaction tracking

Currently all high level interfaces which send transactions to `nodeos` have
an option to wait for the transaction to appear in a block.  In order to
streamline tests which require a group of several transactions to be verified,
an interface will be added to wait for every id in a list of transaction ids
to appear in a block.  This interface will optionally wait for the transactions
to be in an irreversible block.

### Dependency reduction

Some tests which use the test harness invoke `curl`.  The `curl` tool is no
longer installed by default on recent Debian derivatives.  To eliminate the
dependency on `curl`, the test harness will make use of the Python standard
library `urllib.request` module.

### Add the Python package to a Debian package

To facilitate use by others, especially eosjs, once the test harness has been
converted to a Python package, it will be added to an appropriate Debian
package using the `dh-python` tooling provided by the Debian project.

### Filesystem usage

When a new `nodeos` instance is launched for a test, it writes a substantial
number of files to disk, including the block log.  These files must be kept
separate for each instance.  The current hardcoded paths, as mediated by
`eosio-launcher`, follow a naming scheme intended to be developer-friendly.
The first node runs out of the `var/lib/node_bios` directory.  Subsequent
nodes, should they be required, run out of `var/lib/node_00`, 
`var/lib/node_01`, etc.  This naming scheme allows the developer to make use
of the test harness during development work.  The log files for each node
are in predictable, easily remembered locations, making it easy for a developer
to investigate the causes of a failed test.

This naming scheme will be preserved and extended.  The `var/lib` prefix will
be removed.  In its place will be a parent directory named after the name of
the test being run as it is known to `ctest`.  This allows all tests to run
simultaneously while maintaining the required filesystem isolation without
the loss of human navigability that would be incurred by randomly generated
temporary directory names.

In addition, the path names for a cluster and for individual nodes will be
customizable to facilitate cases where greater clarity can be achieved by
specific node directory names.

### Network usage

A running `nodeos` instance listens on up to four different network ports,
depending on which plugins are enabled, and there is a community requested
feature which would expand that to five.  Berkley sockets restrict listening
to one process per IP address and port, requiring coordination between nodes
in a cluster and across clusters to avoid collisions.

The current test harness hardcodes an initial listen port of 8888 on 127.0.0.1
and uses each port in sequence per node in the cluster for the http plugin.
The revised system will take advantage of the entirety of the 127.0.0.0/8
subnet by choosing an IP address to run the cluster on deterministically based
on the name of the test as it is known to `ctest`.  The listen port for the
http plugin will remain the default 8888 and sequential ports, though at least
one test will start on an alternate port to ensure customized ports work
throughout the system.

As with filesystem usage, the IP address and ports used will be customizable.
The test harness currently exposes the port number for the HTTP plugin for
customization but does not honor it in all cases.  This defect will be
corrected and the option for customization extended to the other
network-enabled plugins.  The IP address will be exposed for customization.
The `eosio-launcher` currently supports custom IP addresses, including remote
IP addresses.  This functionality will be preserved while reimplementing it
in Python.

## Improved support for typical scenarios

The test harness serves dual purposes: it executes as part of the automated
testing invoked prior to accepting a Github pull request and it is used by
developers on their own machines to quickly and easily compose test scenarios
during development.

### Improved CI/CD support

To improve the utility of the test harness in CI/CD scenarios, it will:

- Provide support for executing tests in mixed network scenarios, including:
  - Release builds from multiple versions;
  - Debug builds;
  - Pinned builds.
- Execute code coverage tests in cooperation with CMake.
- Capture logs and other test artifacts into archives on commmand for upload to:
  - Github test results;
  - A status dashboard showing test results and code coverage statistics.

### Improved debugging support

To further improve its utility in debugging scenarios, it will:

- Utilize the new, more direct process control to detect abnormal termination
  of nodeos and invoke the tools required to recover a core dump on modern
  Linux distributions.
  - Extract the stack trace from core dumps in human readable form and
    incorporate it into the logs.
- Capture the topology of the `nodeos` network as it exists in the test and
  render it in graphviz format for ease of human interpretation.
- Support customization of the `logging.json` file used to control the log
  levels within the `nodeos` log system on a per node basis.
- Optionally combine the logs from requested nodes into a single file,
  annotated as necessary to indicate the origin of each line of text, to
  provide a unified view of the timeline of the network during the test.
  - Provide support for simple filters during generation of a combined log
- Incorporate what has evolved into a substantial amount of boilerplate in
  every test into appropriate setup methods and members within the test harness
  classes.

