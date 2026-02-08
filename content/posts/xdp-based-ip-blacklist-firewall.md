+++
date = '2026-02-08T20:21:30+01:00'
title = 'XDP Based IP Blacklist Firewall'
description = 'XDP Based IP Blacklist Firewall'
categories = ['xdp', 'network', 'ebpf', 'linux']
+++

IP blacklist firewalls exist to solve a very old and very practical problem: known bad traffic should not consume system resources. In most real environments, a large <!--more--> portion of unwanted traffic originates from repeat offenders. These sources are often identifiable by IP address or network range and tend to reappear.

The idea behind an IP blacklist firewall is actually very simple:  if the source is already known to be malicious, there is no reason to let its packets travel deeper into our system.

This approach does not aim to detect new attacks or analyze payloads. Its goal is early rejection, reducing noise and protecting downstream components.

## Why Traditional Firewalls Fall Short

Traditional Linux firewalls such as iptables or nftables operate relatively late in the networking stack.

By the time a packet reaches these layers:

- The kernel has already allocated memory for it
- The packet has been parsed and wrapped in kernel data structures
- In many cases, connection tracking logic has already been involved

Under low traffic, this overhead is acceptable. Under high traffic or scanning activity, it becomes a scalability problem. The system spends a lot of memory and CPU cycles processing packets that are ultimately dropped.

This is not a design flaw of course. These firewalls are built for flexibility and correctness, not for dropping traffic at line rate.

## What is XDP ?

XDP (eXpress Data Path) is a Linux kernel technology that allows running eBPF programs at the earliest possible point in packet reception, often directly in the network driver.

Instead of waiting for packets to traverse the networking stack, XDP processes them before socket buffers are allocated. This enables extremely low latency decisions, predictable performance under load and early packet drops with minimal CPU overhead

XDP programs are intentionally limited in scope. They are designed to make fast, simple decisions, not complex stateful inspections.

## Key Map Type: LPM TRIE

An IP blacklist is not just a list of individual IP addresses. In practice, it often includes network ranges. To support this efficiently, I used an LPM (Longest Prefix Match) trie map.

The LPM trie allows storing CIDR prefixes (e.g. `/32`, `/24`, `/16`), performing prefix-based lookups and automatically selecting the most specific match.

This is the same logic used in routing tables and fits naturally with firewall policies that need to block both individual hosts and entire subnets.

## Architecture Overview

<p align="center"> <img src="/images/siper-architecture.png" width="700" alt="Siper Firewall Architecture"><br> <em>Siper Firewall Architecture</em> </p>

The architecture is intentionally split into control plane and data plane to keep packet processing fast while allowing flexible policy management.

In user space, a JSON-based blacklist is managed through the Siper CLI with `add` and `del` commands. These commands update the blacklist file, while the `run` command loads and attaches the XDP program and then populates a pinned LPM trie eBPF map named `ipv4_lpm_map`. This pinned map is the “live policy” in the kernel and can be updated at runtime without reloading the XDP program.

A rule in the blacklist file is simple: an ID, a CIDR, metadata, and an enabled flag:

```json
{
 "version": "1",
 "created_at": "2026-01-30 23:22:17.574697932 +0000 UTC",
 "updated_at": "2026-01-30 23:22:17.574734841 +0000 UTC",
 "rules": [
  {
   "id": "01KG8KDTK67512H0H5RMJGCZTA",
   "cidr": "10.0.0.2/32",
   "family": "ipv4",
   "enabled": true,
   "source": "Manual",
   "comment": "Malicious IP address",
   "created_at": "2026-01-30 23:22:17.574732587 +0000 UTC"
  }
 ]
}
```

On the kernel side, the blacklist is stored in an LPM trie map. This choice is deliberate: it allows blocking both exact IPs and subnets efficiently using longest-prefix-match, without doing complex policy logic in the datapath.

