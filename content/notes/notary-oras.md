---
Title: Notary, ORAS and OCI
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

Already changes are coming in ORAS that unified the ORAS artifact spec into the new OCI artifact spec, to cover scenarios where images aren't the only artifact to be distributed, such as signatures, SBOMs, attestation, etc. but that references container-related artifacts.

## OCI and ORAS

Notation leverages ORAS to store signatures into OCI registries.
The [ORAS project](https://oras.land/) is a set of tools and libraries that enable to use OCI registries to store arbitray artifacts.
But what are OCI Registries?

The [Open Container Initiative](https://opencontainers.org/) (OCI) defines the specifications and standards for container technologies.
This includes the [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec), the API for working with container registries.
Registries that implement the distribution-spec are referred to as OCI Registries.

### OCI and ORAS artifacts

The main OCI artifact type is the [OCI image](https://github.com/opencontainers/image-spec). With time, people used registries to store arbitrary artifacts, leveraging performance, security and reliability capabilities of registries.
One growing example is artifacts for securing the sofware supply chain like SBOMS, signatures, attestations, scan results, etc.

The [OCI artifacts](https://github.com/opencontainers/artifacts) project aims to generalize the artifact types that can be distributed by and stored into OCI registries. 
The image manifest has a `config.mediaType` field to differentiate between the various types of artifacts.
This field is supposed to be filled by the authors of new artifact types, so ORAS did to support a wide range of artifact types.

It introduced the [ORAS artifact specification](https://github.com/oras-project/artifacts-spec) and related `application/vnd.cncf.oras.artifact.manifest.v1+json` `mediaType`.
This media type bases on the OCI image manifest but removes constraints such as a required `config` object and required & ordinal `layers` (more on the OCI image manifest spec [here](https://github.com/opencontainers/image-spec/blob/main/manifest.md)).

The ORAS artifact manifest adds a `subject` property supporting a graph of independently linked artifacts.
It provides a means to define artifacts that can be related to an OCI image manifest, OCI image index or another ORAS artifact manifest (for example [here](https://github.com/oras-project/artifacts-spec/blob/main/scenarios.md#notary-v2-signatures) a Notary V2 signature that references an image).
By defining a new manifest, registries and clients opt-into new capabilities, without breaking existing behaviour, such as discovery provided by the ORAS [referrers API](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md#manifest-referrers-api).

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

## Show me the code

- [runSign()](https://github.com/notaryproject/notation/blob/main/cmd/notation/sign.go#L65)
- [prepareSigningContent()](https://github.com/notaryproject/notation/blob/main/cmd/notation/sign.go#L96)
- [pushSignature()](https://github.com/notaryproject/notation/blob/main/cmd/notation/sign.go#L111)

### COSE

As a detail, the supported signing protocols are [JWS](https://www.rfc-editor.org/rfc/rfc7515) and [COSE](https://www.rfc-editor.org/rfc/rfc9052).

JWS represents content secured with digital signatures or Message Authentication Codes (MACs) using JSON-based data structures

COSE describes how to create and process signatures, message authentication codes, and encryption using CBOR for serialization.
[CBOR](https://www.rfc-editor.org/rfc/rfc7049) is a data format whose design goals include the possibility of small code size, small message size, and extensibility without the need for version negotiation. 
These design goals make it different from earlier binary serializations such as ASN.1 and MessagePack.
CBOR was designed specifically to be small in terms of both messages transported and implementation size and to have a schema-free decoder.

CBOR extended the data model of JavaScript Object Notation (JSON) by allowing for binary data, among other changes. 

- CBOR has capabilities that are not present in JSON and are appropriate to use.  One example of this is the fact that CBOR has a method of encoding binary data directly without first converting it into a base64-encoded text string.
- COSE is not a direct copy of the JOSE specification.  In the process of creating COSE, decisions that were made for JOSE were re-examined.  In many cases, different results were decided on, as the criteria were not always the same.

The [JOSE](https://jose.readthedocs.io/en/latest/) Working Group produced a set of documents [RFC7515](https://www.rfc-editor.org/rfc/rfc7515) [RFC7516](https://www.rfc-editor.org/rfc/rfc7516) [RFC7517](https://www.rfc-editor.org/rfc/rfc7517) [RFC7518](https://www.rfc-editor.org/rfc/rfc7518) that specified how to process encryption, signatures, and Message Authentication Code (MAC) operations and how to encode keys using JSON (like for JWS).

## What's next

It happened that ORAS worked to unify their artifact specification into a new OCI standard specification (Reference Types for image and distribution specs).

We're waiting to see a bump in the Notation Go library to [support the Reference Type](https://github.com/notaryproject/notation-go/issues/136) (and then Notation CLI) of the ORAS Go library (now release candidate [v2.0.0-rc.4](https://github.com/oras-project/oras-go/tree/v2.0.0-rc.4)).

Sources:
- https://oras.land/blog/oras-0.15-a-fully-functional-registry-client/#whats-next-for-oras
- https://github.com/opencontainers/image-spec/pull/934
- https://github.com/opencontainers/distribution-spec/pull/335
- https://github.com/oras-project/oras-go/issues/271
- https://github.com/notaryproject/notation-go/issues/136

See you soon with updates on the OCI artifact spec!
