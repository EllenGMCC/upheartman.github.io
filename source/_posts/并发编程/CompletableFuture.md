---
title: CompletableFuture
categories: 
- 并发编程
---

**原文：** https://mp.weixin.qq.com/s/NsX-AKGWjFcr9vbPWrJK3A

# 前言

最近看公司代码，多线程编程用的比较多，其中有对CompletableFuture的使用，所以想写篇文章总结下

在日常的Java8项目开发中，CompletableFuture是很强大的并行开发工具，其语法贴近java8的语法风格，与stream一起使用也能大大增加代码的简洁性

大家可以多应用到工作中，提升接口性能，优化代码

# 基本介绍

CompletableFuture是Java 8新增的一个类，用于异步编程，继承了Future和CompletionStage

这个Future主要具备对请求结果独立处理的功能，CompletionStage用于实现流式处理，实现异步请求的各个阶段组合或链式处理，因此completableFuture能实现整个异步调用接口的扁平化和流式处理，解决原有Future处理一系列链式异步请求时的复杂编码

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc1843fb9c2d49a399a5ffb30b858c6d~tplv-k3u1fbpfcp-zoom-1.image)

**Future的局限性**

1、Future 的结果在非阻塞的情况下，不能执行更进一步的操作

我们知道，使用Future时只能通过isDone()方法判断任务是否完成，或者通过get()方法阻塞线程等待结果返回，它不能非阻塞的情况下，执行更进一步的操作。

2、不能组合多个Future的结果

假设你有多个Future异步任务，你希望最快的任务执行完时，或者所有任务都执行完后，进行一些其他操作

3、多个Future不能组成链式调用

当异步任务之间有依赖关系时，Future不能将一个任务的结果传给另一个异步任务，多个Future无法创建链式的工作流。

4、没有异常处理

**现在使用CompletableFuture能帮助我们完成上面的事情，让我们编写更强大、更优雅的异步程序**

# 基本使用

**创建异步任务**

通常可以使用下面几个CompletableFuture的静态方法创建一个异步任务

```java
public static CompletableFuture<Void> runAsync(Runnable runnable);														//创建无返回值的异步任务
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);					//无返回值，可指定线程池（默认使用ForkJoinPool.commonPool）
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);											//创建有返回值的异步任务
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);	//有返回值，可指定线程池
```

> 使用示例：

```java
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    //do something
}, executor);
```

```java
int poiId = 111;
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
	PoiDTO poi = poiService.loadById(poiId);
  return poi.getName();
});
// Block and get the result of the Future
String poiName = future.get();
```

**使用回调方法**

通过`future.get()`方法获取异步任务的结果，还是会阻塞的等待任务完成

CompletableFuture提供了几个回调方法，可以不阻塞主线程，在异步任务完成后自动执行回调方法中的代码

```java
public CompletableFuture<Void> thenRun(Runnable runnable);									   //无参数、无返回值
public CompletableFuture<Void> thenAccept(Consumer<? super T> action);         //接受参数，无返回值
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn); //接受参数T，有返回值U
```

> 使用示例：

```java
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> "Hello")
  																									.thenRun(() -> System.out.println("do other things. 比如异步打印日志或发送消息"));
//如果只想在一个CompletableFuture任务执行完后，进行一些后续的处理，不需要返回值，那么可以用thenRun回调方法来完成。
//如果主线程不依赖thenRun中的代码执行完成，也不需要使用get()方法阻塞主线程。
```

```java
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> "Hello")
  																									.thenAccept((s) -> System.out.println(s + " world"));
//输出：Hello world
//回调方法希望使用异步任务的结果，并不需要返回值，那么可以使用thenAccept方法
```

```java
CompletableFuture<Boolean> future = CompletableFuture.supplyAsync(() -> {
  PoiDTO poi = poiService.loadById(poiId);
  return poi.getMainCategory();
}).thenApply((s) -> isMainPoi(s));			// boolean isMainPoi(int poiId);

future.get();
//希望将异步任务的结果做进一步处理，并需要返回值，则使用thenApply方法。
//如果主线程要获取回调方法的返回，还是要用get()方法阻塞得到
```

**组合两个异步任务**

