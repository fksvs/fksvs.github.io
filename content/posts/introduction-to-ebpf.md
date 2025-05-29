+++
date = '2025-05-29T12:09:46+03:00'
title = 'Introduction to eBPF'
description = 'A brief introduction into eBPF'
categories = ['ebpf', 'linux']
+++

## What is eBPF ?

eBPF (extended Berkeley Packet Filter) is a modern Linux kernel feature that lets you safely plug in small, sandboxed programs to extend or observe system behavior without modifying or reconfiguring the kernel or applications.<!--more-->Think of it as a flexible add-on platform: you write a tiny piece of code (an eBPF program), load it at runtime into various hooks (more on hooks later), and instantly gain powerful tracing, networking, or security capabilities. This approach leaves your core system unchanged while opening up new ways to monitor and enhance both kernel and application functionality.

## Why Use eBPF ?

The Linux kernel is complex; not all applications are suited for source-code modification or deep monitoring. Making any change to a codebase requires intimate familiarity with its existing code and architecture. At the same time, you’ll face challenges that aren’t purely technical when you try to contribute those changes upstream. Patches must be accepted by the community (and you may have no ability to modify proprietary software), all in the name of “greater good.”

For example, imagine you’ve devised a clever way to intercept the system call that accepts incoming network connections. You could build a kernel module or patch the kernel itself, but then, a new Linux release arrives, and your solution may not work across versions. What if a customer needs it on an older LTS kernel? And if you want it merged upstream, you’re looking at weeks of discussion, revisions and extra development. Clearly, that’s not an ideal workflow.

This is where eBPF comes into play. eBPF programs can be loaded dynamically into the running kernel where no source edits, no reboots and no module installs is required. There are dozens of hooks (tracepoints, kprobes, and more) you can attach to and trace. In our example, you simply write an eBPF program, attach it to the appropriate tracepoint or kprobe for socket accepts, and start monitoring immediately. No downtime, no merge conflict. Just write, load and observe.

## History of eBPF

I won’t dive too deeply into eBPF’s history, it’s a long story. eBPF evolved from the original BPF beginning with Linux kernel 3.8 in 2014. That release completely overhauled the BPF instruction set, rewrote the interpreter, introduced eBPF maps, added the eBPF verifier, and brought in the new `bpf()` system call so user-space programs can interact with kernel-loaded eBPF code.

## How eBPF Work: Architecture Overview

### Kernel Space
---
#### eBPF Programs

An eBPF program is a compact stream of bytecode that the Linux kernel stores in memory and executes whenever a chosen hook triggers it. You usually write the source in a restricted subset of C compiled with Clang, but modern toolchains also let you generate the same bytecode from Rust via Aya, Go with cilium/ebpf loaders that embed C, Python through BCC which compiles C snippets on the fly, or even hand-crafted assembly. The compiler produces an ELF object whose text section holds fixed-length eBPF instructions. Once the kernel loads this object, it can JIT-translate the bytecode into native machine code so it runs at almost native speed.

#### eBPF Hooks

Hooks are like bus stops, bus stops in stop and pick-up-drop off passengers, it's the same for our kernels. When the kernel reaches to a hook, it stops there and runs the attached eBPF program. There are a lot of different hooks such as tracepoint, kprobes, uprobes, XDP and more that give a broad data access and enables them to complete complex operations. We can use tracepoints or kprobes to hook and get information about system calls for example, or we can use XDP to listen network packets and do some processing on them. eBPF hooks is one of the main things in eBPF subsystem.

#### eBPF Helper Functions

eBPF Hooks are like bus stops: just as buses pause to pick up or drop off passengers, the kernel pauses at a hook and runs any attached eBPF program. There are many types of hooks such as tracepoints, kprobes, uprobes, XDP, and more. Each providing access to different data and enabling complex operations. For example, you can use tracepoints or kprobes to inspect system calls, or XDP to intercept and process network packets at the driver level. eBPF hooks are a core feature of the eBPF subsystem.

#### eBPF Maps

eBPF maps are shared data structures that let eBPF programs exchange information with one another and with user-space processes which is not something included in classic BPF. Because a map lives in kernel memory, both kernel-space code (other eBPF programs) and user-space applications can read from or write to it, as long as they agree on the map’s layout and alignment.

Typical patterns, summarized from _Learning eBPF_ by Liz Rice, include:

- Pushing configuration down: a user-space tool writes policy or tuning data into a map so an eBPF program can pick it up at runtime.

- Saving state across events: one eBPF program stores context in a map for a later invocation of the same program or for a different eBPF program that needs that state.

- Exporting results up: an eBPF program records metrics, counters, or detection events in a map, and a user- space agent retrieves them to log, visualize, or trigger alerts.

By acting as a fast, lock-free bridge between the kernel and user space, maps are a cornerstone of almost every production eBPF workflow.

#### eBPF Verifier

The eBPF verifier is a core component of the eBPF subsystem and the safety inspector for every eBPF program. Before the kernel runs any code it walks every possible path to be sure loops end, memory stays in safe bounds and only approved helper calls appear. It also watches stack size and blocks instructions that could hang or damage the system. If anything looks risky the verifier rejects the upload so a faulty eBPF program never gets the chance to crash the kernel.

#### eBPF Bytecode & Virtual Machine

