Linux Networking and Kernel Revision Notes
1. Locking and Shared Data
Why locking is needed

Locking is needed because shared data may be accessed at the same time by multiple execution paths.

Spinlock

A spinlock is a simple lock used in kernel code, especially in atomic contexts where sleeping is not allowed.

RCU vs Spinlock

RCU is a Linux concurrency mechanism for read-mostly data. Readers can access data with very little overhead. Writers make a copy, update it, replace the pointer, and free the old version only after all old readers have finished.

For a dropped-packet counter in a network driver, RCU is not a good choice. This variable is updated very often and is just a single counter, not a read-mostly structure.

Between RCU and spinlock, spinlock is better because:

it is simpler
it is more suitable for protecting a small shared variable
it matches frequent update workloads better
2. Lock/Unlock Mismatch
Why simple counting is not enough

The numbers of spin_lock() and spin_unlock() calls may not match because Linux spinlocks have several paired forms.

For example:

spin_lock_irqsave() is paired with spin_unlock_irqrestore()
not with plain spin_unlock()

Also, a lock may be taken in one function and released in:

a helper function
an error path
a cleanup path

So simple text counting does not prove a bug.

What really matters

Reliability depends on whether every control path releases the lock correctly.

If one path returns without unlocking, the driver may:

deadlock
block progress
hang part of the system
Better design

A better design is to:

use one cleanup path such as goto out_unlock
add lockdep_assert_held() checks to catch mistakes
3. Tree vs Trie
Tree

A tree is a general hierarchical structure. Its nodes are usually organised by:

size relationship
parent-child relationship

Structures like binary search trees are good for:

ordered lookup
sorting
Trie

A trie is a special tree that works by following parts of a key step by step, such as:

bits
characters
prefixes

It does not compare whole values directly in the same way as a normal search tree.

In the Linux kernel

Tries are useful for prefix-based lookup, such as:

IP routing
longest-prefix matching
Which is better for packet length sorting?

For sorting packets by packet length, a tree is better because packet length is just a numeric value, so ordering by size is simple and direct.

A trie is better for prefix matching, not simple numeric sorting.

4. ICMP Overview
What ICMP is

ICMP is a control and signalling protocol, not a normal data transport protocol.

Typical uses

Typical uses include:

Echo Request / Reply for ping
Destination Unreachable for reporting network, host, or port failures
Time Exceeded for TTL expiry
Redirect for better routing
Parameter Problem for header errors
ICMP in Linux

In the Linux kernel, ICMP is implemented as part of the IP stack.

After IP receives a packet:

ICMP packets are passed to ICMP handling code

When the kernel detects forwarding or delivery errors:

it builds and sends ICMP error messages back to the source host

Linux also:

keeps ICMP statistics
uses ICMP for mechanisms such as Path MTU Discovery
5. Reading ICMP Dispatch Code
Meaning of the dispatch line

This line means:

The kernel looks at the ICMP packet type, uses that type as an index into the icmp_pointers[] table, finds the corresponding handler function, calls it with the packet skb, and stores the return value in success.

Exam point

This shows:

table-driven dispatch
modular protocol handling
type-based function selection
skb_network_offset()

Purpose: Returns the byte offset of the network-layer header from skb->data
Stack area: sk_buff helper
Exam point: It tells where the L3 header starts in the current data area.

skb_pagelen()

Purpose: Calculates total packet length including linear and paged parts
Stack area: sk_buff helper
Exam point: Not just the linear buffer; it must include paged fragments.

netif_rx()

Purpose: Hands a received packet from the driver into the kernel receive path
Stack area: Receive entry
Exam point: It is the driver-to-stack handoff, often leading to backlog or softirq processing.

napi_poll()

Purpose: Polls a network device and processes a batch of packets under a budget
Stack area: Receive / NAPI
Exam point: Shows polling, budget control, fewer interrupts, and the throughput-latency trade-off.

ipv6_rcv()

Purpose: Processes an incoming packet at the IPv6 layer
Stack area: IPv6 receive
Exam point: Basic validation first, then handoff to later IPv6 processing.

