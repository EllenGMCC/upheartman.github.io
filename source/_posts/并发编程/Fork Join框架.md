---
title: Fork/Join框架
categories: 
- 并发编程
---

> ❝
>
> 面试官：你先简单介绍下Fork/Join框架吧
>
> ❞

**「回答：」**

Fork/Join 框架是 JDK7 提供了的一个用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架

其中Fork 就是把一个大任务切分为若干子任务并行的执行，Join 就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算1+2+.....＋10000，可以分割成 10 个子任务，每个子任务分别对 1000 个数进行求和，最终汇总这 10 个子任务的结果。如下图所示：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e51b55c3c4748e99c6a4ddce3e514d2~tplv-k3u1fbpfcp-zoom-1.image)

可以看出，主要就是采用分治任务模型的，这个计算框架里的Fork对应的是分治任务模型里的任务分解，Join对应的是结果合并。

> ❝
>
> 面试官：等等，为什么需要Fork/Join呢，线程池不行么
>
> ❞

**「回答：」**

我们知道，计算机中一个任务一般是由一个线程来处理的，如果此时出现了一个非常耗时的大任务，如果是普通的ThreadPoolExecutor就会出现线程池中只有一个线程正在处理这个大任务而其他线程却空闲着，这会导致CPU负载不均衡，空闲的处理器无法帮助工作，而Fork/Join框架可以根据工作窃取算法从其他线程的任务队列里窃取任务来执行

> ❝
>
> 面试官：工作窃取算法？能详细讲讲这个么
>
> ❞

**「回答：」**

比如我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应，比如A线程负责处理A队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。

> ❝
>
> 面试官：那这样不是存在竞争问题，这样有什么缺点么，以及怎么解决的呢
>
> ❞

**「回答：」**

是的，因为这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程和窃取任务的线程分别从双端队列的的两端拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列

**「我详细讲讲这个双端队列：」**

ForkJoinPool 的每个工作线程都维护着一个双端队列，里面存放的对象是任务。如下图：

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b3890b74ace4b7e9f1d05d7f38150be~tplv-k3u1fbpfcp-zoom-1.image)

这个任务需要通过 ForkJoinPool 来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b71c1f35ab5479697333e3627add6ac~tplv-k3u1fbpfcp-zoom-1.image)

同时在遇到 join() 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。

注意：在既没有自己的任务，也没有可以窃取的任务时，进入休眠。

> ❝
>
> 面试官：嗯嗯，那讲讲怎么使用的吧
>
> ❞

**「回答：」**

我们要使用 ForkJoin 框架，必须首先创建一个 ForkJoin 任务。它提供在任务中执行 fork() 和 join() 操作的机制，通常情况下我们不需要直接继承 ForkJoinTask 类，而只需要继承它的子类，Fork/Join 框架提供了以下两个子类：

RecursiveAction：用于没有返回结果的任务。(比如写数据到磁盘，然后就退出了。 一个RecursiveAction可以把自己的工作分割成更小的几块， 这样它们可以由独立的线程或者CPU执行。 我们可以通过继承来实现一个RecursiveAction)

RecursiveTask ：用于有返回结果的任务。(可以将自己的工作分割为若干更小任务，并将这些子任务的执行合并到一个集体结果)

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a521228737240a5a8683447548387f8~tplv-k3u1fbpfcp-zoom-1.image)

这两个子类都定义了抽象方法compute()，RecursiveAction定义的compute()没有返回值，而RecursiveTask定义的compute()方法是有返回值的。两个子类也是抽象类，需要定义子类去扩展

还有就是fork()方法会异步地执行一个子任务，而join()方法则会阻塞当前线程来等待子任务的执行结果

**「使用举例：」**

```
public class MyTest {
    public static void main(String[] args) {
        // 创建分治任务线程池
        ForkJoinPool fjp = new ForkJoinPool(4);
        // 创建分治任务
        Fibonacci fib = new Fibonacci(30);
        // 启动分治任务
        Integer result = fjp.invoke(fib);
        // 输出结果
        System.out.println(result);
    }

    // 递归任务
    static class Fibonacci extends RecursiveTask<Integer> {
        final int n;
        
        Fibonacci(int n) {
            this.n = n;
        }

        protected Integer compute() {
            if (n <= 1)
                return n;
            Fibonacci f1 = new Fibonacci(n - 1);
            // 创建子任务
            f1.fork();
            Fibonacci f2 = new Fibonacci(n - 2);
            // 等待子任务结果，并合并结果
            return f2.compute() + f1.join();
        }
    }
}
```

ForkJoin框架处理的任务基本都能使用递归处理，比如求斐波那契数列等，但递归算法的缺陷是：

- 1.只用单线程处理，
- 2.递归次数过多时会导致堆栈溢出；

ForkJoin解决了这两个问题，使用多线程并发处理，充分利用计算资源来提高效率，同时避免堆栈溢出发生。

**「同时JDK8的parallelStream()底层使用的是Fork/Join」**

