---
title: "eBPF Deep Dive: Installing Code in the Kernel, and Where iptables Got Pushed Aside"
date: 2026-10-09
draft: false
tags: ["ebpf", "kubernetes", "networking", "cilium", "calico", "linux", "kernel"]
translationKey: "ebpf-deep-dive"
summary: "How eBPF hooks into the kernel, passes the verifier and JIT, and shares state through maps — and why iptables' linear chains and eBPF's hash maps diverge once a cluster reaches thousands of Pods, seen through Cilium and Calico."
---

As part of Cloud Club's 10th cohort, I spent a few months in a Kubernetes CNI study group going back over the network stack from the ground up. The cluster I was running used VXLAN-based Flannel, so I had already written that structure up in a [previous post]({{< ref "/posts/kubernetes-cni-networking" >}}). At the end of that post, while comparing Calico and Cilium, I wrote one paragraph saying "Cilium bypasses netfilter with eBPF" and moved on. When the study group reached its Cilium week, I ended up digging into eBPF starting from the official docs, and it seemed a waste to let those notes drift away. This post is that one paragraph, unpacked.

The starting point is a small puzzle I ran into on a Cilium cluster. From inside a Pod, I sent curl to a ClusterIP while capturing that address on the same Pod's `eth0`.

```bash
$ kubectl exec -it client -- sh
/ # curl -s http://10.96.45.10/ | head -1
<!DOCTYPE html>
/ # tcpdump -ni eth0 host 10.96.45.10
tcpdump: listening on eth0, link-type EN10MB (Ethernet)
^C
0 packets captured
```

curl succeeded. Yet the ClusterIP never appears, **not even on the Pod's own interface**. On a kube-proxy iptables cluster, the ClusterIP rides on the Pod's `eth0` as-is and only turns into a Pod IP after DNAT in the host's `PREROUTING`. The chains I was used to seeing on the node aren't there either.

```bash
$ iptables -t nat -S | grep KUBE-SVC
$ conntrack -L | grep 10.96.45.10
conntrack v1.4.7: 0 flow entries have been shown.
```

No DNAT rule, and no entry in conntrack for the connection. The packet gets there anyway. `ip route`, `iptables`, `conntrack`, and `tcpdump` **all see nothing**. This post follows that packet to where it is actually handled and lays out the structure of eBPF along the way: where in the kernel eBPF plugs in, how a program gets loaded and what it shares state through, and why iptables chains and eBPF maps behave so differently once a cluster grows to thousands of Pods. Cilium and Calico come along as examples of how that structure is actually used.

