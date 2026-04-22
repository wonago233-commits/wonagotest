# wonagotest


Locking is needed because shared data may be accessed at the same time by multiple execution paths.

spinlock

RCU vs Spinlock
RCU is a Linux concurrency mechanism for read-mostly data. Readers can access data with very little overhead. Writers make a copy, update it, replace the pointer, and free the old version only after all old readers have finished.
For a dropped-packet counter in a network driver, RCU is not a good choice. This variable is updated very often and is just a single counter, not a read-mostly structure. Between RCU and spinlock, spinlock is better because it is simpler and more suitable for protecting such a shared variable.

mismatch of lock and unlock

The numbers of spin_lock() and spin_unlock() calls may not match because Linux spinlocks have several paired forms. For example, spin_lock_irqsave() is paired with spin_unlock_irqrestore(), not plain spin_unlock(). Also, a lock may be taken in one function and released in a helper or error path.
So simple text counting does not prove a bug. Reliability depends on whether every control path releases the lock correctly. If one path returns without unlocking, the driver may deadlock or block progress.
A better design is to use one cleanup path such as goto out_unlock, and add lockdep_assert_held() checks to catch locking mistakes.

tree vs trie, 
A tree is a general hierarchical structure. Its nodes are usually organised by size relationship or parent-child relationship, and structures like binary search trees are good for ordered lookup and sorting.
A trie is a special tree that works by following parts of a key step by step, such as bits, characters, or prefixes, rather than comparing whole values directly.
In the Linux kernel, tries are useful for prefix-based lookup such as IP routing.
For sorting packets by packet length, a tree is the better choice because packet length is just a numeric value, so ordering by size is simple and direct. A trie is better for prefix matching, not simple numeric sorting.



CMP is a control and signalling protocol, not a normal data transport protocol. Typical uses include Echo Request/Reply for ping, Destination Unreachable for reporting network, host, or port failures, Time Exceeded for TTL expiry, Redirect for better routing, and Parameter Problem for header errors.
In the Linux kernel, ICMP is implemented as part of the IP stack. After IP receives a packet, ICMP packets are passed to ICMP handling code; when the kernel detects forwarding or delivery errors, it builds and sends ICMP error messages back to the source host. Linux also keeps ICMP statistics and uses ICMP for mechanisms such as Path MTU Discovery.

This line means:
the kernel looks at the ICMP packet type, uses that type as an index into the icmp_pointers[] table, finds the corresponding handler function, calls it with the packet skb, and stores the return value in success.


Function / code object	One-line purpose	Stack area	What to emphasise in an exam answer
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
host/network byte-order helpers	Convert between host and network endianness	Basic helper logic	Endianness conversion, not protocol logic
skbuff.h inline block	Moves headers and updates pointers/lengths inside skb	sk_buff core operations	This is the basis of header push/pull/put logic
loopback code block	Handles packet transmission/reception on the loopback interface	Special device path	Packet stays inside host but still uses the common network stack design
netfilter/core.c block	Registers or runs packet-processing hooks	Filtering framework	Hook-based extensibility in the kernel
ip_fragment.c block	Handles fragment queueing or packet reassembly	IPv4 reassembly	Queue state, overlap detection, completion checks
ip_output.c block	Handles part of the IPv4 transmit path	IPv4 output	Output-stage processing before lower-layer transmit
dev.c block	Implements generic network device receive/transmit logic	Core networking / device layer	Often used to show stack layering and deferral mechanisms
br_input.c block	Handles incoming packets in the Linux bridge code	Layer 2 bridging	L2 forwarding/bridging, not Layer 3 routing


Common mistakes in these questions

First, students often describe only the function name, not the stack position.
Second, they say only “this function processes packets”, which is too vague.
Third, they ignore what comes before and after the function.
Fourth, in sk_buff questions, they forget to mention offsets, pointer movement, or linear vs paged data.

Useful exam sentence template

English answer template:

This function belongs to the ____ stage of the Linux networking stack.
It usually takes ____ as input and mainly performs ____.
After this, the packet or state is typically passed to ____ for further processing.
So its main role is to provide ____ between ____ and ____.

If you want, I can next turn this into a shorter revision sheet in the format:

function name + stack layer + caller + next step + typical exam point.

cbs_init() principles of network stack design
The code shows several important Linux network stack design principles. First, it uses fail-fast validation by rejecting missing parameters immediately, which avoids invalid setup. Second, it shows modularity by creating a default pfifo child qdisc instead of reimplementing queue storage. Third, it uses encapsulation, since CBS-specific state is stored in private qdisc data rather than exposed globally. Fourth, it shows abstraction by accessing the device and transmit queue through helper functions, not direct hardware-specific code. Fifth, it uses polymorphism through enqueue and dequeue function pointers, so the same framework can support different implementations such as software or offload paths. Sixth, it demonstrates safe concurrency by protecting shared global structures with a spinlock. Seventh, it registers the qdisc in kernel data structures, showing standard object registration and lookup. Finally, it uses a watchdog timer, which reflects deferred, timer-based control instead of busy waiting. Overall, the code shows modular, reusable, synchronized, and hardware-independent kernel design


===Design Principle
Layering
Each layer should do its own job and hide internal details from other layers.
Zero-copy
A packet should stay in the same memory area as much as possible, to avoid repeated copying.
Layering vs zero-copy trade-off
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
Network design must consider CPU caches, packet size, and multicore behavior, not just protocol logic.
Longer packets improve efficiency
Longer packets reduce per-packet overhead, so they usually give better throughput than short packets.
Processor affinity
Keeping related flows on the same CPU core improves cache hits and increases throughput..
