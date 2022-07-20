### Experimental Binary Package

There are a handful of binary packages we require for DUNE that we want to be easily accessible to DUNE but not upfront and visible to users alongside other releases. This is due to us not supporting these DUNE related binaries at the same level of support as the other released binaries that go alongside our release notes as assets.

To be specific, there are 3 binary .deb packages:
* the x86_64 "-dev" package
* ARM8 package
* ARM8 "-dev" package

### Criteria For Storage Location

Since we're not going to place these packages as assets in a release, we need another location to store them. Criteria for storage is:
* Leverage a GitHub service so that it is no-cost for us
* Package versions should be well labeled and easily discoverable based on version so that DUNE (or some other tool) can download them
* Storage of new packages should be easily automatable upon creation of a release (not now, but in future)

### Single-Layer Docker Image As Storage

We're going to store these .deb packages as a single layer Docker image named `experimental-binaries`. This is certainly stretching what the intended purpose is of a Docker image, but it meets all above criteria.

* It's a GitHub service (container storage), so it's no-cost for us
* Docker images can be labeled and referenced with "v3.1.0-rc2", "v3.2.3", etc
* It's trivial to grant a GitHub Action write access to a container package store

The only downside is that it's non-trivial to download these packages. An example that extracts all .deb packages in version v3.1.0-rc2 would be:

```
CONTAINER_PACKAGE=eosnetworkfoundation/experimental-binaries
GH_ANON_BEARER=$(curl -s "https://ghcr.io/token?service=registry.docker.io&scope=repository:${CONTAINER_PACKAGE}:pull" | jq -r .token)

curl -s -L -H "Authorization: Bearer ${GH_ANON_BEARER}" https://ghcr.io/v2/${CONTAINER_PACKAGE}/blobs/$(curl -s -L -H "Authorization: Bearer ${GH_ANON_BEARER}" https://ghcr.io/v2/${CONTAINER_PACKAGE}/manifests/v3.1.0-rc2 | jq -r .layers[0].digest) | tar zx
```