icmp_rcv()

Purpose: Processes a received ICMP packet
Stack area: ICMP receive
Exam point: Usually dispatches according to ICMP type.

icmp_send()

Purpose: Builds and sends an ICMP reply or error message
Stack area: ICMP transmit
Exam point: Creates a response and sends it through the IPv4 output path.

dev_hard_start_xmit()

Purpose: Hands a packet to the NIC driver for transmission
Stack area: Device transmit
Exam point: Key device-layer transmit interface below qdisc and protocol code.

skb_gso_segment()

Purpose: Splits a large packet into smaller segments for transmission
Stack area: Transmit optimisation
Exam point: Shows segmentation/offload design and performance optimisation.

cbs_init()

Purpose: Initialises a Constant Bandwidth Server scheduler instance
Stack area: Scheduling / qdisc
Exam point: State setup, configuration, and scheduler modularity.

counter_validate()

Purpose: Validates packet counter / anti-replay state in WireGuard receive processing
Stack area: Secure receive path
Exam point: Fast validation and replay protection.

corkscrew_interrupt()

Purpose: Interrupt service routine for an older 3Com NIC driver
Stack area: Driver / ISR
Exam point: Minimal urgent work in ISR, then defer the rest.

vti6_xmit()

Purpose: Transmits a packet through an IPv6 tunnel
Stack area: Tunnel transmit
Exam point: Encapsulation plus handoff to the normal output path.

udp_compress()

Purpose: Compresses a UDP header for 6LoWPAN
Stack area: Low-power networking
Exam point: Saves bytes for constrained links.

kmalloc(..., GFP_KERNEL)

Purpose: Normal kernel allocation that may sleep
Stack area: Memory management
Exam point: Wrong for interrupt context because it may block.

kmalloc(..., GFP_ATOMIC)

Purpose: Atomic allocation that must not sleep
Stack area: Memory management
Exam point: Suitable for ISR / softirq, but with tighter memory constraints.

spin_lock() / spin_unlock()

Purpose: Acquire and release a spinlock
Stack area: Concurrency
Exam point: Used in atomic contexts; an apparent mismatch may come from branches or wrappers.

host/network byte-order helpers

Purpose: Convert between host and network endianness
Stack area: Basic helper logic
Exam point: Endianness conversion, not protocol logic.

skbuff.h inline block

Purpose: Moves headers and updates pointers/lengths inside skb
Stack area: sk_buff core operations
Exam point: This is the basis of header push/pull/put logic.

loopback code block

Purpose: Handles packet transmission/reception on the loopback interface
Stack area: Special device path
Exam point: The packet stays inside the host but still uses the common network stack design.

netfilter/core.c block

Purpose: Registers or runs packet-processing hooks
Stack area: Filtering framework
Exam point: Hook-based extensibility in the kernel.

ip_fragment.c block

Purpose: Handles fragment queueing or packet reassembly
Stack area: IPv4 reassembly
Exam point: Queue state, overlap detection, and completion checks.

ip_output.c block

Purpose: Handles part of the IPv4 transmit path
Stack area: IPv4 output
Exam point: Output-stage processing before lower-layer transmit.

dev.c block

Purpose: Implements generic network device receive/transmit logic
Stack area: Core networking / device layer
Exam point: Often used to show stack layering and deferral mechanisms.

br_input.c block

Purpose: Handles incoming packets in the Linux bridge code
Stack area: Layer 2 bridging
Exam point: L2 forwarding/bridging, not Layer 3 routing.
Mistake 1: only describing the function name

Students often describe only the function name, not the stack position.

Mistake 2: being too vague

They say only “this function processes packets”, which is too vague.

Mistake 3: ignoring context

They ignore what comes before and after the function.

Mistake 4: weak sk_buff answers

In sk_buff questions, they forget to mention:

offsets
pointer movement
linear vs paged data
8. Useful Exam Sentence Template
English answer template

