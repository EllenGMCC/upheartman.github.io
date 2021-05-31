---
title: ReentrantLock
categories: 
- 并发编程
---

**原文：** https://mp.weixin.qq.com/s/gC8Uj8otLGnOKN6ryb1uZQ

**个人公众号月伴飞鱼，公众号文章首发，转载请注明出处，谢谢**

ReentrantLock 中文我们叫做可重入互斥锁，可重入的意思是同一个线程可以对同一个共享资源重复的加锁或释放锁，互斥就是 AQS 中的排它锁的意思，只允许一个线程获得锁。

# 简单应用

ReentrantLock 的使用相比较 synchronized 会稍微繁琐一点，所谓显示锁，也就是你在代码中需要主动的去进行 lock 操作。一般来讲我们可以按照下面的方式使用 ReentrantLock

```
Lock lock = new ReentrantLock();
lock.lock();
try {
  doSomething();
}finally {
  lock.unlock();
}
```

`lock.lock ()` 就是在显式的上锁。上锁后，下面的代码块一定要放到 try 中，并且要结合 finally 代码块调用`lock.unlock ()`来释放锁，否则一定 doSomething 方法中出现任何异常，这个锁将永远不会被释放掉。

**公平锁和非公平锁**

synchronized 是非公平锁，也就是说每当锁匙放的时候，所有等待锁的线程并不会按照排队顺去依次获得锁，而是会再次去争抢锁。ReentrantLock 相比较而言更为灵活，它能够支持公平和非公平锁两种形式。只需要在声明的时候传入 true。

```
Lock lock = new ReentrantLock(true);
```

而默认的无参构造方法则会创建非公平锁。

**tryLock方法**

前面我们通过 `lock.lock ();` 来完成加锁，此时加锁操作是阻塞的，直到获取锁才会继续向下进行。ReentrantLock 其实还有更为灵活的枷锁方式 tryLock。

tryLock 方法有两个重载，第一个是无参数的 tryLock 方法，被调用后，该方法会立即返回获取锁的情况。获取为 true，未能获取为 false。我们的代码中可以通过返回的结果进行进一步的处理。第二个是有参数的 tryLock 方法，通过传入时间和单位，来控制等待获取锁的时长。如果超过时间未能获取锁则放回 false，反之返回 true。使用方法如下：

```
if(lock.tryLock(2, TimeUnit.SECONDS)){
   try {
      doSomething();
   } catch (InterruptedException e) {
      e.printStackTrace();
   }finally {
      lock.unlock();
   }
}else{
  doSomethingElse();
}
```

我们如果不希望无法获取锁时一直等待，而是希望能够去做一些其它事情时，可以选择此方式。

# 类结构

ReentrantLock 类本身是不继承 AQS 的，实现了 Lock 接口，如下：

```
public class ReentrantLock implements Lock, java.io.Serializable {}
```

Lock 接口定义了各种加锁，释放锁的方法，接口有如下几个：

```
// 获得锁方法，获取不到锁的线程会到同步队列中阻塞排队
void lock();
// 获取可中断的锁
void lockInterruptibly() throws InterruptedException;
// 尝试获得锁，如果锁空闲，立马返回 true，否则返回 false
boolean tryLock();
// 带有超时等待时间的锁，如果超时时间到了，仍然没有获得锁，返回 false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
// 释放锁
void unlock();
// 得到新的 Condition
Condition newCondition();
```

ReentrantLock 就负责实现这些接口，我们使用时，直接面对的也是这些方法，这些方法的底层实现都是交给 Sync 内部类去实现的，Sync 类的定义如下：

```
abstract static class Sync extends AbstractQueuedSynchronizer {}
```

都是最终继承自 AbstractQueuedSynchronizer。这就是著名的 AQS。通过查看 AQS 的注释我们了解到， AQS 依赖先进先出队列实现了阻塞锁和相关的同步器（信号量、事件等）。

AQS 内部有一个 volatile 类型的 state 属性，实际上多线程对锁的竞争体现在对 state 值写入的竞争。一旦 state 从 0 变为 1，代表有线程已经竞争到锁，那么其它线程则进入等待队列。

