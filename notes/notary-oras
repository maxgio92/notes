---
Title: Sign images with Notation
---

# Sign image with Notary and ORAS

## Notary and Notation

[Notary project](https://notaryproject.dev/) is a set of tools that helps you sign, store, and verify OCI artifacts using OCI-conformant registries.

[Notation]() is an implementation of the [Notary v2 specifications](https://github.com/notaryproject/notaryproject).
As an implementation provides a CLI that adds signatures as standard items in the registry ecosystem, and can build a set of simple tooling for signing and verifying these signatures.

Notary v2 provides for multiple signatures of an [OCI Artifact](https://github.com/opencontainers/artifacts) (including container images) to be persisted in an OCI conformant registry.
Artifacts are signed with private keys, and validated with public keys.

To support user deployment flows, signing an OCI Artifact will not change the @digest or artifact:tag reference.
To support content movement across multiple certification boundaries, artifacts and their signatures will be easily copied within and across OCI conformant registries.

To deliver on the Notary v2 goals of cross registry movement of artifacts with their signatures, changes to several projects are anticipated, including [OCI distribution-spec](https://github.com/opencontainers/distribution-spec), [CNCF Distribution](https://github.com/distribution/distribution), [OCI Artifacts](https://github.com/opencontainers/artifacts), [ORAS](https://github.com/oras-project/oras) with further consumption from projects (e.g. [containerd](https://github.com/containerd/containerd)).

Already changes are coming in ORAS that unified the ORAS artifact spec into the new OCI artifact spec, to cover scenarios where images aren't the only artifact to be distributed, such as signatures, SBOMs, attestation, etc.

## OCI and ORAS

Notation leverages ORAS to store signatures into OCI registry [...].

[ORAS]() project is [...].

## Quickstart

### Requirements

- [docker](https://docs.docker.com/engine/reference/commandline/cli/) or [podman](https://docs.podman.io/en/latest/Commands.html)
- [notation](https://github.com/notaryproject/notation/releases)
- [skopeo](https://github.com/containers/skopeo/blob/main/install.md)

### Run the demo

Run a local [ORAS](https://github.com/oras-project) OCI regsitry:

```shell
export PORT=5000
export REGISTRY=localhost:${PORT}

docker run -d -p ${PORT}:5000 ghcr.io/oras-project/registry:v0.0.3-alpha
```

Build and push an OCI image to the local registry with a tag:

```shell
export REPO=${REGISTRY}/net-monitor
export IMAGE=${REPO}:v1

docker build -t $IMAGE https://github.com/wabbit-networks/net-monitor.git#main
docker push $IMAGE
```

See how the image is not signed (the are no signing artifacts on the local repository that reference the just pushed image):

```shell
notation list --plain-http $IMAGE
```

Generate a certificate key pair to sign the image:

```shell
notation cert generate-test --default "wabbit-networks.io"
```

Sign the image with the certificate key just created:

```shell
notation sign --plain-http $IMAGE
```

Add the certificate to the local trust store:

```shell
notation cert add --name "wabbit-networks.io" ~/.config/notation/localkeys/wabbit-networks.io.crt
```

Verify that the image is signed, against the trust store.

```shell
notation verify --plain-http $IMAGE
```

But now, let's get more detail and see what is a signature.

## Inpspect the signature artifacts

As of now of Notation v0.12 a signature is an ORAS [artifact-spec](https://github.com/oras-project/artifacts-spec) compatible OCI image, on an OCI registry that references an OCI image.
As a digest makes unique an artifact (i.e. an image), the `subject` field of the signature image manifest references the signing content.

Let's check that on our local registry!

### Inspect with Skopeo

Skopeo is a command line utility that performs various operations on container images and image repositories.
It is able to inspect a repository on a container registry and fetch images manifests and layers.

So, let's analyse the local registry with Skopeo, and see how the signed image is referenced by the signature in his image manifest.

Get the signature digest:

```shell
notation list --plain-http $IMAGE
```

Inspect the `subject`'s digest of the signature image manifest:

```shell
skopeo inspect --tls-verify=false docker://localhost:5000/net-monitor@sha256:$SIG_DIGEST --raw | jq '.subject.digest'
```

We see that it matches the digest of the signed image.

The end!