```
@Override
        public <P_IN> O evaluateParallel(PipelineHelper<T> helper,
                                         Spliterator<P_IN> spliterator) {
                                         //调用的ForkJoinTask的invoke方法
            return new FindTask<>(this, helper, spliterator).invoke();
        }
```

> ❝
>
> 面试官：讲了这么多，你知道什么场景适合使用Fork/Join么
>
> ❞

**「回答：」**

我觉得最佳应用场景是多核、多内存的环境、可以分割计算再合并的计算密集型任务， 比如超大数组排序，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时，最好配合使用 ManagedBlocker

**「一些注意点：」**

ForkJoinPool在执行过程中，会创建大量的子任务，导致GC进行垃圾回收，这些是需要注意的

> ❝
>
> 面试官：嗯嗯，Fork/Join的源码你看过多少呢
>
> ❞

**「回答：」**

嗯嗯，我分别讲讲核心方法

先看看构造函数

# 构造函数

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc078dfcea9443719911297ccda814b4~tplv-k3u1fbpfcp-zoom-1.image)

**「重要参数解释」** 1.parallelism：并行度，默认情况下跟我们机器的cpu个数保持一致，使用 `Runtime.getRuntime().availableProcessors()`可以得到我们机器运行时可用的CPU个数。

2.factory：创建新线程的工厂。默认情况下使用`ForkJoinWorkerThreadFactory defaultForkJoinWorkerThreadFactory`。

3.handler：线程异常情况下的处理器，该处理器在线程执行任务时由于某些无法预料到的错误而导致任务线程中断时进行一些处理，默认情况为null

4.mode：这个参数要注意，在ForkJoinPool中，每一个工作线程都有一个独立的任务队列，mode表示工作线程内的任务队列是采用何种方式进行调度，可以是先进先出FIFO，也可以是后进先出LIFO。如果为true，则线程池中的工作线程则使用先进先出方式进行任务调度，默认情况下是false。

> ❝
>
> 面试官：异常处理器？具体说说怎么获取异常信息的吧
>
> ❞

**「回答：」**

ForkJoinTask 在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以 ForkJoinTask 提供了 isCompletedAbnormally() 方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过 ForkJoinTask 的 getException 方法获取异常。示例如下

```
if(task.isCompletedAbnormally()){
   System.out.println(task.getException());
}
```

getException 方法返回 Throwable 对象，如果任务被取消了则返回CancellationException。如果任务没有完成或者没有抛出异常则返回 null

> ❝
>
> 面试官：嗯嗯，你继续吧
>
> ❞

首先要知道的是ForkJoinPool使用数组保存所有的WorkQueue

每个worker有属于自己的WorkQueue，但不是所有的WorkQueue都有对应的worker

```
volatile WorkQueue[] workQueues;     // main registry
```

从队列中获取任务图解：

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1665c072e30e407f98d365e3d6fafbbe~tplv-k3u1fbpfcp-zoom-1.image)

# 核心方法

## Fork方法

fork() 做的工作只有一件事，既是把任务推入当前工作线程的工作队列里。

```
public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
           //push到工作队列workQueue
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
        //非ForkJoinThread 线程）提交过来的任务，被放到工作队列被称为submission queue
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```

**「externalPush」**

添加外部任务到submission队列(WorkQueue[]数组的偶数位置) ![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/730d01dfb82d410799621e56a2f93677~tplv-k3u1fbpfcp-zoom-1.image)

## Join方法

join() 的工作则复杂得多，大概是下面的流程：

1. 检查调用 join() 的线程是否是 ForkJoinThread 线程。如果不是（例如 main 线程），则阻塞当前线程，等待任务完成。如果是，则不阻塞。
2. 查看任务的完成状态，如果已经完成，直接返回结果。
3. 如果任务尚未完成，但处于自己的工作队列内，则完成它。
4. 如果任务已经被其他的工作线程偷走，则窃取这个小偷的工作队列内的任务，执行，以期帮助它早日完成欲 join 的任务。
5. 如果偷走任务的小偷也已经把自己的任务全部做完，正在等待需要 join 的任务时，则找到小偷的小偷，帮助它完成它的任务。
6. 递归地执行第5步。

## Submit方法

```
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
            //提交到工作队列
        externalPush(task);
        return task;
    }
```

ForkJoinPool 自身拥有工作队列，这些工作队列的作用是用来接收由外部线程（非 ForkJoinThread 线程）提交过来的任务，而这些工作队列被称为 submission queue 。

submit() 和 fork() 其实没有本质区别，只是提交对象变成了 submission queue 而已（还有一些同步，初始化的操作）。

submission queue 和其他 work queue 一样，是工作线程”窃取“的对象

> ❝
>
> 面试官：嗯嗯，知道这些够了，这个问题先到这吧
>
> ❞

Fork/Join框架源码细节很复杂，有兴趣的可以去看看执行任务相关方法的具体细节，篇幅有限，暂时先介绍这些

**「看到这了，点个赞再走吧，谢谢」**

公众号，欢迎关注！！！！！！

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f87ff76702e4cd981f17f2c5a80623c~tplv-k3u1fbpfcp-zoom-1.image)