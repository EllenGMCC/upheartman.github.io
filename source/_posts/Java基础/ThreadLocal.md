---
title: ThreadLocal
categories: 
- Java基础
---

**原文：** https://juejin.cn/post/6868217883085373454

**「面试中经常被问到ThreadLocal的内容，今天我们来还原下面试现场」**

> ❝
>
> 面试官：看你简历写了熟悉JDK常用类源码，那我问问，先来个简单的，你知道ThreadLocal是什么？
>
> ❞

**「回答：」**

ThreadLocal 提供了线程本地变量的实例，它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本

> ❝
>
> 面试官：ok，那你知道它的应用场景，你在工作中有用到它么
>
> ❞

**「回答：」**

应用场景1.可以使每个线程需要一个独享的对象，每个Thread内有自己的实例副本

比如构造一个线程安全的SimpleDateFormat

```java
/**
 * 生产出线程安全的日期格式化工具
 */
class ThreadSafeFormatter {
    public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
}
```

应用场景2.让每个线程内需要保存全局变量

例如使用拦截器中获取用户信息，可以让不同方法直接使用，避免参数传递的麻烦

```java
class Interceptor {
    public void preHandle(String userId) {
        //模拟查询数据库后获取到的用户信息
        User user = new User("月伴飞鱼 " + userId);
        UserContextHolder.holder.set(user);
        new Service().process();
    }
}

class UserContextHolder {
    public static ThreadLocal<User> holder = new ThreadLocal<>();
}

class User {
    String name;
    public User(String name) {
        this.name = name;
    }
}
```

在一个线程的生命周期内，都通过这个静态ThreadLocal实例的get()方法取得自己set过的那个对象，避免了将这个对象（例如user对象）作为参数传递的麻烦

我在工作中也是使用它来让同一个线程内的不同方法间的全局数据共享

```
对了，Spring中也有使用到ThreadLocal的地方，之前看源码看到过
```

DateTimeContextHolder类，用到了ThreadLocal

```java
public final class DateTimeContextHolder {
    private static final ThreadLocal<DateTimeContext> dateTimeContextHolder = new NamedThreadLocal("DateTimeContext");
```

还有RequestContextHolder类

```java
public abstract class RequestContextHolder {
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder = new NamedThreadLocal("Request attributes");
    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder = new NamedInheritableThreadLocal("Request context");
```

> ❝
>
> 面试官：好的，不错哦，小伙子那我来点难的，你讲讲ThreadLocal的原理吧
>
> ❞

**「回答：」**

每个线程中都有一个ThreadLocalMap数据结构，当执行set方法时，其值是保存在当前线程的threadLocals变量中，当执行set方法中，是从当前线程的threadLocals变量获取

```
//java.lang.Thread#threadLocals
ThreadLocal.ThreadLocalMap threadLocals = null;
```

下面Thread，ThreadLocal以及ThreadLocalMap三者之间的关系

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1aaf15eca42f4024b721d9622852ef52~tplv-k3u1fbpfcp-zoom-1.image)

# ThreadLocalMap类

也就是`ThreadLocal.ThreadLocalMap threadLocals`

ThreadLocalMap类是每个线程Thread类里面的变量，里面最重要的是一个键值对数组`Entry[] table`，可以认为是一个map，键值对：

• 键：这个ThreadLocal

• 值：实际需要的成员变量，比如user或者simpleDateFormat对象

```java
private Entry[] table;
```

## Hash冲突

ThreadLocalMap这里采用的是线性探测法，也就是如果发生冲突，就继续找下一个空位置，而不是用链表拉链，Entry中也没有next字段，所以就不存在链表的情况了

每个ThreadLocal对象都有一个hash值threadLocalHashCode，每初始化一个ThreadLocal对象，hash值就增加一个固定的大小0x61c88647

```java
private static final int HASH_INCREMENT = 0x61c88647;

private final int threadLocalHashCode = nextHashCode();
```

**「在插入过程中，根据ThreadLocal对象的hash值，定位到table中的位置i，过程如下：」**

1、如果当前位置是空的，那么正好，就初始化一个Entry对象放在位置i

2、不巧，位置i已经有Entry对象了，如果这个Entry对象的key正好是即将设置的key，那么重新设置Entry中的value

3、很不巧，位置i的Entry对象，和即将设置的key没关系，那么只能找下一个空位置

这样的话，在get的时候，也会根据ThreadLocal对象的hash值，定位到table中的位置，然后判断该位置Entry对象中的key是否和get的key一致，如果不一致，就判断下一个位置

