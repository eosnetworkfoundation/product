# Proposal for new command line tool “leap-util”

Background
---
Nodeos is getting cluttered with commands. These commands should be spun out into a separate utility leap-util per:

https://github.com/eosnetworkfoundation/mandel/issues/697

More to that, we already have tools such eosio-blocklog containing all-blocklog functionality that should be integrated into one leap-util tool for clarity and simplicity of use.

Having a single “entry point” to a maintenance tool hopefully will make it easier for users.


Tool Proposal
---
Proposed leap-util tool should be a unified tool to manage all eosio executable services that will return to a command line upon execution. This excludes any tasks requiring spawning background processes/daemons, listening to sockets/etc.
Since unified tool should have ability to accept hierarchical command line arguments, it can be very similar to a very well knows Amazon “aws” command line tool, in particular:
leap-util [options] <command> <subcommand> [parameters]
Potential examples of using this tool can be as follows (these are for illustration purpose only, not real examples to be implemented):

    leap-util snapshot to-json -o snapshot.json
    leap-util blocklog vacuum –blocks-dir ./blocks

    Where options are:
        -h [ --help ]                         Print this help message and exit.
        -v [ --version ]                      Print version information.
        --full-version                        Print full version information.



<command> - is the “service” name.

Currently when invoking commands with nodeos, we need to provide a plugin name to which certain commands will be directed. It may be beneficial to create a user friendly abstraction and group commands in meaningful service names instead of plugin names, for example:

    snapshot
    blocklog
    …

<subcommand> - a command within a service associated with a single finite action that returns a value/completes command execution

Ideas for Usability Improvement
---
There are several ideas that can make leap-util tool more user friendly:


- Auto completion of commands / subcommands / parameters
- Using colored output when possible
- Nicely formatted success / error reporting
- Hints on incorrect input (such as "did you mean blocklog?")


From the development standpoint it is desirable to make integration of new commands and functionality as easy as possible and selected framework/architecture should account for that

Implementation Options
---
There are multiple cli parsing/helper libraries available for C/C++. Unless research shows any substantial benefits of alternative libraries, we will probably continue to use **CLI11** (https://cliutils.github.io/CLI11/book/), as it is supporting subcommands naturally, it is a one BSD licensed header and it is already used in "cleos"

For the reference, please see https://hackingcpp.com/cpp/libs/cmdline_args_parsing.html

![](https://hackmd.io/_uploads/BJv6ZqxA9.png)

Very useful comparison between Boost PO and CLI11 can be found here:
https://iscinumpy.dev/post/comparing-cli11-and-boostpo/

CLI11-based Implementation
---
This proposal comes with demo/scaffold implementation of two simple subcommands here:

https://github.com/AntelopeIO/leap/tree/leap-util

Following desing considerations were taken into account:
- make adding a new subcommands easy, clear and sort of plugin-ish. Provide abstract interface for subcommand and offer developer clear way of defining command, struct with options and handlers
- every subcommand interface implementation should be in separate files
- inclusion/exclusion into/from main() must be trivial
- autocomplete handled "automatically", nothing needs to be done by developer implementing new subcommand


CLI11 Bash Autocomplete Implementation
---
Release (and development) versions of CLI11 arew lacking autocomplete implementation. However, it can be asilly added. There is a working implementation done by the user HailoRT on github here:

https://github.com/hailo-ai/CLI11/commit/635773b0a1d76a1744c122b98eda6702c909edb2

I forked, built and tested these changes and autocomplete works perfectly for bash. My fork is here:

https://github.com/766C6164/CLI11

In order to build custom version of CLI11 library (including single-header file):

    mkdir build
    cd build
    cmake -DCLI11_SINGLE_FILE=ON ..
    make -j

To start enjoying benefiuts of auto-complete with bash:
- used custom build CLI11 (forked version, or patch release version) as specified above
- add a "lookup" configuration file/script to appropriate bash directory https://github.com/AntelopeIO/leap/commit/97c19806d956ae48b10f793419340e1e1668af58

How it works (standard way for bash):

when tab is pressed after user entered command + additional characters bash will invoke appropriate script that was placed in bash autocomplete config folder that will pull available command line options/subcommands directly from leap-util with following call: leap-util --_autocomplete [query string]. Then complete command will be executed to do completion.

Completion works really well in leap-util branch and can be plugged-in into other CLI11-based cli utils such as cleod in a matter of minutes:
https://github.com/AntelopeIO/leap/tree/leap-util


Commands to be Migrated to leap-util (for now)
---

All of eosio-blocklog (and deprecating eosio-backlog tool):


    eosio-blocklog command line options:
      --blocks-dir arg (="blocks")    the location of the blocks directory
                                      (absolute path or relative to the current
                                      directory)
      -o [ --output-file ] arg        the file to write the output to (absolute or
                                      relative path).  If not specified then output
                                      is to stdout.
      -f [ --first ] arg (=0)         the first block number to log or the first to
                                      keep if trim-blocklog
      -l [ --last ] arg (=4294967295) the last block number to log or the last to
                                      keep if trim-blocklog
      --no-pretty-print               Do not pretty print the output.  Useful if
                                      piping to jq to improve performance.
      --as-json-array                 Print out json blocks wrapped in json array
                                      (otherwise the output is free-standing json
                                      objects).
      --make-index                    Create blocks.index from blocks.log. Must
                                      give 'blocks-dir'. Give 'output-file'
                                      relative to current directory or absolute
                                      path (default is <blocks-dir>/blocks.index).
      --trim-blocklog                 Trim blocks.log and blocks.index. Must give
                                      'blocks-dir' and 'first' and/or 'last'.
      --extract-blocks                Extract range of blocks from blocks.log and
                                      write to output-dir.  Must give 'first'
                                      and/or 'last'.
      --output-dir arg                the output directory for the block log
                                      extracted from blocks-dir
      --smoke-test                    Quick test that blocks.log and blocks.index
                                      are well formed and agree with each other.
      --vacuum                        Vacuum a pruned blocks.log in to an un-pruned
                                      blocks.log
      -h [ --help ]                   Print this help message and exit.

Discussion topics
---

1. In order to use bash autocomplete, a patched or forked version of CLI11 needs to be used. We need to decide what will be a right approach.
2. Also for autocomplete, a config script for bash needs to be added to installation step for *.deb packages. Should we require bash-completion package as mandatory dependency for leap, or if its not installed on user system, just ignore/warn about it and autocomplete wont work
3. Since we would be using CLI11 already in 2 places and potentially other cli utils will use it in future, it should be probably shaped into library vs one header file.

###### tags: `Proposal` `leap-util`