eBPF bytecode is the slim set of instructions that a C, Rust, or similar source file becomes after you compile it with clang or another eBPF-aware compiler. The kernel keeps this bytecode in a protected memory area and feeds it to the in-kernel eBPF virtual machine, this virtual machine is a software implementation of CPU with ten 64-bit registers and a 512-byte stack. When an attached hook fires, the virtual machine JIT-translates them into the host CPU’s native opcodes so they run almost as fast as regular kernel code. The verifier’s earlier checks guarantee the program cannot stray outside its sandbox, giving you safe, high-performance logic that lives entirely inside the kernel.

#### eBPF CO-RE & BTF

CO-RE (Compile Once, Run Everywhere) lets you compile an eBPF program against a single kernel version and then deploy that same binary across multiple kernels by automatically adjusting for any structural differences at load time. To enable this, you generate a `vmlinux.h` header with `bpftool` from a running system, which contains the exact data-structure and function-signature layouts that your program will encounter.

BTF (BPF Type Format) provides a compact metadata format for those layouts, and when you compile with CO-RE enabled, the compiler embeds BTF-derived relocation hints into the bytecode. At load time, a library like libbpf uses those hints to patch field offsets and sizes to match the live kernel, so your program “just works” everywhere without manual header juggling.

### User Space
---
#### Compiling

The compiler is the stage that turns your readable C or Rust code into eBPF bytecode. Today, either Clang or a modern GCC can handle this translation.

#### Loading and Attaching

During the loading step, a small user-space loader (for example `bpftool` or your own loader) feeds the bytecode to the kernel. If the kernel’s verifier gives it a green light, the program is attached to the event you chose (a network packet, a tracepoint, and so on) and pinned under `/sys/fs/bpf/<program-name>`.

## Use Cases of eBPF

### Security Auditing

eBPF lets us drop lightweight programs right into the Linux kernel, so they can watch system calls, network traffic, and other system events in real time. Catching things this early means we can detect threats, network anomalies, or even track down malware long before regular user-space tools would notice. And since everything runs inside the kernel, we skip the extra overhead of bouncing back and forth between kernel and user space. eBPF can also enforce runtime policies by blocking or altering suspicious events before they cause harm, turning it into a high-performance, modern way to audit and protect a system in real time.

### Performance Monitoring

With eBPF, checking how fast (or slow) things run on Linux is way more straightforward. You can drop a small eBPF program onto the exact kernel hook you care about, say, a scheduler event or a specific system call, and start collecting timing data, CPU usage, and I/O stats instantly. There’s no need to recompile the app you’re profiling or juggle heavy monitoring suites; eBPF just latches onto the live system and streams the numbers you want. That means you can keep tabs on any process or service, tweak your code, and see the impact right away. No extra installs, no complicated setup, just fast and flexible performance insights.

### Network Observability

With eBPF we can peek at every network event flying through the system, right from inside the kernel (and even on the NIC before the packet hits the stack). That means we get real-time stats for bandwidth, latency, and connection counts, can spot odd traffic patterns, and trace exactly which process is talking to which host without bolting on big monitoring suites. If some app suddenly starts blasting data to a suspicious server, an eBPF program can flag it and even quarantine the process on the spot. All of this happens with minimal overhead, giving us a handy, always-on microscope for security checks, performance tuning, and quick network troubleshooting.

### Tracing and Profiling

eBPF lets us both trace and profile running code right inside the kernel, giving a detailed view of what each app is doing. With tracing we can hook into functions, system calls, or scheduler events and collect rich context (arguments, timestamps, stack traces) to see exactly where requests slow down or where locks pile up. At the same time, eBPF’s profiling tools sample CPU cycles, memory touches, and even per-process network traffic, so we learn how many resources a single binary really eats instead of just looking at whole-machine averages. Put together, these two abilities help us spot the precise function or query that is the bottleneck, reveal which service is hogging CPU or RAM, and guide smart decisions about tuning, refactoring, or moving workloads.

## Example Use Case Scenario

Problem: We need a way to notice and report suspicious command-line executions that happen inside running containers.

Solution: We write an eBPF program that hooks the system-call paths used to start new commands (for example, with tracepoints and kprobes on execve). Each time a process launches, the program reads the full command path, its arguments, the process ID, the container (cgroup) ID, and a timestamp. It compares the command and its arguments to simple security rules so we avoid false alerts. If the run looks risky and it's inside a container, eBPF passes all that information to a small user-space helper for logging, alerts, or deeper checks. Because eBPF lives in the kernel, we get this real-time view without rebuilding or redeploying the container image.

## Resources

- Liz Rice, Learning eBPF: Programming the Linux Kernel for Enhanced Observability, Networking and Security (O’Reilly, 2023)
- [eBPF Documentation](https://docs.ebpf.io/)
- [eBPF (Wikipedia)](https://en.wikipedia.org/wiki/EBPF)
- [What is eBPF? – eBPF.io](https://ebpf.io/what-is-ebpf/)
- [Understanding eBPF – IBM Think Blog](https://www.ibm.com/think/topics/ebpf)
- [eBPF Guide – Datadog](https://www.datadoghq.com/knowledge-center/ebpf/)
- [eBPF for Security – Aqua Security Academy](https://www.aquasec.com/cloud-native-academy/devsecops/ebpf-linux/)