# Alternative CDT implementation proposal

The latest version of CDT is based on LLVM9 with some custom modifications which presents some challenges in the following areas

* hard to upgrade to the latest version of LLVM and Clang
* hard to integrate with the existing C++ toolchain system such as profiler or static analysis tools,
* lack of debugger support

We propose to provide an alternative CDT version to meet the previous challeges especially providing the ability to debug their contracts with debugger.
In addition, the proposed version provides an alternative native execution which can highlight undefined behaviors in contract code allowing for cleaner more robust code.
Not only  does it improve the contract developer's productivity, it also helps the quality of contracts. 

## Approach

### Decouple the code generation and code compilation.
Current `eosio-cpp` in CDT integrates the entire contract compiling process into one executable and provides a lot of non-stadnard arguments to control its behaviors. This makes it harder to maintain and upgrade. We propose to separate the code and ABI generation process into another executable and leave the compiler/linker as they are without modificcation. To faciliate the user experience, the cmake `add_contract` function should be rewritten to integerate the code/abi generation with the compilation process.

### Provide a different native build mode
Current CDT native build generates an executable to be execute on the host system and can be only used for eosiolib library unit test only. There is no direct support to build excutable contracts into native binary. We propose to revamp it to build smart contract into shared libraries on the host system. To use the generated shared libary, a test launcher can be developed to load and executes the shared libraries for debug purpose.

### Provide different cmake toolchain file for wasm and native build:
To provide both wasm and native build to users, we also need to provide different cmake toolchain files to switch between those two modes. 


### Provide a contract test launcher for launching the native contract.
This contract test launcher need to provide the implementation of eosio specific intrinsics and a library for creating and manage test chains, controling how the contracts are loaded and executed.

## Benefits
With the changes described above, we could achieve 

* far faster CDT toolchain build time:  just use the pacakged LLVM binary from upstream to avoid the length compilation process of LLVM and Clang;
* smaller and faster contracts codes: the code size and speed up the contract could be about 10% better than the LLVM7 based solution;
* debugger support: compiling the contract into native shared library and then can be loaded from modified nodeos or tester for debugging purpose;
* existing tool chain integration: using standard `clang` instead of the specialized `eosio-cpp` makes it easier to integration with other tools such clang-tidy which can be used for contract code refactoring.