This function belongs to the ____ stage of the Linux networking stack.
It usually takes ____ as input and mainly performs ____.
After this, the packet or state is typically passed to ____ for further processing.
So its main role is to provide ____ between ____ and ____.

9. cbs_init() and Network Stack Design Principles
Main idea

The code shows several important Linux network stack design principles.

Principles shown in cbs_init()
1. Fail-fast validation

It rejects missing parameters immediately, which avoids invalid setup.

2. Modularity

It creates a default pfifo child qdisc instead of reimplementing queue storage.

3. Encapsulation

CBS-specific state is stored in private qdisc data rather than exposed globally.

4. Abstraction

It accesses the device and transmit queue through helper functions, not direct hardware-specific code.

5. Polymorphism

It uses enqueue and dequeue function pointers, so the same framework can support different implementations such as software or offload paths.

6. Safe concurrency

It protects shared global structures with a spinlock.

7. Registration-based design

It registers the qdisc in kernel data structures, showing standard object registration and lookup.

8. Deferred control

It uses a watchdog timer, which reflects deferred, timer-based control instead of busy waiting.

Summary sentence

Overall, the code shows modular, reusable, synchronized, and hardware-independent kernel design.

Linux Network Stack Design Principles
10. Core Design Principles
Layering

Each layer should do its own job and hide internal details from other layers.

Zero-copy

A packet should stay in the same memory area as much as possible, to avoid repeated copying.

Layering vs Zero-copy Trade-off

Strict layering improves abstraction, but zero-copy often requires cross-layer knowledge, so the two goals conflict.

Linear buffer for speed

Linux prefers one contiguous buffer because it is faster to process than jumping through linked fragments.

Reserve headroom and tailroom

Extra space is reserved in advance so headers and trailers can be added later without moving the payload.

Modularity

Functions and components should be separated so parts can be added, removed, or reused more easily.

Loadable modules

New kernel functionality can be inserted into a running system without rebuilding the whole kernel.

Reuse rather than rewrite

A good design lets you replace one layer or component without rewriting the whole stack.

Scalability

A design for a small network may not work for a huge network, so protocols must scale with network size.

Abstraction for scale

Large systems often keep only summary information, because full detail would be too expensive to store and process.

Backward compatibility

Old systems and existing code often have to keep working, even if that prevents a cleaner redesign.

Complement rather than replace

A new mechanism is often better when it works with the existing stack instead of completely replacing it.

Performance-aware design

Network design must consider:

CPU caches
packet size
multicore behaviour
not just protocol logic.
Longer packets improve efficiency

Longer packets reduce per-packet overhead, so they usually give better throughput than short packets.

Processor affinity

Keeping related flows on the same CPU core improves cache hits and increases throughput.

11. Fast Revision Version
Concurrency
Spinlock: simple lock for shared data in atomic context
RCU: good for read-mostly structures, not for a frequently updated counter
Lock correctness
Lock/unlock counts may look mismatched because APIs are paired in different forms
What matters is whether every path releases the lock
Data structures
Tree: good for ordered numeric lookup and sorting
Trie: good for prefix lookup, such as IP routing
ICMP
Control/signalling protocol
Used for ping, error reporting, TTL expiry, redirects, and header problems
Implemented inside the Linux IP stack
Code reading
Always explain:
stack position
input
main action
next stage
sk_buff
Focus on:
offsets
pointer movement
header push/pull
linear vs paged data
Design principles
layering
modularity
abstraction
zero-copy
scalability
backward compatibility
deferred processing
performance-aware design

functions————topics:
Low latency / real-time → PREEMPT_RT, interrupt latency, QoS, small buffers, CPU affinity
High throughput → zero-copy, sk_buff design, NAPI, XDP, queue steering, large packets
Security / filtering → Netfilter, eBPF, XDP
Extensibility → layering, protocol handlers, loadable modules, Netlink
Scalability → OSPF/BGP, summarisation, multicore steering
Read-mostly concurrency → RCU
Compatibility → backward compatibility, complement-not-replace design


