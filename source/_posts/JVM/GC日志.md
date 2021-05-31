---
title: GC日志
categories: 
- JVM
---

**原文：** https://juejin.cn/post/6875632328345911303

# 环境准备

这样一个案例程序：

```java
public class Main {

  public static void main(String[] args) {
    byte[] array1 = new byte[4 * 1024 * 1024];
    array1 = null;

    byte[] array2 = new byte[2 * 1024 * 1024];
    byte[] array3 = new byte[2 * 1024 * 1024];
    byte[] array4 = new byte[2 * 1024 * 1024];
    byte[] array5 = new byte[128 * 1024];

    byte[] array6 = new byte[2 * 1024 * 1024];
  }
}
```

我们采用如下参数来运行上述程序：

> ❝
>
> -XX:NewSize=10M -XX:MaxNewSize=10M -XX:InitialHeapSize=20M -XX:MaxHeapSize=20M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15 -XX:PretenureSizeThreshold=3M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
>
> ❞

参数介绍：

```
-XX:NewSize:初始年轻代大小
-XX:MaxNewSize:最大年轻代大小
-XX:InitialHeapSize:定义堆的初始化大小，默认值是物理内存的1/64，其实就是:-Xms
-XX:MaxHeapSize:定义最大堆的大小，默认为物理内存的1/4，其实就是:-Xmx
-XX:SurvivorRatio:Eden区与Survivor区的大小比值
-XX:MaxTenuringThreshold:年轻代对象转换为老年代对象最大年龄值
-XX:PretenureSizeThreshold=3M:对象大小超过3M时直接在老年代分配内存
-XX:+UseParNewGC:使用ParNew收集器
-XX:+UseConcMarkSweepGC:使用CMS收集器
-XX:+PrintGCDetails:GC时打印详细信息
-Xloggc:输出GC日志信息到文件中
```

打印的GC日志情况：

```
0.118: [GC (Allocation Failure) 0.118: [ParNew (promotion failed): 8143K->8713K(9216K), 0.0043061 secs]0.122: [CMS: 8194K->6675K(10240K), 0.0038347 secs] 12239K->6675K(19456K), [Metaspace: 3042K->3042K(1056768K)], 0.0082981 secs] [Times: user=0.03 sys=0.01, real=0.01 secs] 
Heap
 par new generation   total 9216K, used 2213K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  27% used [0x00000007bec00000, 0x00000007bee297c8, 0x00000007bf400000)
  from space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
  to   space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
 concurrent mark-sweep generation total 10240K, used 6675K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3063K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 334K, capacity 388K, committed 512K, reserved 1048576K
```

**「日志详解：」**

GC：表明进行了一次垃圾回收，前面没有Full修饰，表明这是一次Young GC

Allocation Failure：表明本次引起GC的原因是因为在年轻代中没有足够的空间能够存储新的数据了

ParNew：表明本次GC发生在年轻代并且使用的是ParNew垃圾收集器。ParNew是一个Serial收集器的多线程版本，会使用多个CPU和线程完成垃圾收集工作（默认使用的线程数和CPU数相同，可以使用`-XX：ParallelGCThreads`参数限制）

ParNew (promotion failed): 8143K->8713K(9216K) 8143K->8713K(9216K)：单位是KB，三个参数分别为：GC前该内存区域(这里是年轻代)使用容量，GC后该内存区域使用容量，该内存区域总容量。

0.0043061 secs：该内存区域GC耗时，单位是秒

`CMS: 8194K->6675K(10240K), 0.0038347 secs] 12239K->6675K(19456K)` 8194K->6675K(10240K)：GC前该内存区域(这里是老年代)使用容量变化，10240K表示该内存区域总容量， 12239K->6675K(19456K)：三个参数分别为：堆区垃圾回收前的大小，堆区垃圾回收后的大小，堆区总大小

Times: user=0.03 sys=0.01, real=0.01 secs：分别表示用户态耗时，内核态耗时和总耗时

同时可以看到出现了promotion failed，那什么情况下会出现promotion failed？

> ❝
>
> 在进行Young GC时，Survivor Space放不下，对象只能放入老年代，而此时老年代也放不下时会出现
>
> ❞

# 详细介绍

**「接下来详细介绍GC过程：」**

首先我们看如下代码：

```java
byte[] array1 = new byte[4 * 1024 * 1024];
array1 = null;
```

这行代码直接分配了一个4MB的大对象，此时这个对象会直接进入老年代，接着array1不再引用这个对象

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22a9f1d2bf00436596986a0c91942072~tplv-k3u1fbpfcp-zoom-1.image)

接着看下面的代码：

```java
byte[] array2 = new byte[2 * 1024 * 1024];
byte[] array3 = new byte[2 * 1024 * 1024];
byte[] array4 = new byte[2 * 1024 * 1024];
byte[] array5 = new byte[128 * 1024];
```

