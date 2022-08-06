---
Title: ELF and the Linux kernel binary
---

## Readings

* ELF and the special sections of Linux binary: https://lwn.net/Articles/531148/
* 101: https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/
* Sections and segments: https://stackoverflow.com/questions/21279618/when-is-an-elf-text-segment-not-an-elf-text-segment
* Comment section:
  * https://www.quora.com/What-is-the-significance-of-comment-section-in-ELF
* GCC and the comment section:
  * https://stackoverflow.com/questions/16033385/how-to-know-the-gcc-version-used-to-build-the-linux
  * https://stackoverflow.com/questions/2387040/how-to-retrieve-the-gcc-version-used-to-compile-a-given-elf-executable
* Kernel packages: https://dl.managed-protection.com/u/baas/help/20.10/user/en-US/linux-packages.html?TocPath=Installing%20the%20software%7C_____2
* https://www.geeksforgeeks.org/difference-between-linker-and-loader/

## Get GCC from a Linux release

### 0. `extract-vmlinux`

Download `vmlinux`-extractor script, which extracts the ELF binary from the `zlib`-compressed `vmlinuz` file.

```shell
curl -sLO https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux
```

### 1. Download the image package

For example, a CentOS-released one:

```shell
curl -sLO http://archive.kernel.org/centos-vault/7.2.1511/os/x86_64/Packages/kernel-3.10.0-327.el7.x86_64.rpm
```

Extract to CPIO, and uncompress it:

```shell
rpm2cpio kernel-3.10.0-327.el7.x86_64.rpm | cpio -idmv
```

### 2. Extract the ELF binary:

Extract the Linux ELF binary:

```shell
./extract-vmlinux ./kernel-3.10.0-327.el7.x86_64/boot/vmlinuz-3.10.0-327.el7.x86_64 > ./kernel-3.10.0-327.el7.x86_64/vmlinux-3.10.0-327.el7.x86_64
```

### 3. Dissect the binary:

```shell
objdump \
 --full-contents \
 --section .comment \
 ./kernel-3.10.0-327.el7.x86_64/vmlinux-3.10.0-327.el7.x86_64
```