等待队列是一个链表结构的 FIFO 队列，这能够确保公平锁的实现。同一线程多次获取锁时，如果之前该线程已经持有锁，那么对 state 再次加 1。释放锁时，则会对 state-1。直到减为 0，才意味着此线程真正释放了锁。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b017582a34254c07a482fa9834d3e418~tplv-k3u1fbpfcp-watermark.image)

Sync 继承了 AbstractQueuedSynchronizer ，所以 Sync 就具有了锁的框架，根据 AQS 的框架，Sync 只需要实现 AQS 预留的几个方法即可，但 Sync 也只是实现了部分方法，还有一些交给子类 NonfairSync 和 FairSync 去实现了，NonfairSync 是非公平锁，FairSync 是公平锁，定义如下：

```
// 同步器 Sync 的两个子类锁
static final class FairSync extends Sync {}
static final class NonfairSync extends Sync {}
```

几个类整体的结构如下：

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa0acd6469594c2b8167c248d2686d41~tplv-k3u1fbpfcp-watermark.image)

图中 Sync、NonfairSync、FairSync 都是静态内部类的方式实现的，这个也符合 AQS 框架定义的实现标准。

# 构造器

ReentrantLock 构造器有两种，代码如下：

```
// 无参数构造器，相当于 ReentrantLock(false)，默认是非公平的
public ReentrantLock() {
    sync = new NonfairSync();
}
 
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

无参构造器默认构造是非公平的锁，有参构造器可以选择。

从构造器中可以看出，公平锁是依靠 FairSync 实现的，非公平锁是依靠 NonfairSync 实现的

# 源码解析

## Sync同步器

Sync 表示同步器，继承了 AQS：

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0a3681bd82b45d4b75860eff29a04e4~tplv-k3u1fbpfcp-watermark.image)

从图中可以看出，lock 方法是个抽象方法，留给 FairSync 和 NonfairSync 两个子类去实现。

## 加锁方法

**FairSync公平锁**

FairSync 公平锁只实现了 lock 和 tryAcquire 两个方法，lock 方法非常简单，如下：

```
// acquire 是 AQS 的方法，表示先尝试获得锁，失败之后进入同步队列阻塞等待
final void lock() {
    acquire(1);
}
```

在 FairSync 并没有重写 acquire 方法代码。调用的为 AbstractQueuedSynchronizer 的代码，如下：

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

首先调用一次 tryAcquire 方法。如果 tryAcquire 方法返回 true，那么 acquire 就会立即返回。

但如果 tryAcquire 返回了 false，那么则会先调用 addWaiter，把当前线程包装成一个等待的 node，加入到等待队列。然后调用 acquireQueued 尝试排队获取锁，如果成功后发现自己被中断过，那么返回 true，导致 selfInterrupt 被触发，这个方里只是调用`Thread.currentThread ().interrupt ();`进行 interrupt。

acquireQueued 代码如下：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54613f62b37a479cbc98e6781869104d~tplv-k3u1fbpfcp-watermark.image)

在此方法中进入自旋，不断查看自己排队的情况。如果轮到自己（ header 是已经获取锁的线程，而 header 后面的线程是排队到要去获取锁的线程），那么调用 tryAcquire 方法去获取锁，然后把自己设置为队列的 header。在自旋中，如果没有排队到自己，还会检查是否应该应该被中断。

**整个获取锁的过程我们可以总结下：**

直接通过 tryAcquire 尝试获取锁，成功直接返回；

如果没能获取成功，那么把自己加入等待队列；

自旋查看自己的排队情况；

如果排队轮到自己，那么尝试通过 tryAcquire 获取锁；

如果没轮到自己，那么回到第三步查看自己的排队情况。

从以上过程我们可以看到锁的获取是通过 tryAcquire 方法。而这个方法在 FairSync 和 NonfairSync 有不同实现。

这个tryAcquire 方法是 AQS 在 acquire 方法中留给子类实现的抽象方法，FairSync 中实现的源码如下：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd0566ddb3e246a1873a085b8cbb4e25~tplv-k3u1fbpfcp-watermark.image)

实际上它的实现和 NonfairSync 的实现，只是在 `c==0` 时，多了对 hasQueuedPredecessors 方法的调用。故名思义，这个方法做的事情就是判断当前线程是否前面还有排队的线程。

当它前面没有排队线程，说明已经排队到自己了，这是才会通过 CAS 的的方式去改变 state 值为 1，如果成功，那么说明当前线程获取锁成功。接下来就是调用 setExclusiveOwnerThread 把自己设置成为锁的拥有者。else if 中逻辑则是在处理重入逻辑，如果当前线程就是锁的拥有者，那么会把 state 加 1 更新回去。

通过以上分析，我们可以看出 AbstractQueuedSynchronizer 提供 acquire 方法的模板逻辑，但其中真正对锁的获取方法 tryAcquire，是在不同子类中实现的，这是很好的设计思想。

**NonfairSync非公平锁**

NonfairSync 底层实现了 lock 和 tryAcquire 两个方法，如下:

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e298c7eea22b4e28934b470f4012e2d6~tplv-k3u1fbpfcp-watermark.image)

**nonfairTryAcquire**

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad1643e6212c4cdea02b10517618226a~tplv-k3u1fbpfcp-watermark.image)

以上代码有三点需要注意：

通过判断 AQS 的 state 的状态来决定是否可以获得锁，0 表示锁是空闲的；

else if 的代码体现了可重入加锁，同一个线程对共享资源重入加锁，底层实现就是把 state + 1，并且可重入的次数是有限制的，为 Integer 的最大值；

这个方法是非公平的，所以只有非公平锁才会用到，公平锁是另外的实现。

无参的 tryLock 方法调用的就是此方法，tryLock 的方法源码如下：

```
public boolean tryLock() {
    // 入参数是 1 表示尝试获得一次锁
    return sync.nonfairTryAcquire(1);
}
```

其底层的调用关系(只是简单表明调用关系，并不是完整分支图)如下：

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01c1b307b4b64ac2bdb7526a0f04b7cf~tplv-k3u1fbpfcp-watermark.image)

## 解锁方法

```
public void unlock() {
    sync.release(1);
}
```

和 lock 很像，实际调用的是 sync 实现类的 release 方法。和 lock 方法一样，这个 release 方法在 AbstractQueuedSynchronizer 中，

```
if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
        unparkSuccessor(h);
    return true;
}
return false;
```

这个方法中会先执行 tryRelease，它的实现也在 AbstractQueuedSynchronizer 的子类 Sync 中，如果释放锁成功，那么则会通过 unparkSuccessor 方法找到队列中第一个 `waitStatus<0`的线程进行唤醒。我们下面看一下 tryRelease 方法代码：

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b78dfe477abe49689545f69b25d307d2~tplv-k3u1fbpfcp-watermark.image)

tryRelease 方法是公平锁和非公平锁都公用的，在锁释放的时候，是没有公平和非公平的说法的。

从代码中可以看到，锁最终被释放的标椎是 state 的状态为 0，在重入加锁的情况下，需要重入解锁相应的次数后，才能最终把锁释放，比如线程 A 对共享资源 B 重入加锁 5 次，那么释放锁的话，也需要释放 5 次之后，才算真正的释放该共享资源了

# 总结

本篇文章 ReentrantLock 的使用及其核心源代码，其实 Lock 相关的代码还有很多。我们可以尝试自己去阅读。

ReentrantLock 的设计思想是通过 FIFO 的队列保存等待锁的线程。通过 volatile 类型的 state 保存锁的持有数量，从而实现了锁的可重入性。而公平锁则是通过判断自己是否排队成功，来决定是否去争抢锁

然后我们了解到AQS 搭建了整个锁架构，子类锁只需要根据场景，实现 AQS 对应的方法即可，不仅仅是 ReentrantLock 是这样，JUC 中的其它锁也都是这样，只要对 AQS 了如指掌，锁其实非常简单