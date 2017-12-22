---
title: CompeletableFuture的使用
date: 2017-12-21 18:27:44
layout: tag
tags: [Java, CompeletableFuture]
---


## 例子

我们就使用Java8 in action里面的商店的例子来说明。
我们写了一个应用，这个应用需要通过互联网接口从其他的服务商那里取得价格，
由于会有好多个服务商，因此我们先将操作封装到Shop类中。

<!--more-->

```java
public class Shop {
    Random random = new Random();
    String name;
    public Shop(String name) {
        this.name = name;
    }
    public double getPrice(String product) {
        return caculatePrice(product);
    }

    // price既跟店铺name有关系，也跟product有关系
    public double caculatePrice(String product) {
        delay();
        return random.nextDouble() * name.charAt(0) + product.charAt(1);
    }
    public static void delay() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}
```
我们用 `delay` 来模拟耗时操作，每次从服务商那边获取价格有一个1s的延迟，可以看到如果串行获取多个服务商的价格的话，延迟会非常严重，对用户来说是不可接受的。

### 以前的future方式

我们可以将获取价格封装一个异步版本，返回`Future`，在需要的时候使用`get`方法来得到返回的价格

```java
public Future<Double>  getPriceAsync(String product) {
    CompletableFuture<Double> future = new CompletableFuture<>();
    new Thread(()->{
        double price = getPrice(product);
        future.complete(price);
    }).start();
    return future;
}
```

我们来测试一下异步版本的耗时：

```java
public static void singleShop() throws ExecutionException, InterruptedException {
    Shop shop = new Shop("");
    long current = System.currentTimeMillis();
    Future<Double> future = shop.getPriceAsync("abc");
    long returned = System.currentTimeMillis();
    System.out.println("返回使用了:" + (returned - current) + "msecs");

    double price = future.get();
    long caculated = System.currentTimeMillis();
    System.out.println("price is " + price);
    System.out.println("计算使用时间:" + (caculated - current) + "msecs");
}
```

测试结果：

```java
返回使用了:75msecs
price is 140.00108871644375
计算使用时间:1077msecs
```
可以看到方法返回的速度是很快的，在返回后与得到值之间有很长的间隔，我们可以利用这段时间来做点别的。

### CompletableFuture方式

Java8提供了CompletableFuture，里面有`supplyAsync`方法可以让我们直接提交一个任务，返回`Future`
可以看到代码精简到了一行。

```java
public Future<Double> getPriceAsyncElegently(String product) {
    return CompletableFuture.supplyAsync(() -> getPrice(product));
}
```

看到这你可能会说了，不就是把操作封装了一下嘛，我自己也可以写一个方法，然后一行返回，别急，我们接着来看CompletableFuture提供给我们的其他功能，简直不要太顺手。

### 与Stream结合使用

上面说了会从很多的服务商那边获取价格，上面只是获取了一家，但假如是10家呢？我们就需要写10遍了，太繁琐，我们使用Stream来实现一下。

先声明一下店铺,我直接复制了多个店铺：

```java
List<Shop> shopList = Arrays.asList(new Shop("a"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("b"),
     new Shop("c"));
```

用CompletableFuture跟Stream结合来计算价格

```java
public static List<Double> manyShopsFuture(String product) {
    List<CompletableFuture<Double>> stream = shopList.stream()
        .map(s -> CompletableFuture.supplyAsync(() -> s.getPrice(product)))
        .collect(Collectors.toList());

    return stream.stream().map(CompletableFuture::join).collect(Collectors.toList());
}
```

在这里我们使用了2个stream来操作，因为如果把join操作写到第一个stream中的话，实际上操作已经变成了线性的了，所以这里我们先获取future，再统一join等待结果返回。


不过还记得么，Stream类也提供了并行流，实现起来好像更加简单：

```java
public static List<Double> manyShopsParallel(String product) {
    return shopList.parallelStream().map(shop -> shop.getPrice(product)).collect(Collectors.toList());
}
```

