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
6. Important Kernel / Networking Functions
Function / Code Object	One-line Purpose	Stack Area	What to Emphasise in an Exam Answer
skb_network_offset()	Returns the byte offset of the network-layer header from skb->data	sk_buff helper	It tells where the L3 header starts in the current data area
skb_pagelen()	Calculates total packet length including linear and paged parts	sk_buff helper	Not just the linear buffer; must include paged fragments
netif_rx()	Hands a received packet from the driver into the kernel receive path	Receive entry	Driver-to-stack handoff, often leading to backlog / softirq processing
napi_poll()	Polls a network device and processes a batch of packets under a budget	Receive / NAPI	Polling, budget control, fewer interrupts, throughput vs latency balance
ipv6_rcv()	Processes an incoming packet at the IPv6 layer	IPv6 receive	Basic validation and then handoff to later IPv6 processing
icmp_rcv()	Processes a received ICMP packet	ICMP receive	Usually dispatches according to ICMP type
icmp_send()	Builds and sends an ICMP reply or error message	ICMP transmit	Creates a response and then sends it through the IPv4 output path
dev_hard_start_xmit()	Hands a packet to the specific NIC driver for transmission	Device transmit	Key device-layer transmit interface below qdisc/protocol code
skb_gso_segment()	Splits a large packet into smaller segments for transmission	Transmit optimisation	Shows offload/segmentation design and performance optimisation
cbs_init()	Initialises a Constant Bandwidth Server scheduler instance	Scheduling / qdisc	State setup, configuration, scheduler modularity
counter_validate()	Validates packet counter / anti-replay state in WireGuard receive processing	Secure receive path	Fast validation and replay protection
corkscrew_interrupt()	Interrupt service routine for an older 3Com NIC driver	Driver / ISR	Minimal urgent work in ISR, then defer the rest
vti6_xmit()	Transmits a packet through an IPv6 tunnel	Tunnel transmit	Encapsulation plus handoff to the normal output path
udp_compress()	Compresses a UDP header for 6LoWPAN	Low-power networking	Saves bytes for constrained links
kmalloc(..., GFP_KERNEL)	Normal kernel allocation that may sleep	Memory management	Wrong for interrupt context because it may block
kmalloc(..., GFP_ATOMIC)	Atomic allocation that must not sleep	Memory management	Suitable for ISR / softirq, but with tighter memory constraints
spin_lock() / spin_unlock()	Acquire and release a spinlock	Concurrency	Used in atomic contexts; apparent mismatch can come from branches or wrappers
Host/network byte-order helpers	Convert between host and network endianness	Basic helper logic	Endianness conversion, not protocol logic
skbuff.h inline block	Moves headers and updates pointers/lengths inside skb	sk_buff core operations	This is the basis of header push/pull/put logic
Loopback code block	Handles packet transmission/reception on the loopback interface	Special device path	Packet stays inside host but still uses the common network stack design
netfilter/core.c block	Registers or runs packet-processing hooks	Filtering framework	Hook-based extensibility in the kernel
ip_fragment.c block	Handles fragment queueing or packet reassembly	IPv4 reassembly	Queue state, overlap detection, completion checks
ip_output.c block	Handles part of the IPv4 transmit path	IPv4 output	Output-stage processing before lower-layer transmit
dev.c block	Implements generic network device receive/transmit logic	Core networking / device layer	Often used to show stack layering and deferral mechanisms
br_input.c block	Handles incoming packets in the Linux bridge code	Layer 2 bridging	L2 forwarding/bridging, not Layer 3 routing
7. Common Mistakes in Code Questions
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
