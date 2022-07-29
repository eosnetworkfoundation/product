## Initial Build/CI Tenets

Most of these talk to the main mandel repo, as it's the most complex, but basic principles apply to other repos we build deliverables from too.

### High Performance Builds 

CI builds and tests need to turn around as fast as possible. Waiting 1 hour for a simple merge like we're seeing on GitHub's free builders is not acceptable. Builds and tests need to be performed on very fast high core count boxes. We can accomplish this for a low price and with limitless scaling by using ephemeral on-demand cloud instances. Our long term goal should be somewhere in the 5 minute ballpark to do a full CI run.

### Single Linux Binary Build To Rule Them All (well, per Platform & Arch)

Only build a single Linux binary that runs on any modern Linux distro. Ultimately this single binary is the only binary we support. This makes it easier to wrangle reproducible builds below. It also makes it easier to accommodate users who want to run something other then Ubuntu and/or its LTS releases as well as LTS releases made between releases of our software.

We'll still want to run this binary on at least a couple distros in testing, and will still want to test multiple builds with different compiler & library combinations to fuzz around different compilers and perform builds that represent a user just building manually from the command line on our supported platforms.

One downside with this approach is that currently it means we'll end up static linking OpenSSL. This is undesirable since it static links in a TLS implementation and we'd need to be mindful to do a quick turnaround should there be a security issue. Mid-term we can solve this problem, in the short-term I think it's a trade off we need to make.

More generally, this means we need to be mindful to only dynamic link to properly dated system provided libraries that have a strict long term compatibility regime. Acceptable libraries would include, for example, glibc, zlib, and libcurl. Whereas libraries such as openssl, boost, and LLVM do not make such guarantees. libc++ typically does a good job with compatibility, but historically we want something newer than what is included in our oldest supported distro so we need to static link a newer one.

### Fully Reproducible Binary Builds

The single supported binary packaged in to a .deb must be fully byte-for-byte reproducible. We can already get this with 99+% certainty just by building in a container which is what we have done forever. But to get it 100% certain we need to build in a distro supporting pinned packages. We should build our single Linux binary with [snapshotted Debian packages](https://snapshot.debian.org/).

A user needs to be able to easily build this reproducible binary. So, critically, there will be no (directly exposed) build scripts. The command a user performs to locally build a pinned reproducible binary is something like
```
DOCKER_BUILDKIT=1 docker build -o . .
```
and out pops the mandel-3.1.0.deb file that can be installed. This is simple enough to be documented in the README.

Requiring Docker to build the reproducible pinned binaries may cause some friction with some users. Once we implement the Self Snapshotting Chainbase DB, we can move away entirely from the "pinned" requirement and terminology -- this build simply becomes the "Reproducible Build" making it easier for users to avoid it if they'd like (and simply use the manual build steps without losing the compatibility guarantees of the current pinned builds).

The design should be mindful to provide easy access to the pinned compiler toolchain for development purposes (for example, to try and fix a compiler error locally). A carefully constructed dockerfile could accomplish this by something like
```
DOCKER_BUILDKIT=1 docker build --target export-toolchain -o - . > pinned-toolchain.tar
```

### Cross-Build non-Linux non-x86 Binaries

We already support ARM8 Linux builds, in the future we may support other platforms again (macOS, Windows). Always build all the binaries on the fast ephemeral Linux hosts. i.e. cross-compile all other platforms and architectures.

While this does trade some complexity for some other complexity I think it's the right trade off. There are a number of benefits: It allows anyone with access to an x86 Linux box the ability to perform a byte-for-byte reproducible build of the ARM8 build, or some other platform build, etc. It eliminates variability in build times (e.g. building on a 128 core x86 machine vs just a 48 core ARM machine). And should we ever support other platforms again, it eliminates maintaining completely unique build environments and reduces macOS physical fleet requirements.

We will still need to test on real ARM hardware though, so we will not be able to fully escape some of the overhead in maintaining different environments.