```c
struct ipv4_lpm_key {
 __u32 prefixlen;
 __u32 data;
};

struct {
 __uint(type, BPF_MAP_TYPE_LPM_TRIE);
 __type(key, struct ipv4_lpm_key);
 __type(value, __u32);
 __uint(map_flags, BPF_F_NO_PREALLOC);
 __uint(max_entries, 65535);
} ipv4_lpm_map SEC(".maps");
```

The Siper CLI loads the compiled object, attaches XDP to the chosen interface, and pins the program and maps so they persist and can be accessed later:

```go
if err := netlink.LinkSetXdpFdWithFlags(l, fd, 0); err != nil {
  if err2 := netlink.LinkSetXdpFdWithFlags(l, fd, int(unix.XDP_FLAGS_SKB_MODE)); err2 != nil {
    return nil, fmt.Errorf("attach native: %v; attach skb fallback: %w", err, err2)
  }
}

if err := objs.IPv4LpmMap.Pin(IPv4LpmMapPinPath); err != nil {
 return nil, err
}

if err := objs.MetricsMap.Pin(MetricsMapPinPath); err != nil {
 return nil, err
}

if err := objs.XDPSiperFirewall.Pin(ProgramPinPath); err != nil {
 return nil, err
}
```

Once attached, `run` reads the blacklist and turns each CIDR into an LPM key (prefix length + masked network address) before updating the pinned map:

```go
func (objs *SiperObjs) AddCidr(key *IPv4LpmKey) error {
 var value uint32 = 1

 m, err := ebpf.LoadPinnedMap(IPv4LpmMapPinPath, nil)
 if err != nil {
  return err
 }
 defer m.Close()

 err = m.Update(key, value, ebpf.UpdateAny)
 if err != nil {
  return err
 }

 return nil
}
```

In kernel space, the XDP program `xdp_siper_firewall` runs at the network driver level, extracts the source IP address from each incoming packet, and performs a longest-prefix-match lookup against the LPM trie. If the source matches a blacklisted CIDR, the packet is dropped immediately; otherwise it is passed to the Linux network stack.

```c
__u32 saddr = bpf_ntohl(iph->saddr);
int *blocked = map_lookup(saddr);
if (blocked && *blocked) {
 metrics_inc(METRICS_DROP, pkt_len);
 return XDP_DROP;
}

metrics_inc(METRICS_PASS, pkt_len);
return XDP_PASS;
```

At the same time, the datapath updates a dedicated per-CPU metrics map so visibility doesn’t require logging in the hot path. The CLI later reads and aggregates these counters:

```c
struct datarec {
__u64 packets;
__u64 bytes;
}; 

#define METRICS_PASS 0
#define METRICS_DROP 1

struct {
 __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
 __type(key, __u32);
 __type(value, struct datarec);
 __uint(max_entries, 2);
} metrics_map SEC(".maps");
```

### **Packet flow and processing**

1. A network packet arrives at the interface and is handed to the network driver.
2. The attached XDP program runs before the packet enters the Linux networking stack.
3. The program performs bounds checks and parses Ethernet + IPv4 headers.
4. The source IP is extracted from the IPv4 header.
5. The program performs a longest-prefix-match lookup in `ipv4_lpm_map`.
6. If a matching CIDR exists, it returns `XDP_DROP`.
7. If no match exists, it returns `XDP_PASS` and the packet continues into the normal network stack.
8. In both cases, packet/byte counters are updated in `metrics_map`.
9. The CLI reads metrics from the pinned map and prints pass/drop totals.

## What I Learned

This project reinforced several important lessons for me:

- XDP (eBPF) programming requires strict discipline. Unsafe assumptions are rejected by the verifier
- Endianness matters everywhere and mistakes are silent but deadly
- Map choice is a design decision, not an implementation detail

Working at this level changed the way I think about networking and performance.

## Limitations

The current implementation has deliberate limitations of course:

- IPv4 only
- No rate limiting
- No logging in the data path
- No allowlist or priority logic

These are conscious trade-offs to keep the data path minimal and fast. The project is focused on early packet rejection, not full-featured firewall behavior. Maybe I will add these features.