PREEMPT_RT should not be enabled by default. Linux 6.12 includes mainline PREEMPT_RT support, so it is available without relying on an out-of-tree RT kernel.
It should be enabled only if the SBC has real-time requirements, such as motor control, industrial control, robotics, or real-time audio, where low and predictable latency matters.
For general embedded workloads, it is usually better left disabled, because RT features can add overhead and may reduce overall throughput.

musician perform:
Standard Linux is not a strict real-time operating system, so it is not ideal by default for orchestra-level synchronisation.
The kernel should be improved with PREEMPT_RT, real-time priorities, and threaded interrupts to reduce scheduling delay.
Audio packets should be given the highest priority, with video below them and background traffic last, because timely sound matters more than perfect image quality.
NIC drivers, interrupt handling, softirq processing, NAPI behaviour, and CPU affinity should be tuned to reduce packet-path latency.
Audio and video buffers should be kept as small as possible to reduce end-to-end delay, but not so small that jitter causes glitches or loss of sync.
Accurate timestamping and clock synchronisation are also required, so that all participants play and hear at the correct time.
The central server must efficiently handle more than seventy simultaneous media streams, or queueing delay will become a bottleneck.
Therefore Linux could be used, but only with heavy tuning, and even then Internet propagation delay may still prevent perfect real-time orchestral performance.

self-drive Car
This question is about Linux suitability for safety-critical embedded control, and about the networking properties needed inside a self-driving car.

Linux is a good choice for many in-car computing tasks because it has strong driver support, good networking, modular design, and a mature software ecosystem.
However, standard Linux is not a strict real-time operating system, so it may not be ideal for the most safety-critical control loops such as braking or steering.
A realistic design would use Linux for higher-level functions such as planning, logging, communication, and sensor fusion, while the most time-critical control is handled by a dedicated RTOS or separate controller.
In-car networking should provide low latency, low jitter, high reliability, traffic prioritisation, fault tolerance, and deterministic timing.
These properties are important because control messages must arrive on time and in the correct order, even under heavy load.
The Linux network stack supports modularity, prioritisation, driver support, scalability, and performance tuning quite well.
It can be improved with real-time patches, CPU affinity, queue tuning, and other optimisations.
But it is still mainly a general-purpose best-effort stack, so strict hard real-time guarantees are weaker than in a dedicated automotive RTOS.
Therefore, Linux is useful in a self-driving car, but it is best used as part of a mixed architecture rather than as the only control platform.

RCU VS Spinlock in memory-mapped
A locking mechanism is usually needed, especially for writes and for any read-modify-write access. This register represents shared hardware state, so concurrent access could cause races or lost updates. RCU would not be a good choice, because it is designed for read-mostly kernel memory structures using copy-and-replace, not for directly protecting memory-mapped device registers. A spinlock is more effective because register access is short, needs immediate mutual exclusion, and may occur in interrupt context. A simple read may not always need locking if it is purely observational, but in practice a shared spinlock should protect both read and write paths.

SoftISR High speed internet NAPI Polling less work in ISR but do it afrerwards
Linux handles this by deferring non-urgent interrupt work into softirq queues. The hardware ISR does only minimal urgent work, queues the rest, and exits quickly. The queued work is then run soon afterwards at kernel checkpoints such as interrupt return, system-call exit, scheduler activity, and timer/jiffy events via do_softirq().

This is efficient, but not fully robust on modern high-speed hardware: pure interrupt-driven softirq handling can suffer from heavy interrupt load, cache disruption, and scheduling overhead. Linux improves this with NAPI, mixing interrupts with polling under load.


SMP Cache hit
SMP (Symmetric Multiprocessing) uses two or more equal processors sharing the same main memory and I/O system. Each CPU can run tasks independently under one operating system. Typically, each processor has a private cache connected through a shared bus or interconnect to common main memory.
A cache hit means the required data is found in the processor’s cache, so main memory does not need to be accessed. Caches improve SMP performance because cache access is much faster than RAM access, reducing memory latency and shared-bus traffic. This lets multiple processors run in parallel more efficiently, though cache coherence is needed.

