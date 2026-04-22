# wonagotest

为什么需要加锁？
Locking is needed because shared data may be accessed at the same time by multiple execution paths.

spinlock短临界区短，适合少量操作和频繁的数据
临界区就是访问共享数据、因此需要保护的那一小段代码。


RCU vs Spinlock
RCU is a Linux concurrency mechanism for read-mostly data. Readers can access data with very little overhead. Writers make a copy, update it, replace the pointer, and free the old version only after all old readers have finished.
For a dropped-packet counter in a network driver, RCU is not a good choice. This variable is updated very often and is just a single counter, not a read-mostly structure. Between RCU and spinlock, spinlock is better because it is simpler and more suitable for protecting such a shared variable.

mismatch of lock and unlock
在这类驱动中，spin_lock() 与 spin_unlock() 的出现次数不相等，通常不是 bug 本身。原因是锁 API 有多种配对形式，例如 spin_lock_irqsave() 要配 spin_unlock_irqrestore()，而不是普通 spin_unlock()。有些锁还可能在不同函数或错误处理路径中释放。
因此，单纯按文本计数不能判断正确性；必须检查每条控制路径是否最终释放锁。若有遗漏，代码可能死锁，可靠性下降。
更可靠的写法是统一错误出口，用 goto out_unlock 处理解锁，并用 lockdep_assert_held() 做运行时检查。


The numbers of spin_lock() and spin_unlock() calls may not match because Linux spinlocks have several paired forms. For example, spin_lock_irqsave() is paired with spin_unlock_irqrestore(), not plain spin_unlock(). Also, a lock may be taken in one function and released in a helper or error path.
So simple text counting does not prove a bug. Reliability depends on whether every control path releases the lock correctly. If one path returns without unlocking, the driver may deadlock or block progress.
A better design is to use one cleanup path such as goto out_unlock, and add lockdep_assert_held() checks to catch locking mistakes.

tree vs trie, 
tree 是一种通用层次结构，节点通常按大小关系或父子关系组织，例如二叉搜索树可按数值大小查找和排序。
trie 是一种特殊树结构，它不是直接比较整体大小，而是按 key 的每一位、每个字符或每一段逐级查找，所以很适合前缀匹配。
在 Linux 内核里，trie 常用于路由或地址查找，因为 IP 地址有前缀特性。
如果目标是按 packet length 排序，tree 更好，因为长度是普通数值，直接比较大小即可；trie 不擅长这种简单数值排序，结构会更复杂。


A tree is a general hierarchical structure. Its nodes are usually organised by size relationship or parent-child relationship, and structures like binary search trees are good for ordered lookup and sorting.
A trie is a special tree that works by following parts of a key step by step, such as bits, characters, or prefixes, rather than comparing whole values directly.
In the Linux kernel, tries are useful for prefix-based lookup such as IP routing.
For sorting packets by packet length, a tree is the better choice because packet length is just a numeric value, so ordering by size is simple and direct. A trie is better for prefix matching, not simple numeric sorting.



1）延迟中断处理
NET_RX_SOFTIRQ + net_rx_action()
说明不是在硬中断里做完整收包，而是推迟到 softirq

2）轮询替代中断风暴
n->poll(n, weight)
表示进入 NAPI 轮询模式，一次批量处理一批包

3）预算控制
weight 和 budget
单设备、全局两级限额

4）公平性
多个 NAPI 在链表里轮流处理
没做完的进 repoll，而不是一直霸占 CPU

5）延迟控制
time_limit
避免软中断一直跑

6）每 CPU 本地化
this_cpu_ptr(&softnet_data)
每个 CPU 管自己的收包队列，减少跨核竞争

7）并发保护和 race 避免
test_bit(NAPI_STATE_SCHED, &n->state)
加上 lock/unlock，保证只有该处理的时候才处理

CMP is a control and signalling protocol, not a normal data transport protocol. Typical uses include Echo Request/Reply for ping, Destination Unreachable for reporting network, host, or port failures, Time Exceeded for TTL expiry, Redirect for better routing, and Parameter Problem for header errors.
In the Linux kernel, ICMP is implemented as part of the IP stack. After IP receives a packet, ICMP packets are passed to ICMP handling code; when the kernel detects forwarding or delivery errors, it builds and sends ICMP error messages back to the source host. Linux also keeps ICMP statistics and uses ICMP for mechanisms such as Path MTU Discovery.

This line means:
the kernel looks at the ICMP packet type, uses that type as an index into the icmp_pointers[] table, finds the corresponding handler function, calls it with the packet skb, and stores the return value in success.