```java
//thenCompose方法中的异步任务依赖调用该方法的异步任务
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn);	
//用于两个独立的异步任务都完成的时候
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, 
                                              BiFunction<? super T,? super U,? extends V> fn);	
```

> 使用示例：

```java
CompletableFuture<List<Integer>> poiFuture = CompletableFuture.supplyAsync(
  () -> poiService.queryPoiIds(cityId, poiId)
);
//第二个任务是返回CompletableFuture的异步方法
CompletableFuture<List<DealGroupDTO>> getDeal(List<Integer> poiIds){
  return CompletableFuture.supplyAsync(() ->  poiService.queryPoiIds(poiIds));
}
//thenCompose
CompletableFuture<List<DealGroupDTO>> resultFuture = poiFuture.thenCompose(poiIds -> getDeal(poiIds));
resultFuture.get();
```

thenCompose和thenApply的功能类似，两者区别在于thenCompose接受一个返回`CompletableFuture<U>`的Function，当想从回调方法返回的`CompletableFuture<U>`中直接获取结果U时，就用thenCompose

如果使用thenApply，返回结果resultFuture的类型是`CompletableFuture<CompletableFuture<List<DealGroupDTO>>>`，而不是`CompletableFuture<List<DealGroupDTO>>`

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
  .thenCombine(CompletableFuture.supplyAsync(() -> "world"), (s1, s2) -> s1 + s2);
//future.get()
```

**组合多个CompletableFuture**

当需要多个异步任务都完成时，再进行后续处理，可以使用**allOf**方法

```java
CompletableFuture<Void> poiIDTOFuture = CompletableFuture
	.supplyAsync(() -> poiService.loadPoi(poiId))
  .thenAccept(poi -> {
    model.setModelTitle(poi.getShopName());
    //do more thing
  });

CompletableFuture<Void> productFuture = CompletableFuture
	.supplyAsync(() -> productService.findAllByPoiIdOrderByUpdateTimeDesc(poiId))
  .thenAccept(list -> {
    model.setDefaultCount(list.size());
    model.setMoreDesc("more");
  });
//future3等更多异步任务，这里就不一一写出来了

CompletableFuture.allOf(poiIDTOFuture, productFuture, future3, ...).join();		//allOf组合所有异步任务，并使用join获取结果
```

该方法挺适合C端的业务，通过poiId异步的从多个服务拿门店信息，然后组装成自己需要的模型，最后所有门店信息都填充完后返回

这里使用了join方法获取结果，它和get方法一样阻塞的等待任务完成

多个异步任务有任意一个完成时就返回结果，可以使用**anyOf**方法

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Result of Future 1";
});

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
    }
    return "Result of Future 2";
});

CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
       throw new IllegalStateException(e);
      return "Result of Future 3";
});

CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(future1, future2, future3);

System.out.println(anyOfFuture.get()); // Result of Future 2
```

## 异常处理

```java
Integer age = -1;

CompletableFuture<Void> maturityFuture = CompletableFuture.supplyAsync(() -> {
  if(age < 0) {
    throw new IllegalArgumentException("Age can not be negative");
  }
  if(age > 18) {
    return "Adult";
  } else {
    return "Child";
  }
}).exceptionally(ex -> {
  System.out.println("Oops! We have an exception - " + ex.getMessage());
  return "Unknown!";
}).thenAccept(s -> System.out.print(s));
//Unkown!
```

exceptionally方法可以处理异步任务的异常，在出现异常时，给异步任务链一个从错误中恢复的机会，可以在这里记录异常或返回一个默认值

使用handler方法也可以处理异常，并且无论是否发生异常它都会被调用

```java
Integer age = -1;

CompletableFuture<String> maturityFuture = CompletableFuture.supplyAsync(() -> {
    if(age < 0) {
        throw new IllegalArgumentException("Age can not be negative");
    }
    if(age > 18) {
        return "Adult";
    } else {
        return "Child";
    }
}).handle((res, ex) -> {
    if(ex != null) {
        System.out.println("Oops! We have an exception - " + ex.getMessage());
        return "Unknown!";
    }
    return res;
});
```

# 分片处理

分片和并行处理：分片借助stream实现，然后通过CompletableFuture实现并行执行，最后做数据聚合（其实也是stream的方法）

