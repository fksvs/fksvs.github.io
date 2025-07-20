+++
date = '2025-07-20T18:19:49+03:00'
title = 'Your First eBPF Program'
description = 'Your First eBPF Program'
categories = ['ebpf', 'linux']
+++


If this is your first time with interacting eBPF, I strongly suggest you to read [What is eBPF](https://ebpf.io/what-is-ebpf/) or [Introduction to eBPF first](https://fksvs.github.io/posts/introduction-to-ebpf/). After that, you can read [Setting up eBPF Development Environment](https://fksvs.github.io/posts/setting-up-ebpf-dev-env/) and start your eBPF journey.

In this article, we are going to learn what tracepoints are, how tracepoints works, development process of a eBPF project, writing eBPF program and loading it with different techniques. Tighten you seatbelts, we are going into a very technical trip. 
<!--more-->

## What is Tracepoint

Tracepoints are a type of eBPF program that attach to predefined static points within the Linux kernel. These tracepoints are compiled directly into the kernel and are strategically chosen to expose meaningful information about important kernel events, such as system calls or scheduler actions. By hooking into these points, developers can observe low-level system behavior without modifying the kernel itself. Tracepoint-based eBPF programs are classified under the `BPF_PROG_TYPE_TRACEPOINT` program type, making them powerful tools for observability and performance analysis.

### Getting Information About Tracepoints

To explore available tracepoints on your system, you can list them using `sudo ls /sys/kernel/debug/tracing/events/` or by inspecting the file `/sys/kernel/tracing/available_events`. Alternatively, tools like `bpftrace` offer a convenient way to list them with the command `sudo bpftrace -l tracepoint:*`. 

When attaching an eBPF program to a tracepoint, you use the `SEC()` macro with the format `"tracepoint/<category>/<name>"` or the shorthand `"tp/<category>/<name>"`. Both forms are functionally equivalent, so for example `tracepoint/syscalls/sys_enter_execve` and `tp/syscalls/sys_enter_execve` refer to the same tracepoint. To find the tracepoint corresponding to a specific system call (if one exists), you can search using:

```bash
sudo cat /sys/kernel/tracing/available_events | grep <syscall name>  
```

Each tracepoint also has a `format` file located under `/sys/kernel/debug/tracing/events/<category>/<name>/format`, which contains detailed information about the arguments and structure of the tracepoint. For example, to inspect the `sys_enter_execve` tracepoint, view:

```bash
cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/format  
```

### How do They Work ?

A tracepoint in the Linux kernel can be either enabled (on) or disabled (off). When a tracepoint is off, it has no effect on system behavior and introduces zero overhead. However, when it is enabled, any eBPF program attached to that tracepoint is executed each time the kernel hits that code path. The eBPF program runs in kernel context and, once it completes its execution, control returns back to the original kernel function that triggered the tracepoint.
## The eBPF Program

We’re going to use the `sys_enter_execve` tracepoint for our first eBPF program. This tracepoint is a great entry point because it gives us insight into process execution on the system. Why is this important? Because every time a new process is created (e.g., `bash`, `python`, `curl`), the `execve` syscall is used. By hooking into this, we can see what’s being executed which is incredibly useful for auditing, security monitoring, or even debugging. To stream the collected data from kernel space to user space, we’ll use a ring buffer, a modern and efficient eBPF map type designed for high-speed event streaming.

```C
#include <stdint.h>          // Standard integer definitions (e.g., uint64_t)
#include <linux/types.h>     // Linux-specific types (e.g., __u32, __u64)
#include <linux/bpf.h>       // eBPF map and program type definitions (e.g., BPF_MAP_TYPE_RINGBUF)
#include <bpf/bpf_helpers.h> // Helper functions provided by eBPF (e.g., bpf_get_current_pid_tgid)
#include <bpf/bpf_tracing.h> // For working with tracepoints
#include <linux/limits.h>    // For PATH_MAX (maximum file path length)

// Define a ring buffer map to transfer events from kernel space to user space
// This map will live in the ".maps" ELF section and is required for event streaming
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF); // Specify that this is a ring buffer map
    __uint(max_entries, 32768);         // Maximum size (in bytes) of the buffer
} exec_ringbuf SEC(".maps");            // SEC macro tells the loader this is a BPF map

// A license declaration is mandatory for eBPF programs
// "Dual BSD/GPL" ensures compatibility with all helper functions
char LICENSE[] SEC("license") = "Dual BSD/GPL";

// Define the structure of the event we will send to user space
// This includes metadata (pid, uid, timestamp) and the executed filename
struct event_t {
    int pid;                          // Process ID
    int uid;                          // User ID
    long int timestamp;               // Timestamp in nanoseconds
    char filename[PATH_MAX];          // Executed filename (full path)
};

// This struct matches the tracepoint argument format for sys_enter_execve
// To find the exact layout of any syscall tracepoint, inspect:
// /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/format
struct sys_enter_execve_args {
    char _[16];         // Padding for internal tracepoint fields (not used)
    uint64_t filename_ptr; // Pointer to the filename string
    uint64_t argv;         // Pointer to argv
    uint64_t envp;         // Pointer to envp
};

// Define the eBPF program that will run on the tracepoint "sys_enter_execve"
// This tracepoint is triggered whenever a process calls execve()
SEC("tracepoint/syscalls/sys_enter_execve")
int trace_execve(struct sys_enter_execve_args *ctx)
{
    struct event_t *event;

    // Reserve space in the ring buffer for our event
    // If the reservation fails (e.g., buffer is full), exit early
    event = bpf_ringbuf_reserve(&exec_ringbuf, sizeof(*event), 0);
    if (!event) {
        return 0;
    }

    // Capture metadata: current process ID, user ID, and timestamp
    event->pid = bpf_get_current_pid_tgid() >> 32; // Upper 32 bits = PID
    event->uid = bpf_get_current_uid_gid() >> 32;  // Upper 32 bits = UID
    event->timestamp = bpf_ktime_get_ns();         // Nanosecond-resolution timestamp

    // Read the filename pointer from tracepoint arguments
    char *filename_ptr = (char *)ctx->filename_ptr;

    // Safely copy the filename string from user space into our event struct
    // If the read fails (e.g., invalid pointer), discard the event and return
    if (bpf_probe_read_user_str(event->filename, PATH_MAX, filename_ptr) < 0) {
        bpf_ringbuf_discard(event, 0); // Release reserved space in the ringbuf
        return -1;
    }

    // Print to kernel debug trace buffer for quick testing/logging
    // You can view this with: sudo cat /sys/kernel/debug/tracing/trace_pipe
    bpf_printk("%d %d %s\n", event->pid, event->uid, event->filename);

    // Submit the event to the ring buffer so it can be consumed in user space
    bpf_ringbuf_submit(event, 0);

    return 0;
}
```

Save this file as `execve_prog.bpf.c` in your working directory. To compile the BPF program, use `clang` with BPF as the target:

```bash
$ clang -O2 -c -g -target bpf -o execve_prog.o execve_prog.bpf.c
```

- `-O2` enables optimizations
- `-g` includes debug info
- `-target bpf` ensures that clang emits eBPF bytecode
- `-c` compiles without linking
- `-o execve_prog.o` sets the output file

After compiling, you can inspect the generated BPF object file using:

```bash
$ file execve_prog.o
```

You should see output similar to:

```Bash
execve_prog.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info, not stripped
```

This means your code successfully compiled to a relocatable BPF object file, ready to be loaded with a loader like `bpftool`, `libbpf`, or a custom user-space program.


## Loading, Attaching, and Tracing with Your eBPF Program

Once you’ve compiled your eBPF program into an object file (`execve_prog.o`), the next step is to load it into the kernel and attach it to the proper tracepoint. We’ll use the  `bpftool` utility for this.

### Loading & Attaching with `bpftool`

You can load and automatically attach your eBPF program like this:

```bash
sudo bpftool prog load execve_prog.o /sys/fs/bpf/execve_prog autoattach
```

Here’s what this does:

- Loads the object file to the BPF filesystem at `/sys/fs/bpf/execve_prog`
- Automatically attaches it to the correct tracepoint (`sys_enter_execve` in our case), because `autoattach` matches the program section.

### Verifying the Loaded Program

To confirm that your program was loaded successfully:

```bash
sudo bpftool prog show
```

You’ll see output similar to:

```text
307: tracepoint  name trace_execve  tag eec1f73a84dba016  gpl
     loaded_at 2025-07-20T16:49:00+0300  uid 0
     xlated 344B  jited 198B  memlock 4096B  map_ids 107,109
     btf_id 403
```

This tells you:

- Program ID: `307`
- Program type: `tracepoint`
- Section name: `trace_execve`
- Associated map IDs: `107`, `109`

### Viewing Your eBPF Map

You can also list all loaded maps (including your ring buffer) with:

```bash
sudo bpftool map show
```

Expected output:

```text
107: ringbuf  name exec_ringbuf  flags 0x0
     key 0B  value 0B  max_entries 32768  memlock 45464B
```

This confirms that your ring buffer map (`exec_ringbuf`) is active and ready to stream data.

### Debug Output: Viewing `bpf_printk()` Logs

Since ring buffers require a user-space reader (which we haven’t implemented yet), we used `bpf_printk()` for simple logging.

To view these logs, use:

```bash
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

This file streams real-time trace output from the kernel. You should see lines like:

```text
    (sd-worker)-20644   [007] ...21  9115.174417: bpf_trace_printk: 20644 0 /usr/lib/systemd/systemd-nsresourcework
    (sd-worker)-20645   [015] ...21  9115.176158: bpf_trace_printk: 20645 0 /usr/lib/systemd/systemd-nsresourcework
    (sd-worker)-20646   [006] ...21  9115.177218: bpf_trace_printk: 20646 0 /usr/lib/systemd/systemd-nsresourcework
```

These logs show the PID, UID, and path of every process invoking `execve()`.

## What's Next ?

Congratulations! You’ve just taken your first real step into the world of eBPF. But writing C code and loading it into the kernel is just the beginning. eBPF is a _means_, not the end. It’s a powerful tool to extract meaningful data from kernel space. The real magic happens when you start processing, analyzing, and acting on that data.

From here, you can build tools for security monitoring, performance profiling, or network visibility. Whether you're tracking process executions, system call latency, or packet-level traffic, eBPF gives you deep, low-overhead insight into what’s really happening on your system.

## Resources

- [eBPF Documentation](https://docs.ebpf.io/)
- [eBPF (Wikipedia)](https://en.wikipedia.org/wiki/EBPF)