# 主要方法介绍

**「T initialValue()：初始化」**

```java
protected T initialValue() {
//如果不重写本方法，这个方法会返回null
        return null;
    }
private T setInitialValue() {
//当线程第一次使用get方法访问变量时，将调用此方法
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
        //取出map中属于本ThreadLocal的value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

该方法会返回当前线程对应的“初始值”，这是一个`延迟加载`的方法，只有在调用get的时候，才会触发

当线程第一次使用get方法访问变量时，将调用此方法，除非线程先前调用了set方法，在这种情况下，不会为线程调用本initialValue方法

通常，每个线程最多调用一次此方法，但如果已经调用了remove()后，再调用get()，则可以再次调用此方法

如果不重写本方法，这个方法会返回null。一般使用匿名内部类的方法来重写initialValue()方法，以便在后续使用中可以初始化副本对象。

**「void set(T t)：为这个线程设置一个新值」** 和setInitialValue方法很类似

```java
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

**「T get()：得到这个线程对应的value。如果是首次调用get()，则会调用initialValue来得到这个值」**

```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
        //取出map中属于本ThreadLocal的value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

get方法是先取出当前线程的ThreadLocalMap，然后调用`map.getEntry`方法，把本ThreadLocal的引用作为参数传入，取出map中属于本ThreadLocal的value

**「void remove()：删除对应这个线程的值，代码很简单」**

```java
public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

> ❝
>
> 面试官：ok，ok，好了，好了，那你知道使用ThreadLocal会有哪些问题么，你是怎么解决的
>
> ❞

# 内存泄漏

内存泄漏是指某个对象不再有用，但是占用的内存却不能被回收

我们知道，如果这个对象只被弱引用关联（没有任何强引用关联），那么这个对象就可以被回收 所以弱引用不会阻止GC，因为这个弱引用的机制，ThreadLocalMap的每个Entry都是一个对key的弱引用，同时每个Entry都包含了一个对value的强引用

正常情况下，当线程终止，保存在ThreadLocal里的value会被垃圾回收，因为没有任何强引用了

但是，如果线程不终止（比如线程需要保持很久），那么key对应的value就不能被回收，因为有以下的调用链：`Thread ---> ThreadLocalMap ---> Entry(key为null) ---> value`

因为value和Thread之间还存在这个强引用链路，所以导致value无法回收，就可能会出现OOM

JDK已经考虑到了这个问题，所以在set，remove，rehash方法中会扫描key为null的Entry，并把对应的value设置为null，这样value对象就可以被回收

但是如果一个ThreadLocal不被使用，那么实际上set，remove，rehash方法也不会被调用，如果同时线程又不停止，那么调用链就一直存在，那么就导致了value的内存泄漏

**「如何避免内存泄露」**

调用remove方法，就会删除对应的Entry对象，可以避兔内存泄漏，所以使用完ThreadLocal之后，应该调用remove方法

```java
ThreadLocal<String> localName = new ThreadLocal();
try {
    localName.set("月伴飞鱼");
    // 其它业务逻辑
} finally {
    localName.remove();
}
```

如果在Spring中，可以使用RequestContextHolder，那么就不需要自己维护ThreadLocal，因为自己可能会忘记调用remove()方法等，造成内存泄漏

> ❝
>
> 面试官：可以，还有么？
>
> ❞

## 空指针问题

之前遇到过一个空指针问题

总结：ThreadLocal在进行get之前，必须先set，否则会报空指针异常

```java
public class ThreadLocalNPE {
    ThreadLocal<Long> longThreadLocal = new ThreadLocal<>();
    public void set() {
        longThreadLocal.set(Thread.currentThread().getId());
    }
    //拆装箱问题
    public Long get() {//long：NullPointerException
        Long res = longThreadLocal.get();
        return res;
    }
 
    public static void main(String[] args) {
        ThreadLocalNPE threadLocalNPE = new ThreadLocalNPE();
        System.out.println(threadLocalNPE.get());//NullPointerException
    }
}
```

还有就是如果在每个线程中ThreadLocal.set()进去的东西本来就是多线程共享的同一个对象，比如static对象，那么多个线程的 ThreadLocal.get()取得的还是这个共享对象本身，还是有并发访问问题

> ❝
>
> 面试官：嗯嗯，这个问题先到这吧
>
> ❞

公众号，欢迎关注！！！！！！

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2776acd1e71942f6b869c73d0d2de6c0~tplv-k3u1fbpfcp-zoom-1.image)