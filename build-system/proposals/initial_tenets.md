### Initial Build/CI Tenets

Most of these talk to the main mandel repo, as it's the most complex, but basic principles apply to other repos we build deliverables from too.

#### Build Fast. Build Cheap.

CI builds and tests need to turn around as fast as possible. Waiting 1 hour for a simple merge like we're seeing on Github's free builders is not acceptable. Builds and tests need to be performed on very fast high core count boxes. We can accomplish this for a low price and with limitless scaling by using ephemeral on-demand cloud instances. Our long term goal should be somewhere in the 5 minute ballpark to do a full CI run.

#### Single Linux Binary Build To Rule Them All (per Platform & Arch)

Only build a single Linux binary that runs on any modern Linux distro. While we'll still want to run this binary on at least a couple distros in CI testing, and may still want to do multiple builds with different compiler & library combinations to test through CI just to fuzz around different compilers, ultimately this single binary is the only binary we support. This makes it easier to wrangle reproducible builds below. It also makes it easier to accommodate users who want to run something other then Ubuntu and/or its LTS releases

One downside with this approach is that currently it means we'll end up static linking OpenSSL. This is undesirable since it static links in a TLS implementation and we'd need to be mindful to do a quick turnaround should there be a security issue. Mid-term we can solve this problem, in the short-term I think it's a trade off we need to make.

#### Fully Reproducible Binary Builds

The .deb and other packages we produce must be fully byte-for-byte reproducable. This is most easily achieved by building in a container with a disto supporting pinned packages (like Debian).

So, critically, no (directly exposed) build scripts. The command a user performs to locally build a pinned reproducible binary is something like
```
DOCKER_BUILDKIT=1 docker build -o . .
```
and out pops the mandel-3.1.0.deb file that can be installed. This is simple enough to be documented in the README.

Users should still have the option of building non-pinned via simple instructions.

#### Cross-Build non-Linux non-x86 Binaries

We already support ARM8 Linux builds, in the future we may support macOS, Windows, etc. Always build all the binaries on the fast ephemeral Linux hosts. i.e. cross-compile all other platforms and architectures.

While this does trade some complexity for some other complexity I think it's the right trade off. It allows anyone with access to an x86 Linux box the ability to perform a byte-for-byte reproducible build of the ARM8 build, or the macOS build, etc. If we do support macOS it reduces the physical fleet requirements. It eliminates maintaining completely unique build environments for macOS & Windows.

We still need real ARM hardware, real macOS boxes, etc to test on though. With the exception of macOS, those too can be on-demand cloud instances.

