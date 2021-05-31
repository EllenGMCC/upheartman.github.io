---
title: Pipeline
categories: 
- 读书笔记
- Redis开发与运维
---

Round Trip Time（RTT，命令往返时间）

Redis提供了批量操作命令（例如mget、mset等），有效地节约RTT

但大部分命令是不支持批量操作的，例如要执行n次hgetall命令，并没有 mhgetall命令存在，需要消耗n次RTT

redis-cli的`--pipe`选项实际上就是使用Pipeline机制，例如下面操作将`set hello world`和`incr counter`两条命令组装：

```
echo -en '*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n*2\r\n$4\r\nincr\r\n$7\r\ncounter\r\n' | redis-cli --pipe
```

但大部分开发人员更倾向于使用高级语言客户端中的Pipeline，目前大部分Redis客户端都支持Pipeline

Pipeline执行速度一般比逐条执行要快

客户端和服务端的网络延时越大，Pipeline的效果越明显

**原生批量命令与Pipeline对比：**

* 原生批量命令是原子的，Pipeline是非原子的

* 原生批量命令是一个命令对应多个key，Pipeline支持多个命令

* 原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端的共同实现

# 最佳实践

Pipeline虽然好用，但是每次Pipeline组装的命令个数不能没有节制，否则一次组装Pipeline数据量过大，一方面会增加客户端的等待时间，另一方 面会造成一定的网络阻塞，可以将一次包含大量命令的Pipeline拆分成多次较小的Pipeline来完成

Pipeline只能操作一个Redis实例，但是即使在分布式Redis场景中，也可以作为批量操作的重要优化手段