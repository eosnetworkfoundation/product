## chainbase memory usability improvements

These two improvements to chainbase are fairly low effort and improve usability considerably. Obviously the ideal situation would be to completely rewrite chainbase to solve all known problems but that's not on the table at the moment.

They're both a little hacky; one more so than the other. But it seems like a positive net improvement.

### new hybrid mapped mode

The default map mode for chainbase's state (where it just mmap()s in the state file) is known to be highly detrimental to nodeos performance; particularly on IOPS constrained VMs. What tends to happen is that the Linux kernel frequently writes the dirty pages out to disk clogging performance.

I'd like to propose elimination of the mapped mode and replacing it with a mode where chainbase (nodeos) tracks its dirty pages itself and periodically writes back pages that have not been written to in some time frame. Critically, via a low priority thread with ioniceness.

This would still give the quick start up and shut down performance expected from mapped mode but without the slowdown from disk grinding (at least after starting up). Certainly performance will be worse than heap mode (anonymous memory mode) but that's okay: this is aiming for a "good enough" performance baseline for users who take the default and right now the performance isn't good enough for many users when they launch on a VM.

Ideal case this is implemented on modern Linux with userfaultfd+io_uring combo. And it's probably worthwhile to prototype with that to get a feel on how well the approach floats. But we'd also need a vanilla SEGV implementation for macOS or older Linux kernels.

### eliminate chain-state-db-size-mb

This is an extremely difficult setting to recommend proper configuration of. Should it be what will likely suffice for the next year? (but what about a unique event like EOS experienced recently) Should it be the amount of sold RAM? Available RAM? Should you just set it to a large number and forget about it? (the latter interferes with a reasonable heap mode) It doesn't help matters that the default chain-state-db-guard-size-mb isn't sufficient for something like WAX. Just a really awful configuration setting that is a huge impediment to "just work"ing.

Based on completion of the prior section there is a really quick hacky solution to eliminate the setting: just give chainbase 1TB and track what pages it is actually using; even for heap mode. The hacky part here is that it leans on undocumented behavior of boost::interprocess::allocator aggressively reusing freed memory before fresh memory. That is, if you give it 1TB and let WAX run on it, it'll really only end up using the first 90GB or so. It won't sprawl out over time.

Obviously relying on undocumented behavior is yucky. But realistically the risk here is low. The boost::interprocess::allocator hasn't been touched in 5 years and that was a superficial change. You'd have to go back to 2015 to see something substantial. The likelihood of this changing out from under us is low enough to regularly eyeball.

This could be mitigated by including a fixed version of boost within chainbase. Something we've noodled on a little before and may be required for performance improvement in other proposal.

### use process RAM for uncommitted changes

Currently chainbase uses the mmap memory for everything, including uncommitted changes that are coming with speculative transactions. But the uncommitted changes don't have to be in the state memory, and it makes sense to store them in normal malloc memory. This would reduce the I/O load on the state storage significantly, so that we would probably not need tmpfs any longer.
