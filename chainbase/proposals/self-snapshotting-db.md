## Self Snapshotting Chainbase DB For Seamless Nodeos Upgrades

Some drawbacks to chainbase's in-memory representation that have haunted us since 1.0 is that it is sensitive to breakage with different boost & compiler versions as well as when needing to modify the layout of its structure or tables (such as when adding WA keys, for example).

The former we have attempted to mitigate with pinned builds. The latter we never had a solution for. In both cases we've just become complacent to tell users that whenever they perform a minor upgrade of nodeos they need to _manually remove some files and restore from a portable snapshot_. Bonkers from a usability standpoint. One would expect nodeos to seamlessly upgrade itself without user intervention.

We've had some ongoing discussions that we may be able to eliminate pinned builds by fixing the version of boost used for chainbase. However even if solving that, we'd still need to ask the user to perform the "manually remove some files" dance when we want to modify some underlying chainbase structure. We need a better more flexible solution that is transparent for the user in the majority of likely cases.

Conceptually what would work as a general purpose seamless upgrade mechanism is if for some eosio chainbase DB (i.e. some shared_memory.bin) it could self describe how to perform a portable snapshot from itself. This would allow a future nodeos, once detecting that the shared_memory.bin was built with a different compiler or older version of nodeos, to have the shared_memory.bin provide a portable snapshot of itself that nodeos can then load its state from.

### Embedded Shared Library

As a fairly easy implementation we can have nodeos embed a shared library in to the shared_memory.bin that instructs future nodeos' how to create a portable snapshot from the existing data.

When some future nodeos starts and loads shared_memory.bin but discovers that either the environment check fails or that the format of the database is not compatible, it will then extract the shared library from the old shared_memory.bin and run the single simple function it exposes such as `create_snapshot(char* dbmem, size_t dbmemsz, int outputfd)` that will create a portable snapshot. The future nodeos will then erase the old DB and reload with the newly created portable snapshot. All of this will be performed without user interaction but with some indication the operation is ongoing since it may take a minute or two.

### Limitations

This will only seamlessly work when going forward in time. A shared library built on Ubuntu 22.04 will not necessarily be able to load on Ubuntu 18.04 (but the converse will, as long as we migrate consensus level crypto -- libchain -- to boringssl or such). A shared library from nodeos 5.0.0 may produce portable snapshot v8 which nodeos 3.1.0 will be unable to parse (but nodeos 5.0.0 would be able to load snapshot v5 that a shared library from 3.1.0 produces).

### Security Implications & Sandboxing

This solution effectively means a shared_memory.bin can execute arbitrary code with the same permissions as that of the nodeos process. This is highly unexpected behavior that could be exploited by a malicious actor. Thus, any code executing in the shared library must be highly sandboxed. Notably, a shared library can execute code before `dlopen()` returns, so the sandbox needs to be in place before this call. The sandboxed code should only be able to output a portable snapshot; that's it -- no write access to other files, no socket access, etc. It needs to offer full protection even if nodeos is running as root.

For Linux seccomp should suffice. For macOS `sandbox_init()`/"Seatbelt" probably suffices. I'm not familiar enough with Win32 or FreeBSD to know what the options are there, but we can simply not implement this feature on those platforms. 

### Performance Exploration

It would be worth exploring modifications to the existing snapshotting code that would let a snapshot be streamed in parallel from the old DB to the new DB. This would improve user experience by decreasing the amount of time required to perform this transparent upgrade.