---
title: MESI
categories: 
- 并发编程
---

**原文：** https://juejin.cn/post/6878226618318323726

# CPU高速缓存

CPU在摩尔定律的指导下以每18个月翻一番的速度在发展，然而内存和硬盘的发展速度远远不及CPU。

这就造成了高性能能的内存和硬盘价格及其昂贵。然而CPU的高度运算需要高速的数据。为了解决这个问题，CPU厂商在CPU中内置了少量的高速缓存以解决I\O速度和CPU运算速度之间的不匹配问题

这个CPU缓存即高速缓冲存储器，是位于CPU与主内存间的一种容量较小但速度很高的存储器。

由于CPU的速度远高于主内存，CPU直接从内存中存取数据要等待一定时间周期，Cache中保存着 CPU刚用过或循环使用的一部分数据，当CPU再次使用该部分数据时可从Cache中直接调用，减少CPU的等待时间，提高了系统的效率

**目前流行的多级缓存结构**

现代CPU为了提升执行效率，减少CPU与内存的交互，一般在CPU上集成了多级缓存架构，常见的为三级缓存结构

> 一级Cache(L1 Cache) 二级Cache(L2 Cache) 三级Cache(L3 Cache)

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18a7a9e82c8c4dbd83b9b781dfd52d02~tplv-k3u1fbpfcp-zoom-1.image)

其中：存储器存储空间大小：内存>L3>L2>L1>寄存器；

存储器速度快慢排序：寄存器>L1>L2>L3>内存；

还有一点值得注意的是：缓存是由最小的存储区块-缓存行(cacheline)组成，缓存行大小通常为64byte。

> 缓存行是什么意思呢？ 比如你的L1缓存大小是512kb,而cacheline = 64byte,那么就是L1里有512 * 1024/64个cacheline

# MESI缓存一致性协议

多核CPU的情况下有多个一级缓存，如何保证缓存内部数据的一致,不让系统数据混乱。 这里就引出了一个一致性的协议MESI。

> MESI (Modified-Exclusive-Shared-Invalid)协议是一种广为使用的缓存一致性协议， x86处理器所使用的缓存一致性协议就是基于MESI协议的

MESI 是指4个状态的首字母。每个Cache line有4个状态，可用2个bit表示，它们分别是：

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae45f1bba4644b4cb5b3fa5151c6daa6~tplv-k3u1fbpfcp-zoom-1.image)

## 多核缓存协同操作

假设有三个CPU A、B、C，对应三个缓存分别是cache a、b、 c。在主内存中定义了x的引用值为0

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1e3038c05a3468a8b8d1963381c0e89~tplv-k3u1fbpfcp-zoom-1.image)

## 单核读取

那么执行流程是：

> CPU A发出了一条指令，从主内存中读取x。 从主内存通过bus读取到缓存中（远端读取Remote read）,这是该Cache line修改为E状态（独享）

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6d6200e2e9a42238b3ab55d3a664d33~tplv-k3u1fbpfcp-zoom-1.image)

## 双核读取

那么执行流程是：

> CPU A发出了一条指令，从主内存中读取x。 CPU A从主内存通过bus读取到 cache a中并将该cache line 设置为E状态。 CPU B发出了一条指令，从主内存中读取x。 CPU B试图从主内存中读取x时，CPU A检测到了地址冲突。这时CPU A对相关数据做出响应。此时x 存储于cache a和cache b中，x在chche a和cache b中都被设置为S状态(共享)。

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d760e46904d348359835b2d1b3b561e5~tplv-k3u1fbpfcp-zoom-1.image)

## 修改数据

那么执行流程是：

> CPU A 计算完成后发指令需要修改x. CPU A 将x设置为M状态（修改）并通知缓存了x的CPU B, CPU B将本地cache b中的x设置为I状态(无效) CPU A 对x进行赋值。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e46e8391e42645efae63e394815f6877~tplv-k3u1fbpfcp-zoom-1.image)

## 同步数据

那么执行流程是：

> CPU B 发出了要读取x的指令。 CPU B 通知CPU A,CPU A将修改后的数据同步到主内存时cache a 修改为E（独享） CPU A同步CPU B的x,将cache a和同步后cache b中的x设置为S状态（共享）。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65b434f29b404024b0b714dcdde2483b~tplv-k3u1fbpfcp-zoom-1.image)

# MESI引入的问题

缓存的一致性消息传递是要时间的，这就使其切换时会产生延迟。当一个缓存被切换状态时其他缓存收到消息完成各自的切换并且发出回应消息这么一长串的时间中CPU都会等待所有缓存响应完成。可能出现的阻塞都会导致各种各样的性能问题和稳定性问题。

**CPU切换状态阻塞解决-存储缓存（Store Bufferes）**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bad4c2b300b5477a806e80745f4fc965~tplv-k3u1fbpfcp-zoom-1.image)

前面说过，MESI协议存在一个问题，就是当CPU0修改当前缓存的共享数据时，需要发送一个消息给其他缓存了相同数据的CPU核心，这个消息传递给其他CPU核心以及收到消息完成各自缓存状态的切换这个过程中，CPU会等待所有缓存响应完成，这样会降低处理器的性能。为了解决这个问题，引入了 StoreBufferes存储缓存。

处理器把需要写入到主内存中的值先写入到存储缓存Store Bufferes中，然后继续去处理其他指令。当所有的CPU核心返回了失效确认时，数据才会被最终提交。但是这种优化又会带来另外的问题。

如果某个CPU尝试将其他CPU占有的共享数据写入到内存，消息提交给store buffer以后，当前CPU继续做其他事情，而如果后面的指令依赖于这个被写入内存的最新数据(由于store buffer还没有写入到内存)，就会产生可见性问题(也就是值还没有更新到内存中,这个时候读取到的共享数据的值是错误的)。

# 总结

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/897ddef3fef14e6e845c032b3cf3fe9c~tplv-k3u1fbpfcp-zoom-1.image)

1.当 CPU1 从主内存中取共享变量 X=2 时，通过上述 CPU读取数据的过程将数据读取到 CPU 缓存 L1 中，此时只有 CPU1 缓存行得到共享变量X，X 的状态则为 E，CPU1 的缓存行会时刻的监听嗅探 bus总线

2.当 CPU2 也从主内存中取共享变量 X=2 时，CPU1 的缓存行会嗅探到CP2 对共享变量X 的操作，此时 CPU1 和 CPU2 的缓存行中 X 的状态会变成 S

3.当 CPU1 对 X 进行+1 操作时，寄存器将 L1 cache 的数据读取，并对 X+1 ,将 X=3 赋值给缓存行 L1 cache，L1 cache 的 X =3 通知 L2 cache，L2 cache 再通知 L3 cache ，最终 CPU 缓存上的 X 都是 3，然后 X=3 将会写到 主内存中，这个回写的过程，CPU1 的 X 状态 会变成 M ，而 CPU2 的缓存行通过 bus 总线嗅探到 主内存X 变量的值改变了，此时 CPU2 的 X 状态就会变成 I ，即该缓存行无效。

4.CPU1 缓存行 X 的值回写到主内存后，CPU1 缓存行的 X 状态又会变成 E

参考：

http://igoro.com/archive/gallery-of-processor-cache-effects/

公众号，欢迎关注！！！！！！

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/636823cf57af4a49bc5836178b22cc8d~tplv-k3u1fbpfcp-zoom-1.image)

