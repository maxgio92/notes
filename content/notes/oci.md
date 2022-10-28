---
Title: OCI specs and manifests
---

The [specification](https://github.com/opencontainers/image-spec/blob/main/spec.md#understanding-the-specification)

Relations:
![image](https://user-images.githubusercontent.com/7593929/198673399-72ee6eef-ea13-4295-b0cb-3dcf7d899aa6.png)

Main OCI [media types](https://github.com/opencontainers/image-spec/blob/main/media-types.md#oci-image-media-types):
- OCI [content descriptor](https://github.com/opencontainers/image-spec/blob/main/descriptor.md) descripts a comoponent of an image. It can reference:
  - OCI [image manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md)
  - OCI [artifact manifest](https://github.com/opencontainers/image-spec/blob/main/artifact.md)
  - OCI [layer](https://github.com/opencontainers/image-spec/blob/main/layer.md)
  - OCI artifact [blobs](https://github.com/opencontainers/image-spec/blob/main/artifact.md#artifact-manifest-property-descriptions)
  - ...