Every figure in chapters 1 and 2 is taken from the official [ebpf.io "What is eBPF?"](https://ebpf.io/what-is-ebpf/) document (CC BY 4.0), in the order the document presents them.

---

## 1. The Kernel Before eBPF — You Could Only Hand It Configuration

![The kernel before eBPF](images/ebpf-kernel-arch.png)

This is the kernel drawn without eBPF. Processes enter only through system calls, and packets descend Sockets → TCP/IP → Network Device in order. The **Configuration (sysfs, netlink, procfs)** box in the middle is the only channel through which a user can influence the kernel.

Every tool from the previous post goes into that box. `ip route`, `bridge fdb`, and `iptables` push **data** into kernel data structures over netlink. Flannel's `flanneld`, Calico's Felix and BIRD, all programmed the kernel through this same channel. What they have in common is that **the kernel code stays put; only the data that code reads changes**. The code that looks up the routing table, the code that walks an iptables chain top to bottom, is fixed inside the kernel.

In this structure, changing what the kernel actually does leaves you two options: patch the kernel source and rebuild, or load a kernel module. The first is impractical anywhere you run a distribution kernel, and the second means a bug in one module can take the whole kernel down. You want to insert **logic** at a specific point in the network stack that says "handle this packet like so," and there was no safe way to put anything but data there.

eBPF adds a third option in that spot. It is **a channel for passing code, not data**.

---

## 2. eBPF — A Channel for Passing Code

Where the configuration interface was a channel for writing values into kernel data structures, eBPF is a channel for attaching small programs at predetermined points in the kernel. This chapter walks through what such a program looks like and which gates it passes to get into the kernel.

### 2.1 Hooks — Places to Attach a Program at Each Kernel Junction

![eBPF hook overview](images/ebpf-hook-overview.png)

Same figure as chapter 1, now with eBPF icons attached. **Each icon is one hook.** An eBPF program does not run just anywhere; it attaches only to these hooks the kernel has defined in advance: syscall entry, sockets, TCP/IP, network device, down to the NIC. The icons along the file path on the left belong to tracing and security tools. This post cares about the right-hand column.

| Icon in the figure | Kernel hook | What Cilium uses it for |
|---|---|---|
| Left column (File, Storage) | kprobe, tracepoint, LSM | Tracing and security tools (bpftrace, bcc, Tetragon) |
| Next to Syscall | cgroup socket hooks (`connect`, `sendmsg`, `getpeername`) | Service handling (socket LB) |
| TCP/IP | **tc** (traffic control) ingress / egress | All Pod traffic: policy, forwarding, encapsulation |
| Network Device | **XDP** (eXpress Data Path) | NodePort acceleration |

Chapter 3 looks separately at how these hooks line up against Netfilter's five.

### 2.2 An Event Wakes the Program

![Syscall hook and an eBPF program](images/ebpf-syscall-hook.png)

This figure shows what a program actually looks like. When a process calls `execve()`, the kernel runs the hook's program **first**, before heading to the scheduler. The program is short. It receives a context, pulls the PID and process name out through helper functions, **pushes an event up to userspace** with `perf_submit`, and returns 0.

Three things carry straight over to networking. A program **runs only when an event happens** (a packet arrives on a veth, `connect()` is called). A program **instructs the kernel through its return value**; at a tc hook, the return value is pass, drop, or redirect to another interface. And `perf_submit` is the prototype of Cilium's observability tool, Hubble. Every time the datapath makes a decision about a packet, it exports that decision the same way.

### 2.3 Written in C, Compiled to Bytecode

![Compiling eBPF bytecode with clang](images/ebpf-clang.png)

The document first notes that "most people don't write eBPF directly; they use it indirectly through projects like Cilium, bcc, or bpftrace." Kubernetes operators are that case. Inside those projects, source written in a subset of C is compiled by LLVM into **eBPF bytecode**, which is neither x86 nor ARM. eBPF is a small virtual machine living inside the kernel, and bytecode is that machine's instruction set.

In Cilium, the C source lives in the repository's `bpf/` directory, and the clang shipped in the agent image compiles it **on the node**. Kernel versions and configurations differ from node to node, so it is compiled to fit each one.

### 2.4 The Loader — Into the Kernel Through the bpf() Syscall

![eBPF overview — development and runtime](images/ebpf.png)

The full picture. Bytecode and map definitions go into the **Go Library**, and the library hands them to the kernel through the `bpf()` syscall. Inside the kernel, the program passes the Verifier and the JIT and is attached to a hook, and arrows into eBPF Maps come from both the library and the programs.

The Go Library in this figure is not a metaphor. In Cilium, the thing standing in that spot is `cilium-agent`. It is written in Go and calls `bpf()` through the `cilium/ebpf` library. In the previous post I summarized iptables as "a userspace tool that registers rules with the kernel's Netfilter." The agent is the **userspace tool that registers programs and maps** standing in the same spot. Once registration is done, the agent steps out of the packet path. Even if the agent dies for a while, the programs already attached keep running.

### 2.5 Verifier and JIT — Two Gates Before Going Live

![Loader, verifier, JIT](images/ebpf-loader.png)

Bytecode arriving through `bpf()` does not run immediately. The green tags in the figure are the state changes: **bytecode → approved → x86_64**. It passes through two gates.

The **verifier** statically checks every path before execution. The program must terminate (no unbounded loops), must not touch memory out of bounds, has size and complexity limits, and loading requires root or `CAP_BPF`. This gate is the decisive difference from kernel modules. A module can load any code and can crash the kernel; eBPF admits **only what the kernel has proven safe**. This is also why Cilium is picky about kernel versions. Old verifiers will not accept the size and complexity of Cilium's programs.

The **JIT** (just-in-time compiler) turns the approved bytecode into the machine's native instructions. It runs at the same speed as code that was in the kernel to begin with. The interpreter overhead that the phrase "a virtual machine in the kernel" might suggest is not actually there.

### 2.6 Maps — Storage Shared by the Kernel and Userspace

![eBPF map architecture](images/ebpf-map-architecture.png)

A program's stack disappears when the call ends. State that has to outlive a call goes in a **map**. In the figure, three arrows enter the cylinder: one from userspace, two from programs at different hooks. The userspace loader reads and writes through `bpf()`; programs read and write through helpers. **The agent and the kernel programs, and the programs among themselves, all see the same map.**

Maps come in types, and each type is a different data structure. The types the document lists correspond to Cilium's maps like this.

| Map type | Data structure | In Cilium |
|---|---|---|
| Hash | hash table | `policy` (policy), `lb` (Service → backends) |
| LRU hash | hash that evicts old entries | `ct` (connection tracking) |
| LPM trie | longest-prefix-match trie | `ipcache` (IP → Identity, owning node) |
| Array / per-CPU array | fixed-size array | configuration values, counters |
| Perf / ring buffer | kernel → userspace event queue | Hubble events |

The LPM lookup the routing table used to do **moved into a map with the same data structure**, and the policy decision an iptables chain used to make moved into a hash lookup. Chapter 4 looks at how that difference meets scale.

### 2.7 Helpers and Tail Calls

![Helper function call](images/ebpf-helper.png)

A program cannot call arbitrary kernel functions. It calls only **helper functions the kernel has promised as a stable API**. Even to get a single random number, it uses `bpf_get_prandom_u32()` as in the figure. `bpf_redirect`, which Cilium uses to hand a packet to another interface, `bpf_skb_store_bytes` for rewriting headers, and `bpf_map_lookup_elem` for map lookups are all helpers. Blocking direct calls into kernel internals is what makes it possible for the verifier to prove safety.

![Tail call](images/ebpf-tailcall.png)

A program can also **jump to another program without returning**. That is a **tail call**. It is the means of splitting logic that does not fit in one program under the verifier's size limit across several programs, and Cilium divides the handling of a single Pod's packets across multiple programs this way.

### 2.8 Summary

| Element | Role | In Cilium |
|---|---|---|
| **Program** | Code that runs at a hook. Reads, modifies, forwards, or drops a packet | `bpf_lxc.c`, `bpf_host.c`, `bpf_overlay.c`, `bpf_sock.c` |
| **Map** | Key-value store living in the kernel. Shared among programs and with userspace | `ipcache`, `policy`, `ct`, `lb` |
| **Hook** | The kernel point a program attaches to | tc, cgroup socket, XDP |

In one sentence: **without rebuilding the kernel or loading a module, attach small programs the verifier has proven safe at predetermined kernel junctions, run them on each event, and let those programs and userspace share state through maps.** If the Configuration box of chapter 1 was "a channel for passing data," eBPF is **a channel for passing code**.

![A channel for data and a channel for code](images/kernel-config-vs-ebpf.svg "Before: userspace passes data through netlink/sysfs/procfs. After: userspace passes verified code through bpf()")

---

## 3. Network Hooks — Three More Next to Netfilter's Five

Chapter 2 covered what a hook is; now we look at exactly where the network-path hooks sit. [OS Network Internals]({{< ref "/posts/linux-network-internals" >}}) laid out Netfilter's five hooks (`PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`). The eBPF hooks line up against those five like this.

![Where eBPF network hooks sit relative to Netfilter hooks](images/ebpf-hooks-vs-netfilter.svg "XDP sits before sk_buff allocation, tc ingress before PREROUTING, tc egress after POSTROUTING, and the cgroup socket hook before any packet exists")

| Hook | Position | Order relative to Netfilter |
|---|---|---|
| **XDP** | Right after the NIC driver receives the packet. Before an `sk_buff` even exists | Well before `PREROUTING`. Physical NIC receive only |
| **tc ingress / egress** | Per interface. Ingress just before the packet climbs into the stack, egress just before handing to the driver | Ingress is **before** `PREROUTING`, egress is **after** `POSTROUTING` |
| **cgroup socket** | Inside the `connect()`, `sendmsg()`, `getpeername()` syscalls | **No packet exists yet** |

The position of tc ingress is the key. The tc ingress on the host-side veth is **before a frame leaving the Pod climbs into the host stack**, that is, a point where no bridge lookup, no routing lookup, and no Netfilter has happened yet. If a program here hands the packet straight to another interface with `bpf_redirect`, **everything after it is skipped**. Cilium's official datapath figure shows exactly this.

![Datapath between endpoints on the same node (Cilium docs)](images/cilium-official-datapath-endpoint.svg)

*Source: [Cilium — Life of a Packet](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)*

Two Pods on the same node. A packet leaving the left Endpoint is caught at **TC @ Endpoint** (`bpf_lxc`) and follows the solid line at the bottom **straight to the right Endpoint's `bpf_lxc`**. In the figure, `PREROUTING`, `FORWARD`, and `POSTROUTING` are all **inside dashed boxes**, meaning optional paths taken only when L7 policy is enabled. Netfilter is out of the default path.

Half the reason `conntrack -L` was empty in the opening puzzle is here. The packet never passed through Netfilter, so kernel conntrack never had a chance to see it. The other half is in the cgroup socket hook, covered in chapter 5.

---

## 4. Why Maps Are Fast — Walking a Chain vs. Finding a Hash

Chapter 3 showed a difference in position: eBPF catches packets ahead of Netfilter. But if only the position differed, the advantage would amount to "handles it a bit earlier." The difference you actually feel when Pods grow into the thousands comes not from position but from **data structures**.

### 4.1 An iptables Chain Is Walked Top to Bottom

Recall how kube-proxy's iptables mode implements Services. The `KUBE-SERVICES` chain has one rule per Service; a matching rule jumps to a `KUBE-SVC-*` chain; there, one probability rule per endpoint jumps to a `KUBE-SEP-*` chain; and there the DNAT is applied.

```
-A KUBE-SERVICES -d 10.96.45.10/32 -p tcp --dport 80 -j KUBE-SVC-NGINX
-A KUBE-SERVICES -d 10.96.45.11/32 -p tcp --dport 80 -j KUBE-SVC-API
-A KUBE-SERVICES -d 10.96.45.12/32 -p tcp --dport 5432 -j KUBE-SVC-DB
... (one per Service)
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.33333 -j KUBE-SEP-A
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.50000 -j KUBE-SEP-B
-A KUBE-SVC-NGINX -j KUBE-SEP-C
... (one per endpoint)
```

Netfilter compares the rules in a chain **one by one, top to bottom**. If a packet's destination is on line 1,000 of `KUBE-SERVICES`, it jumps only after 999 comparisons. The number of rules grows in proportion to **Services × endpoints**, and the worst-case matching cost per packet grows with that number. Linear search, O(n).

conntrack hides much of this cost. Only the first packet of a flow walks the chain to the end; once the connection is registered, later packets pass at the single `--ctstate ESTABLISHED` line. So on workloads with many long-lived connections, chain length is hard to feel. The problems are workloads that **open many new connections**, and the update cost in the next section.

### 4.2 Update Cost — The Whole Table Is Rewritten Every Time a Pod Comes Up

Updates hurt more than lookups. iptables is built so that even modifying a single rule means reading the entire table from the kernel, changing it, and **pushing the whole thing back**. So instead of inserting rules one line at a time, kube-proxy rewrites the relevant tables all at once with `iptables-restore` every time an endpoint changes. With thousands of Services, that is tens of thousands of rules, and every time a Pod comes up or goes down, those tens of thousands of lines are replaced wholesale. When hundreds of Pods churn in sequence during a rolling update, updates fall behind updates, and in the gap traffic goes to dead Pods or new Pods receive none. kube-proxy's `minSyncPeriod` option is a buffer that batches these updates to reduce them, and the same problem is why Kubernetes built a new nftables mode.

NetworkPolicy is a bit different. Calico's Felix turns selectors into **ipsets**, leaving rules untouched and changing only the set's members, so there is no cost to rewriting policy rules as Pods come and go. But the chains themselves are still inside Netfilter, and the basis for decisions is still IP. Each node has to maintain the state of which IP belongs to which Pod.

### 4.3 An eBPF Map Finds It Directly by Key

Here is the same job done with eBPF maps.

![Linear walk of an iptables chain vs. constant-time lookup in an eBPF hash map](images/iptables-chain-vs-ebpf-map.svg "Left: a packet walks rule 1..N in KUBE-SERVICES. Right: hash(dst IP:port) lands directly on the backend entry")

A Service lookup is **one hash** in the `lb` map keyed by `(ClusterIP, port, protocol)`. Whether there are 10 Services or 10,000, the cost is the same. A policy lookup is one hash in the `policy` map keyed by `(source Identity, port, protocol)`. The `ipcache` lookup that converts a source IP to an Identity is an LPM trie, so it scales with prefix length, not with the number of entries. **Even with thousands of rules, the cost one packet pays stays the same.**

Updates differ too. Maps are read and written atomically per entry. When a Pod comes up, one line goes into `ipcache` and one backend line into `lb`; the remaining entries are untouched. Since the whole table is never rewritten, updates do not fall behind even when hundreds of Pods churn in sequence.

| | iptables chain | eBPF map |
|---|---|---|
| Lookup | Compare rules top to bottom. **Proportional to rule count** | Hash by key. **Independent of rule count** |
| Update | Rewrite and replace the whole table | Replace one entry atomically |
| 1,000 Pods come up | Thousands of rules added, full rewrite each time | 1,000 entries added to `ipcache` and `lb` |
| What hides lookup cost | conntrack (after the first packet) | Its own `ct` map (same role) |

### 4.4 So When Do You Feel It

To be honest, on a cluster of a few nodes and a few dozen Pods, you cannot feel this difference. With a few hundred iptables rules, linear search is plenty fast and updates do not fall behind. The difference shows when three conditions overlap: Services and Pods in the **thousands**, new connections opened **frequently**, and Pods coming and going **frequently**. Large multi-tenant clusters and clusters flooded with batch workloads fit here.

![Per-packet lookup cost against cluster size](images/lookup-cost-vs-cluster-size.svg "iptables chain cost grows with N; eBPF map cost stays flat")

At that scale, iptables mode's per-packet cost and update cost grow together, while eBPF maps stay flat on both. This is the substance of the one-liner "iptables is O(n), Cilium is O(1)" from the previous post. Because the data structure changed, **the slope of the cost curve** against scale changed; it is not doing the same work a bit faster.

---

## 5. Side Note — How Cilium Uses eBPF

That is the structure of eBPF and the advantage that structure has at scale. This chapter briefly looks at how Cilium actually assembled it. It does not go deep; it only points out where the programs, maps, and hooks from above are used.

### 5.1 Parts

![Cilium architecture (official docs)](images/cilium-arch-official.png)

*Source: [Cilium — Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)*

The official figure has two layers. The Cilium Layer on top holds the daemon (agent) and its surrounding tools, and the daemon builds and attaches the BPF Programs in the Kernel layer below through **Code Generation → Bytecode injection**. Looking at the kernel layer, there is a program **per container, plus one on `eth0`**. It is the ebpf.io figure from section 2.4 redrawn for Cilium.

| Component | Deployment | What it does |
|---|---|---|
| **Agent** (`cilium-agent`) | DaemonSet, one per node | Receives Pod, Service, and policy events and **manages that node's eBPF programs and maps**. The loader from section 2.4 |
| **CNI Plugin** (`cilium-cni`) | Executable, on Pod create/delete | When the runtime calls it, asks the node's agent to "set up the datapath for this Pod" |
| **Operator** (`cilium-operator`) | Deployment, one per cluster | Things that only need doing once per cluster, like IPAM and Identity. **Not in the forwarding path.** Traffic flows even if it dies |
| **Hubble** | Built into the agent + Relay + UI | Turns the decisions the datapath exports through perf buffers into flows and shows them |

Here is what you actually see on a node. A tc program is attached to each Pod's host-side veth.

```bash
$ ip -br link | grep -E 'cilium|lxc'
cilium_net@cilium_host   UP          # veth pair between the host stack and eBPF
cilium_host@cilium_net   UP          # the Pods' gateway IP lives here
lxc1a2b3c4d5e6f@if12     UP          # the Pod's host-side veth — Flannel's veth…, Calico's cali… slot

$ tc filter show dev lxc1a2b3c4d5e6f ingress
filter ... bpf ... cil_from_container direct-action ...
```

### 5.2 The Path a Packet Takes

A packet from Pod A on node-1 to Pod B on node-2 passes through five steps.

1. **The frame leaving the Pod is caught by `bpf_lxc` at the `lxc` tc ingress.** No bridge, no routing table, no iptables has seen it yet
2. **Three maps are consulted.** `ct` (is this a known connection?), `ipcache` (which Identity and which node is the destination IP?), `policy` (may we send to that Identity?). All hash or LPM lookups
3. **It crosses between nodes.** In tunnel mode it is wrapped in VXLAN; in native routing it is left to the host routing table
4. **The receiving node's `bpf_host` takes it, finds the destination Endpoint, and checks that Endpoint's ingress policy.** Again `ipcache` and `policy` lookups
5. **It is pushed straight into the destination Pod's veth.** Neither host routing nor the `FORWARD` chain is involved

In the previous post, Flannel's inter-node path was decided by three lines (route, ARP, FDB), and Calico's by one route learned via BGP. In Cilium's tunnel mode that role moves to **one entry in the `ipcache` map**. Connection tracking is also its own. Once the first packet passes policy, it is recorded in the `ct` map, and later packets and replies on the same connection skip the policy check. What Calico did with a single `--ctstate ESTABLISHED -j ACCEPT` line is done with the same structure in a different store. Cilium's connections are in `cilium bpf ct list`, not `conntrack -L`.

### 5.3 kube-proxy Replacement — Done Before the Packet Exists

The other half of the opening puzzle. curl inside the Pod calls `connect(10.96.45.10:80)`. When the syscall enters the kernel, a program attached to the root cgroup runs **before any packet is created**. The program finds the Service in the `lb` map, picks a backend, and **rewrites the socket's destination address itself to the Pod IP**. After that, the kernel builds an ordinary SYN to the Pod IP. The ClusterIP never existed on the Pod's `eth0` or on any wire.

| | kube-proxy iptables | Cilium socket LB |
|---|---|---|
| When translated | When the first packet reaches `PREROUTING` | Inside `connect()` |
| What is translated | The packet header. Replies reversed by conntrack | The socket's destination. Replies simply come back to the Pod IP |
| Cost per packet | Chain scan on the first packet, conntrack lookup afterward | **Zero.** Once per connection |
| `iptables -t nat` | `KUBE-SVC-*` chains | None. `cilium bpf lb list` |

There is no `KUBE-SVC-*` because there is no kube-proxy, obviously; and `conntrack` is empty because the DNAT finished at the socket stage and Netfilter had nothing to translate. **No translation means nothing to reverse, and nothing to reverse means no need for conntrack.** The application does not know any of this. When it calls `getpeername()`, a program at the same cgroup hook intercepts it and hands back the original ClusterIP.

Some packets have no socket. A packet arriving from outside the cluster on a NodePort has no process that called `connect()`, so the tc program on `eth0` (or XDP) applies **DNAT to the packet header**. The socket LB is the first net and tc is the second.

### 5.4 Identity — Deciding by Number Instead of IP

![Identity-based policy (Cilium docs)](images/cilium-official-identity.png)

*Source: [Cilium — Identity-Based Security](https://docs.cilium.io/en/stable/security/network/identity/)*

The other piece of Cilium's design is that the `policy` map key in chapter 4 holds an Identity, not an IP. The official figure says it all. However many `role=frontend` Pods there are, they are grouped into **one Identity, "frontends"**, and the policy is **one allow arrow** from frontends to backends. Add Pods, and the arrow stays the same.

When a Pod starts, the agent collects and sorts its security-relevant labels (`app=frontend`, namespace, and so on) and assigns that set a single cluster-wide number. It is stored in a `CiliumIdentity` CRD, so **every node sees the same number**.

Policy is stored as relations between these numbers. "Allow frontend → backend 80/TCP" is one line, `(1234, 80, TCP) → allow`, in backend's `policy` map. Scale frontend from 4 Pods to 40 and that line is unchanged; only `ipcache` gains 36 more IP → 1234 lines. On top of the "updates are per-entry" advantage from section 4.3, the policy side is made entirely independent of Pod count.

---

## 6. Side Note — Calico Set Alongside

Having looked at Cilium in chapter 5, let's put Calico in the same spot and compare. The previous post summarized Calico as "a mature architecture stacked on netfilter"; seen through chapter 4, that phrase becomes a bit more concrete.

Calico's default dataplane is iptables. On each node, Felix hooks `cali-FORWARD` into the `filter` table's `FORWARD` chain, branches on interface name, and gives each Pod a `cali-fw-` (from-workload) and `cali-tw-` (to-workload) chain. Policy is evaluated right in front of the Pod's veth. Selectors become ipsets, so rules stay put and only members change, and conntrack passes everything after the first packet. Between nodes it uses one of BGP, IPIP, or VXLAN. Services are left to kube-proxy.

**The strength of this structure is that it stays inside chapter 1's configuration interface.** You can read it to the end with `ip route`, `iptables -S`, `ipset list`, and `conntrack -L`. It does not care about kernel version, and operators debug it with tools they already know. The weaknesses are exactly chapter 4: chains are linear, Service rules are updated by kube-proxy's full rewrites, and because decisions are based on IP, every node has to carry IP → Pod state.

| Aspect | Calico (iptables mode) | Cilium (eBPF) |
|---|---|---|
| **Where packets are handled** | Netfilter `FORWARD` chain | tc ingress, ahead of Netfilter |
| **Policy lookup** | Linear chain walk + ipset membership | `policy` hash map |
| **Policy update** | ipset member change (rules untouched) | Atomic map entry change |
| **Services** | kube-proxy iptables. Full-rewrite updates | `lb` map + socket LB. Per-entry updates |
| **Connection tracking** | Kernel conntrack | Its own `ct` map |
| **Basis for decisions** | IP (ipset per node) | Identity (cluster-wide number) |
| **Between nodes** | BGP / IPIP / VXLAN | VXLAN·Geneve / native routing |
| **Debugging tools** | `iptables`, `ipset`, `conntrack`, `tcpdump` | `cilium bpf *`, `hubble observe`, `bpftool` |
| **Kernel requirement** | Nothing special | 5.10+ (recent Cilium releases) |

One thing must be added to be fair. **Calico also has an eBPF dataplane.** Turn it on and it has the same skeleton as Cilium: tc hooks, its own connection-tracking map, kube-proxy replacement. What differs is the model on top. Calico eBPF **keeps the IP-based policy model and swaps only the dataplane**, so it can fall back to iptables mode; Cilium is **designed around Identity and has no iptables mode**. So the more accurate comparison than "Calico vs. Cilium" is "iptables chains vs. eBPF maps," which is why this post spent the most space on chapter 4.

---

## Closing

This post unpacked the one paragraph the previous post skipped over, "Cilium bypasses netfilter with eBPF." Unpacked, it was three layers. eBPF is **a channel for passing code** in the spot where the configuration interface passed data; that code runs at native speed through the JIT after the verifier proves it safe; and state is shared between kernel and userspace through maps. The network hooks sit ahead of Netfilter (tc) and above it (socket), so when handling finishes there, Netfilter never even gets a chance to see the packet. And because that handling is a hash map lookup, the cost of one packet and the cost of one update stay flat even with thousands of Pods.

Two things struck me most while writing this up. One is that **the structure of the relocated decisions was not unfamiliar**. The `ct` map was conntrack, `ipcache` was the routing table and FDB combined, the `policy` map was Calico's chains, and the `lb` map was the `KUBE-SVC-*` chains. Each data structure from the previous post had a map corresponding to it. What changed is that every lookup is a hash and so indifferent to scale, and that the store is read directly by programs inside the kernel rather than by userspace tools. The other is that **DNAT moved up from the packet to the syscall**. I had accepted conntrack as a precondition of address translation, but finish the translation before a packet exists and conntrack has nothing left to do. The problem was not solved by changing a data structure; it was eliminated by changing the layer where it arises, which is why an empty `conntrack -L` was design, not a defect.

Conversely, it also became clear that this advantage is not free. It is picky about kernel versions, the debugging tools change wholesale, and on a cluster of a few nodes you cannot even feel it. You do not use Cilium because iptables is slow; the criterion is **whether you have reached the scale where iptables' cost curve becomes a problem**. If you are not at that scale, the side you can read to the end with `iptables -S` may be the easier one to operate.

Next I plan to write up the case where a Pod has more than one interface. All of Cilium's decisions hang on the Pod's first veth, so when Multus attaches a second interface, neither policy nor Hubble will see that traffic. I want to use that case to confirm that "where eBPF is attached" is exactly the boundary of "what it can see."

---

### References

- [ebpf.io — What is eBPF?](https://ebpf.io/what-is-ebpf/)
- [Linux kernel — BPF documentation](https://docs.kernel.org/bpf/)
- [cilium/ebpf — Go eBPF library](https://github.com/cilium/ebpf)
- [Cilium — eBPF Datapath](https://docs.cilium.io/en/stable/network/ebpf/)
- [Cilium — Life of a Packet](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)
- [Cilium — Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)
- [Cilium — Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium — Identity-Based Security](https://docs.cilium.io/en/stable/security/network/identity/)
- [Cilium Blog — Why is the kernel community replacing iptables with BPF?](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/)
- [Kubernetes docs — Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [Calico — eBPF dataplane](https://docs.tigera.io/calico/latest/operations/ebpf/)
- My own earlier post — [The Essence of Kubernetes Networking: CNI, VXLAN, and How Pods Communicate]({{< ref "/posts/kubernetes-cni-networking" >}})
