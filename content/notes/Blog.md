---
title: Blogging
---

## CO-RE

### Features

#### Dev time

> see recap paragraph of https://nakryiko.com/posts/bpf-portability-and-co-re/#bpf-state-of-the-art

- vmlinux.h eliminates dependency on kernel headers;

- field relocations (field offsets, existence, size, etc) make data extraction from kernel portable;

- libbpf-provided Kconfig extern variables allow BPF programs to accommodate various kernel version- and configuration-specific changes;

- when everything else fails, app-provided read-only configuration and struct flavors are an ultimate big hammer to address whatever complicated scenario application has to handle.