我们测试一下，比较下两者的运行效率:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    String product = "abc";
    long current = System.currentTimeMillis();
    manyShopsParallel(product);
    long future = System.currentTimeMillis();
    System.out.println("manyShopsParallel cost:" + (future - current));
    manyShopsFuture(product);
    long stream = System.currentTimeMillis();
    System.out.println("manyShopsFuture cost:" + (stream - future));
}
```
执行结果

> manyShopsParallel cost:3153
> manyShopsFuture cost:4002
 
可以看到使用ParallelStream更高效一些，写了这么多，效率却不如默认的好，那如何提高我们自己的程序的运行效率呢？

### 提供自己的线程池

其实CompletableFuture跟parallelStream一样，都是使用的`ForkJoinPool`中的默认线程池，线程数量默认为机器的内核数`Runtime.getRuntime().availableProcessors()`,对于我们这样的等待时间长，IO密集型的应用来说，CPU是大大的浪费了的，parallelStream是无法定制线程池的，但是CompletableFuture我们却可以自行提供，以便根据自己的应用情况作出调整。

《Java并发编程实战》中给过一个计算线程池线程数的公式，为：
> Nthreads = NCPU * UCPU * (1 + W/C)
> 其中：
> NCPU是处理器的核的数目，可以通过Runtime.getRuntime().availableProcessors()得到
> UCPU是期望的CPU利用率（该值应该介于0和1之间）
> W/C是等待时间与计算时间的比率

大家可以计算一下自己的，我这里Ncpu=2，Ucpu=100%，W/C = 1/0.01 = 100 ,因此取线程数=200来构造线程池
如下：

```java
static Executor executor = Executors.newFixedThreadPool(200, new ThreadFactory() {
    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setDaemon(true);
        return thread;
    }
});
```

```java
public static List<Double> manyShopsFuture(String product) {
     List<CompletableFuture<Double>> stream = shopList.stream()
         .map(s -> CompletableFuture.supplyAsync(() -> s.getPrice(product),executor))
         .collect(Collectors.toList());

     return stream.stream().map(CompletableFuture::join).collect(Collectors.toList());
}
```

再次执行一下看计算时间：
> manyShopsParallel cost:3250
> manyShopsFuture cost:1006

Future方式可以说是完全并行了，而parallelStream由于使用默认线程池，并不能一次性全部将任务执行，需要更长的执行时间。

## CompletableFuture组合异步任务

假设我们在获取价格之后，还需要查询服务商的折扣服务才能计算最终展示的价格，这个延迟也会比较大，我们如何来组合这两个异步任务呢？CompletableFuture提供了一系列的then方法，我们这里使用两种来演示一下，一个是`thenApply`,一个是`thenCompose`, `thenApply`是对结果进行处理，`thenCompose`是组合一个新的任务

先定义一下`Discount`
```java
public class Discount {

    public static Double applyDiscount(Double price) {
        Double discount = getDiscount();
        return price * discount;
    }

    public static Double getDiscount() {
        Shop.delay();
        return 0.5;
    }
}
```

然后看一下任务组合调用：

```java
public static List<Double> manyShopsApplyWithDiscount(String product) {
    List<CompletableFuture<Double>> stream = shopList.stream()
        .map(s -> CompletableFuture.supplyAsync(() -> s.getPrice(product),executor))
        .map(future -> future.thenApply(Discount::applyDiscount))
        .collect(Collectors.toList());

    return stream.stream().map(CompletableFuture::join).collect(Collectors.toList());
}
public static List<Double> manyShopsComposeWithDiscount(String product) {
    List<CompletableFuture<Double>> stream = shopList.stream()
        .map(s -> CompletableFuture.supplyAsync(() -> s.getPrice(product),executor))
        .map(future -> future.thenCompose(price ->
            CompletableFuture.supplyAsync(()-> Discount.applyDiscount(price),executor)))
        .collect(Collectors.toList());

    return stream.stream().map(CompletableFuture::join).collect(Collectors.toList());
}
```

在`thenCompose`中我们通过`supplyAsync` 再次提交了一次异步任务，而在`thenApply`中我们直接在原流水线上进行数据处理，不过不会阻塞流水线，也是提交了一个任务，不过是同步执行。这两个方法在我看来就是处理参数的不同而已，不用太过纠结。

测试一下性能：

> manyShopsComposeWithDiscount cost:2126
> manyShopsApplyWithDiscount cost:2019

`thenApply`方法由于减少了线程切换执行时间相对较短，也提醒我们在编程过程中注意这方面的开销。


# 最后

CompletableFuture还提供了很多其他的API可供我们使用，比如说`thenCombine`可以结合两个没有先后关系的异步任务，但是提供回调来处理两个任务的结果，等着大家去发现使用。