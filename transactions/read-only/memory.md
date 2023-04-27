# Parallel Read-only Transaction Execution Improvements
Several improvement ideas related to WASM execution of parallel read-only trxs were identified: parsing contracts only once (https://github.com/AntelopeIO/leap/issues/800), reducing EOS-VM-OC memory slice count (https://github.com/AntelopeIO/leap/issues/645), and calculating virtual memory available to user space correctly (https://github.com/AntelopeIO/leap/issues/801).

This document describes the background and solutions.

## WASM Execution
A transaction execution goes through two major phases:
1. The smart contract of the transaction's actions are translated by CDT from C++ into WASM code.
2. The WASM is executed by an EOS VM.

`nodeos` supports three types of VMs for WASM execution: EOS-VM (interpreter), EOS-VM-JIT, and EOS-VM-OC.

At `nodeos` startup, resources required by WASM execution are created:
- a WAMS interface `wasmif`
- a `EOS-VM` or `EOS-VM-JIT` run time interface `eos_vm_runtime` (based on configuration option `--wasm-runtime`),
- and a `eosvmoc_tier` (when `--eos-vm-oc-enable` is configured). `eosvmoc_tier` contains `eosvmoc::executor`
and `eosvmoc::memory` used for execution. 

```mermaid
flowchart TD
    wasmif[create wasmif] --> runtime
    runtime{EOS-VM or EOS-VM-JIT?} -->|JIT| jitVM
    runtime -->|interpreter| intVM
    jitVM[create JIT eos_vm_runtime] --> ocEnabled
    intVM[create interpreter eos_vm_runtime] --> ocEnabled
    ocEnabled{eos-vm-oc-enable?} -->|yes| ocTier
    ocTier[create eosvmoc_tier]
```

### EOS-VM and EOS-VM-JIT Execution

EOS-VM and EOS-VM-JIT execute similarly. Both involves module, execution context, and backend.

A module stores information about a parsed WASM code, consisting of WASM section definitions and generated bitcode or X86 machine code.
```mermaid
classDiagram
   class module {
      growable_allocator              allocator; 
      uint32_t                        start; // index of start function
      guarded_vector<func_type>       types; // type definitions
      guarded_vector<import_entry>    imports; // imported functions
      guarded_vector<uint32_t>        functions; // functions definitions
      guarded_vector<table_type>      tables; // table definitions
      guarded_vector<memory_type>     memories; // memory definitions
      guarded_vector<global_variable> globals; // globals definitions and storage 
      guarded_vector<export_entry>    exports; // export functions
      guarded_vector<elem_segment>    elements; // element definitions
      guarded_vector<function_body>   code; // generated code
      guarded_vector<data_segment>    data; // initial data
   }
```

An execution context provides a runtime environment to execute a function (action). It instantiates a module by linking the module with
linear memory and stack, and initializing memory.
```mermaid
classDiagram
   class execution_context_base {
      char*                           _linear_memory;
      module&                         _mod;
      wasm_allocator*                 _wasm_alloc;
      detail::host_invoker_t<Host>    _rhf;
      operand_stack                   _os;
   }
   class jit_execution_context {
      execute()
      ...()
   }
   class execution_context {
      execute()
      ...()
   }
   execution_context_base <|-- jit_execution_context
   execution_context_base <|-- execution_context
```

A backend encapsulates everything required for execution an action. The constructor of `backend` takes code,
host functions, allocator and options as input, constructs a parser, parses the code, and creates the execution context as an output.

Put together, given a contract WASM code and a function (action), `wasmif` executes the function in the following steps:

```mermaid
flowchart TD
    isBackendCached{backend for the code cached?} -->|No| backend
    isBackendCached -->|Yes| exec
    backend[create a backend] --> module
    module[create a module] --> isjit
    isjit{JIT?} -->|JIT| jitParse
    isjit -->|interpreter| intParse
    jitParse[parse the contract code, populate module, generate X86 machine code] --> jitCtx
    intParse[parse the contract code, populate module, generate bitcode] --> intCtx
    jitCtx[create a JIT execution context] --> exec
    intCtx[create an interpreter execution context] --> exec
    exec[execute the function]
```

### EOS-VM-OC Execution
To improve the performance, EOS-VM-OC compiles and generates optimized code.

Given a contract WASM code and a function, `eosvmoc_tier` executes the function in the following steps:

```mermaid
flowchart TD
    cached{code compiled and cached?} -->|yes| exec
    cached -->|no| default
    exec[executor::execute] --> mem
    mem[prepare memory] --> cb
    cb[prepare control block] --> setApply
    setApply[set up apply func] --> apply
    apply["call apply func"]
    default[fall back to base runtime for execution]
```
#### Memory Slices

The description of this section is from https://github.com/AntelopeIO/leap/issues/645.

EOS-VM-OC uses a memory mirroring technique so that WASM linear memory can be both protected via page access permissions
and be resized without using `mprotect()`. EOS VM OC refers to these mirrors as "slices".

Prior to 2.1/3.1, Antelope's WASM memory could never exceed 33MiB. EOS-VM-OC would set up `33MiB/64KiB+1=529` slices each
with approximately `4GiB+33MiB` of virtual memory. This meant that EOS-VM-OC required approximately `529*(4GiB+33MiB)` of virtual memory;
about 2.1TiB.

In 2.1+/3.1+ Antelope technically (but Leap does not reliably) supports WASM memory up to full 4GiB (Leap's supported WASM limits 
are only those as defined in the reference contracts which remain 33MiB). EOS-VM-OC was modified so that any growth beyond 33MiB
is handled via `mprotect()`. This allows the optimization to remain in replace for all supported usages of Leap, while still allowing
Leap to technically support the full Antelope protocol which allows any size up to 4GiB. However, this support still required 
increasing the size of a slice to a full 8GiB of virtual memory, meaning that EOS VM OC now requires `529*8GiB` of virtual memory; about 4.2TiB.

```mermaid
flowchart TD
   subgraph vm[Virtual Memory: 4.2TB]
     direction LR
     subgraph slice1[Slice 1 maps prologue]
     end
     subgraph slice2[Slice 2 maps prologue + 1 page]
     end
     subgraph slice529[Slice 529 maps prologue + 528 pages]
     end
   end
   
   subgraph lm[Linear Memory: 4GB]
     direction TB
     subgraph mapped[mapped memory: 33MB]
     end
     subgraph mprotected[mprotect memory: 4GB - 33MB]
     end
     mapped ~~~ mprotected
   end
   
   slice1 ---> mapped
   slice2 ---> mapped
   slice529 ---> mapped
```

## Parallel WASM Execution

### Parallel EOS-VM and EOS-VM-JIT Execution

To support transaction execution on multiple threads, a separate `wasmif` is created for each thread.
The runtime environments (compiled code and memory) are completely isolated from each other.

```mermaid
flowchart LR
   subgraph wasmif_1
      subgraph backend_1[backend]
         subgraph execution_ctx_1[execution context]
            direction TB
            subgraph module1[module]
            end
            subgraph LM1[linear memory]
            end
            subgraph Stack1[stack]
            end
            module1 ~~~ LM1
            LM1 ~~~ Stack1
         end
      end
   end

   subgraph wasmif_2
      subgraph backend_2[backend]
         subgraph execution_ctx_2[execution context]
            direction TB
            subgraph module2[module]
            end
            subgraph LM2[linear memory]
            end
            subgraph Stack2[stack]
            end
            module2 ~~~ LM2
            LM2 ~~~ Stack2

         end
      end
   end
   
   wasmif_1 ~~~ wasmif_2

```

### Parallel EOS-VM-OC Execution
EOS-VM-OC was designed with multi-threaded support in mind. Only one `wasmif` is required. The same compiled code
can be used on multiple threads; each thread uses its own executor and memory.

```mermaid
flowchart LR
   subgraph wasmif
      subgraph eosvmoc[eosvmoc]
         subgraph code[compiled code]
            direction LR
            subgraph runtime1[executor, memory]
            end
            subgraph runtime2[executor, memory]
            end
            
            runtime1 ~~~ runtime2
         end
      end
   end
```

## Issues

### Same Contract Parsed Multiple Times in EOS-VM and EOS-VM-JIT
In multi-threaded EOS-VM and EOS-VM-JIT execution, each thread requires a separate `wasmif`, which results in the same
contract parsed multiple times. Reuse of the parsed code will improve performance.

### Large Virtual Memory Required in EOS-VM-OC

EOS-VM-OC requires a separate memory for each executing thread. 
For example, if 16 parallel threads are allowed with the current strategy of requiring 529 slices per thread, 
that would require `16*4.2TiB` of virtual memory: more virtual memory than allowed on most processors. 
Future efforts, like sync calls and background memory scrubbing, will also increase the need for more active slice sets.

### Virtual Memory Available to User Space Not Calculated Accurately

To determine number of threads allowed for EOS-VM-OC, we need to know total virtual memory available to user space.
Currently we use `VmallocTotal` in `/prop/meminfo`. This is not accurate, as `VmallocTotal` reports virtual memory for kernel allocation;
kernel itself uses some of it.

## Proposed Solutions

### Compile Once for the Same Contract in EOS-VM and EOS-VM-JIT

To avoid parsing contract code multiple times, it is proposed the parsed module is shared by multiple execution contexts:
```mermaid
flowchart TD
   subgraph module[module]
   end
%%  subgraph wamsifs
%%  direction LR
   subgraph wasmif_1
      subgraph backend_1[backend]
         subgraph execution_ctx_1[execution context]
            direction TB
  %%          subgraph module1[module]
  %%          end
            subgraph LM1[linear memory]
            end
            subgraph Stack1[stack]
            end
   %%         module1 ~~~ LM1
            LM1 ~~~ Stack1
         end
      end
   end

   subgraph wasmif_2
      subgraph backend_2[backend]
         subgraph execution_ctx_2[execution context]
            direction TB
     %%       subgraph module2[module]
     %%      end
            subgraph LM2[linear memory]
            end
            subgraph Stack2[stack]
            end
     %%     module2 ~~~ LM2
            LM2 ~~~ Stack2
         end
      end
   end   
   wasmif_1 ~~~ wasmif_2
%%   end
%%   module1 --- module2
   module --> wasmif_1
   module --> wasmif_2
   
```

However, module stores global values (`current` in `global_variable`) in the globals section. This makes globals not thread-safe.

```
   struct init_expr {
      int8_t     opcode;
      expr_value value;
   };
   struct global_type {
      value_type content_type;
      bool       mutability;
   };
   struct global_variable {
      global_type type;
      init_expr   init;
      init_expr   current;
   };
```

In EOS-VM, `current` is modified by `set_global` in `eos-vm/include/eosio/vm/execution_context.hpp`
```
      inline void set_global(uint32_t index, const operand_stack_elem& el) {
      ...
      auto& gl = _mod.globals[index];
      ...
      visit(overloaded{ [&](const i32_const_t& i) {
         gl.current.value.i32 = i.data.ui;
         },
      ...
```
               
In EOS-VM-JIT, `current` is directly accessed in `eos-vm/include/eosio/vm/x86_64.hpp`
```
      void emit_set_global(uint32_t globalidx) {
         auto icount = fixed_size_instr(14);
         auto& gl = _mod.globals[globalidx];
         void *ptr = &gl.current.value;
         // popq %rcx
         emit_bytes(0x59);
         // movabsq $ptr, %rax
         emit_bytes(0x48, 0xb8);
         emit_operand_ptr(ptr);
         // movq %rcx, (%rax)
         emit_bytes(0x48, 0x89, 0x08);
      }
```

All other sections in module are thread-safe. The following table summarizes the thread safety:

| Section   | Description                                                                       | Thread Safe?|
|-----------|-----------------------------------------------------------------------------------|-------------|
| allocator | used only during parsing                                                          | yes         |
| start     | index of start function, shareable                                                | yes         |
| types     | type definitions, shareable                                                       | yes         |
| imports   | imported functions, shareable                                                     | yes         |
| functions | function definitions, shareable                                                   | yes         |
| tables    | table definitions, shareable                                                      | yes         |
| memories  | memory definition, shareable                                                      | yes         |
| globals   | globals definitions and storage                                                   | no          |
| exports   | exported functions, shareable                                                     | yes         |
| elements  | used to initialize tables, shareable                                              | yes         |
| code      | instructions, shareable                                                           | yes         |
| data      | initial data, shareable                                                           | yes         |

We plan to
- Move globals storage out from module and into linear memory
- Modify `set_global` and `get_global` in `execution_context.hpp`, `emit_get_global` and `emit_set_global` in `x86_64.hpp`.
- Cache parsed module per contract
- Cached module is removed from the cache when last block its code was used is before LIB and its linked execution context is not running.

### Reduce Memory Slice Counts
To conserve virtual memory to support more threads, We need to reduce the threshold where EOS VM OC transitions
from mirroring to `mprotect()`.

We plan to
- Gather memory usage from existing contracts.
  * Select most recent one year's EOS Mainnet snapshots and block logs.
  * Modify `eosvmoc::executor::execute()` to report `cb->current_linear_memory_pages` at the end of execution.
  * Replay the block logs.
  * Find max and 95 percentile of current_linear_memory_pages. This is to make sure vast majority of executions do not need using `mprotect`.
- Add a private compile option defining the threshold number of pages where the transition between the two approaches occurs.
  * In `chain/CMakefile`, set a Private variable `max_num_slices` and pass it to using `add_definitions` to C++ code.
  * Use `max_num_slices` to set up memory slices in `memory::memory()`.

### Calculate Virtual Memory Available to User Space Accurately
We plan to use `/proc/self/maps`, find the difference of addresses between `nodeos` program text and `vsyscall` and deduct the total
size of all memory segments.

A memory maps look like
```
560e88dc8000-560e88dcc000 rw-p 04dee000 103:03 20582300                  /home/lh/work/leap-4-0-vmoc-main-thread/build/bin/nodeos
560e88dcc000-560e88df7000 rw-p 00000000 00:00 0
560e8a03d000-560e8a107000 rw-p 00000000 00:00 0                          [heap]
...
7ffdceec3000-7ffdceee4000 rw-p 00000000 00:00 0                          [stack]
7ffdcefe2000-7ffdcefe6000 r--p 00000000 00:00 0                          [vvar]
7ffdcefe6000-7ffdcefe8000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

