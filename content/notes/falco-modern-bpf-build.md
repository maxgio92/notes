---
Title: Building the Falco modern BPF probe
---

## Requirements:

- bpftool
- clang >= 12
- cmake
- GNU make
- exposed vmlinux in BTF format (`/sys/kernel/btf/vmlinux`)

### BPFTool

#### Requirements

- libelf
- zlib

```shell
apt update && apt install -y libelf-dev zlib1g-dev
```

#### Build

On Ubuntu 22 LTS build it locally, to avoid not having it for Linux 5.15:

```shell
apt update && apt install -y git
git clone --recurse-submodules https://github.com/libbpf/bpftool.git
cd bpftool/src
make install
```

## Build

```shell
git clone https://github.com/falcosecurity/libs
mkdir -p libs/build
cd libs/build
cmake -DUSE_BUNDLED_DEPS=ON -DBUILD_LIBSCAP_MODERN_BPF=ON -DBUILD_LIBSCAP_GVISOR=OFF .. 
make ProbeSkeleton
```

## Test it

```shell
make scap-open
sudo ./libscap/examples/01-open/scap-open --modern_bpf --evt_type 1
```