CompletableFuture并不提供单独的分片api，但可以借助stream的分片聚合功能实现

举个例子：

```java
//请求商品数量过多时，做分批异步处理
List<List<Long>> skuBaseIdsList = ListUtils.partition(skuIdList, 10);//分片
//并行
List<CompletableFuture<List<SkuSales>>> futureList = Lists.newArrayList();
for (List<Long> skuId : skuBaseIdsList) {
  CompletableFuture<List<SkuSales>> tmpFuture = getSkuSales(skuId);
  futureList.add(tmpFuture);
}
//聚合
futureList.stream().map(CompletalbleFuture::join).collent(Collectors.toList());
```

# 举个例子

带大家领略下CompletableFuture异步编程的优势

**这里我们用CompletableFuture实现水泡茶程序**

首先还是需要先完成分工方案，在下面的程序中，我们分了3个任务：

* 任务1负责洗水壶、烧开水

* 任务2负责洗茶壶、洗茶杯和拿茶叶

* 任务3负责泡茶。其中任务3要等待任务1和任务2都完成后才能开始

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4c01417c223423d89bf45828ee24cb2~tplv-k3u1fbpfcp-zoom-1.image)

下面是代码实现，你先略过runAsync()、supplyAsync()、thenCombine()这些不太熟悉的方法，从大局上看，你会发现：

1. 无需手工维护线程，没有繁琐的手工维护线程的工作，给任务分配线程的工作也不需要我们关注；
2. 语义更清晰，例如 `f3 = f1.thenCombine(f2, ()->{})` 能够清晰地表述**任务3要等待任务1和任务2都完成后才能开始**；
3. 代码更简练并且专注于业务逻辑，几乎所有代码都是业务逻辑相关的

```java
//任务1：洗水壶->烧开水
CompletableFuture f1 = 
  CompletableFuture.runAsync(()->{
  System.out.println("T1:洗水壶...");
  sleep(1, TimeUnit.SECONDS);

  System.out.println("T1:烧开水...");
  sleep(15, TimeUnit.SECONDS);
});
//任务2：洗茶壶->洗茶杯->拿茶叶
CompletableFuture f2 = 
  CompletableFuture.supplyAsync(()->{
  System.out.println("T2:洗茶壶...");
  sleep(1, TimeUnit.SECONDS);

  System.out.println("T2:洗茶杯...");
  sleep(2, TimeUnit.SECONDS);

  System.out.println("T2:拿茶叶...");
  sleep(1, TimeUnit.SECONDS);
  return "龙井";
});
//任务3：任务1和任务2完成后执行：泡茶
CompletableFuture f3 = 
  f1.thenCombine(f2, (__, tf)->{
    System.out.println("T1:拿到茶叶:" + tf);
    System.out.println("T1:泡茶...");
    return "上茶:" + tf;
  });
//等待任务3执行结果
System.out.println(f3.join());

void sleep(int t, TimeUnit u) {
  try {
    u.sleep(t);
  }catch(InterruptedException e){}
}
```

```
// 一次执行结果：
T1:洗水壶...
T2:洗茶壶...
T1:烧开水...
T2:洗茶杯...
T2:拿茶叶...
T1:拿到茶叶:龙井
T1:泡茶...
上茶:龙井
```

# 注意事项

**1.CompletableFuture默认线程池是否满足使用**

前面提到创建CompletableFuture异步任务的静态方法runAsync和supplyAsync等，可以指定使用的线程池，不指定则用CompletableFuture的默认线程池

```java
private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

可以看到，CompletableFuture默认线程池是调用ForkJoinPool的commonPool()方法创建，这个默认线程池的核心线程数量根据CPU核数而定，公式为`Runtime.getRuntime().availableProcessors() - 1`，以4核双槽CPU为例，核心线程数量就是`4*2-1=7`个

这样的设置满足CPU密集型的应用，但对于业务都是IO密集型的应用来说，是有风险的，当qps较高时，线程数量可能就设的太少了，会导致线上故障

所以可以根据业务情况自定义线程池使用

**2.get设置超时时间不能串行get，不然会导致接口延时`线程数量*超时时间`**