连续分配了4个数组，其中3个是2MB的数组，1个是128KB的数组，如下图所示，全部会进入Eden区域中

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fb0b395276b446980dfa3aa27ac6f60~tplv-k3u1fbpfcp-zoom-1.image)

接着会执行如下代码：

```java
byte[] array6 = new byte[2 * 1024 * 1024];
```

此时还能放得下2MB的对象吗？不可能了，因为Eden区已经放不下了。因此此时会直接触发一次Young GC。

我们看下面的GC日志：

> ❝
>
> ParNew (promotion failed): 8143K->8713K(9216K), 0.0043061 secs
>
> ❞

这行日志显示了，Eden区原来是有8000多KB的对象，但是回收之后发现一个都回收不掉，因为上述几个数组都被变量引用了一，所以一定会直接把这些存活的对象放入到老年代里去，但是此时老年代里已经有一个4MB的数组了，还能放的下3个2MB的数组和1个128KB的数组吗？

明显是不行的，此时一定会超过老年代的10MB大小。

所以此时我们看cms的gc日志：

> ❝
>
> CMS: 8194K->6675K(10240K), 0.0038347 secs] 12239K->6675K(19456K), [Metaspace: 3042K->3042K(1056768K)], 0.0082981 secs
>
> ❞

大家可以清晰看到，此时执行了CMS垃圾回收器的Full GC，我们知道Full GC其实就是会对老年代进行Old GC， 同时一般会跟一次Young GC关联，还会触发一次元数据区（永久代）的GC。

在CMS Full GC之前，就已经触发过Young GC了，此时大家可以看到此时Young GC就已经有了，接着就是执行针对 老年代的Old GC，也就是如下日志：

> ❝
>
> CMS: 8194K->6675K(10240K), 0.0038347 secs]
>
> ❞

**「这里看到老年代从8MB左右的对象占用，变成了6MB左右的对象占用，这是怎么个过程呢？」**

很简单，一定是在Young GC之后，先把2个2MB的数组放入了老年代，此时要继续放1个2MB的数组和1个128KB的数组到老年代，一定会放不下，所以此时就会触发CMS的Full GC

然后此时就会回收掉其中的一个4MB的数组，因为他已经没人引用了

接着放入进去1个2MB的数组和1个128KB的数组

所以大家再看CMS的垃圾回收日志：CMS: 8194K->6836K(10240K), 0.0049920 secs，他是从回收前的8MB变成了 6MB

最后在CMS Full GC执行完毕之后，其实年轻代的对象都进入了老年代，此时最后一行代码要在年轻代分配2MB的数组就可以成功了

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3deb921745754e30976af192c0dd2951~tplv-k3u1fbpfcp-zoom-1.image)

# 补充知识

为了方便理解上述内容，补充以下知识

**「Young GC触发条件：」**

当年轻代Eden区域满的时候会触发一次Young GC

**「Full GC触发条件：」**

Full GC用于清理整个堆空间。它的触发条件主要有以下几种：

1.显式调用`System.gc`方法(建议JVM触发)。

2.元空间不足

3.年代空间不足，引起Full GC。这种情况比较复杂，有以下几种：

- 大对象直接进入老年代引起，由`-XX:PretenureSizeThreshold`参数定义
- Young GC时，经历过多次Young GC仍存在的对象进入老年代。
- Young GC时，动态对象年龄判定机制会将对象提前转移老年代。年龄从小到大进行累加，当加入某个年龄段后，累加和超过survivor区域`-XX:TargetSurvivorRatio`的时候，从这个年龄段往上的年龄的对象进入老年代
- Young GC时，Eden和From Space区向To Space区复制时，大于To Space区可用内存，会直接把对象转移到老年代

4.JVM的空间分配担保机制可能会触发Full GC：

空间担保分配是指在发生Young GC之前，虚拟机会检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。

如果大于，则此次Young GC是安全的。

如果小于，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。

如果HandlePromotionFailure=true，那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小，如果大于，则尝试进行一次Young GC，但这次Young GC依然是有风险的，失败后会重新发起一次Full gc；如果小于或者HandlePromotionFailure=false，则改为直接进行一次Full GC。

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31eb764b7f254098b9108c491a98a11b~tplv-k3u1fbpfcp-zoom-1.image)

**「GC Easy工具」**

这里推荐一个gceasy(https://gceasy.io)工具，可以上传gc文件，然后他会利用可视化的界面来展现GC情况

公众号，欢迎关注！！！！！！

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/636823cf57af4a49bc5836178b22cc8d~tplv-k3u1fbpfcp-zoom-1.image)

参考：

https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html

深入理解Java虚拟机(第3版)
