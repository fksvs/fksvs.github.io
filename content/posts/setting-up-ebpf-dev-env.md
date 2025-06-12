+++
date = '2025-06-12T20:44:52+03:00'
title = 'Setting Up eBPF Development Environment'
description = 'Setting Up eBPF Development Environment'
categories = ['ebpf', 'linux']
+++

Before writing any eBPF programs, it’s essential to set up a proper development environment. Equally important is understanding the overall eBPF workflow, from writing and compiling your eBPF code to loading it into the kernel and inspecting its output.
<!--more-->

## eBPF Development Workflow

1. **Prepare your development environment**: Install the necessary toolchain and libraries.  
2. **Write your eBPF program**: Create an eBPF program that implements your logic in kernel space.  
3. **Write your user-space loader and listener**: Using C, Go, Rust, or any supported language, write a program that loads your compiled eBPF object, attaches it to a hook, and interacts with eBPF maps.  
4. **Compile the eBPF program**: Use Clang to generate eBPF bytecode.  
5. **Load and attach the eBPF program**: Use your user-space loader to load the eBPF bytecode and attach it to the appropriate hook.  
6. **Verify and inspect output**: Check your logs, perf ring buffers, or other eBPF maps to confirm that your eBPF program is running and emitting events as expected.  
7. **Detach and unload**: When you’re finished, gracefully detach your eBPF program and unload it from the kernel.

## Setting Up Environment

### Choosing Linux Distribution

Any Linux distribution running kernel 5.15 or newer (6.x recommended) is suitable for eBPF work.  
eBPF was introduced in 4.8, but most modern helpers, CO-RE and BTF support expect at least 5.10.  
For example, this guide was prepared on Fedora 41 with kernel 6.14.

Check your kernel version:

```shell
$ uname -r
6.14.9-200.fc41.x86_64
```

### Kernel Headers & BTF

Clang needs headers that exactly match the running kernel.

**Fedora/RPM Family**
```shell
# Update your system first
$ sudo dnf update

# Install matching headers
$ sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

**Debian/Ubuntu**
```shell
# Update your system first
$ sudo apt update

# Install kernel headers
$ sudo apt install linux-headers-$(uname -r)
```

BTF (BPF Type Format) is a lightweight binary table that lists the kernel’s C types and field offsets. Tools like libbpf read this table so a single eBPF object can adjust itself to different kernel versions. 

Verify that your kernel ships with BTF:

```shell
$  grep CONFIG_DEBUG_INFO_BTF /boot/config-$(uname -r)
# should return: CONFIG_DEBUG_INFO_BTF=y
```

and confirm the BTF object is present:

```shell
$ ls /sys/kernel/btf/vmlinux
```

If the file is missing, rebuild the kernel with `CONFIG_DEBUG_INFO_BTF=y`.

### Installing Necessary Toolchains

LVM/Clang is the preferred compiler for eBPF. You’ll also need bpftool, libbpf, and bpftrace.

**Fedora**
```shell
$ sudo dnf install clang llvm bpftool elfutils-libelf-devel libbpf libbpf-devel bpftrace
```

**Debian / Ubuntu**
```shell
$ sudo apt install clang llvm bpftool libelf-dev libbpf-dev bpftrace
```

## Resources

- [eBPF application development: Beyond the basics](https://developers.redhat.com/articles/2023/10/19/ebpf-application-development-beyond-basics#)
- [eBPF Tutorial by eunomia](https://eunomia.dev/tutorials/1-helloworld/)