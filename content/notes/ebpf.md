---
title: eBPF links
---

## General

- http://vger.kernel.org/lpc_bpf2018_talks/Using_eBPF_as_a_heterogeneous_processing_ABI_LPC_2018.pdf

## libbpf

- https://github.com/libbpf/libbpf-bootstrap
- https://nakryiko.com/posts/libbpf-bootstrap/

## CO-RE

### Features

#### Dev time

> From “recap” paragraph of https://nakryiko.com/posts/bpf-portability-and-co-re/#bpf-state-of-the-art
- vmlinux.h eliminates dependency on kernel headers;
- field relocations (field offsets, existence, size, etc) make data extraction from kernel portable;
- libbpf-provided Kconfig extern variables allow BPF programs to accommodate various kernel version- and configuration-specific changes;
- when everything else fails, app-provided read-only configuration and struct flavors are an ultimate big hammer to address whatever complicated scenario application has to handle.

## BTF

- https://kinvolk.io/blog/2022/03/btfgen-one-step-closer-to-truly-portable-ebpf-programs/
- https://blog.aquasec.com/vmlinux.h-ebpf-programs
- https://nakryiko.com/posts/bpf-portability-and-co-re/
- https://nakryiko.com/posts/btf-dedup/
- https://github.com/aquasecurity/tracee/tree/main/cmd/tracee-ebpf
- https://github.com/aquasecurity/tracee/blob/1244d3fb53044c7472b4d23ce9f870f3f0115962/cmd/tracee-ebpf/main.go#L604
- https://www.kernel.org/doc/html/v5.11/bpf/btf.html
- https://nakryiko.com/posts/bpf-core-reference-guide
- https://github.com/torvalds/linux/tree/master/tools/bpf/runqslower
- http://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html
- https://www.solo.io/blog/porting-ebpf-applications-to-bumblebee/

### Relocations

- https://www.kernel.org/doc/html/latest/bpf/llvm_reloc.html

### Tracee

- https://aquasecurity.github.io/tracee/v0.6.5/building/nocore-ebpf/

## Related

- https://www.linkedin.com/pulse/elf-linux-executable-plt-got-tables-mohammad-alhyari/
