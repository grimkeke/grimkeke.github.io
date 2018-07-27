---
layout: post
title:  CompletableFuture读书笔记
date:   2018-02-26 17:54:44 +0800
categories: java8
---
## 组合式异步编程
随着硬件平台的变化和应用架构的升级，越来越多的编程任务需要利用硬件平台提供的多核能力进行
并行运算，同时，由于微服务的系统架构，一个应用系统会使用来自多个系统来源的数据，或者由于
应用使用了第三方提供的API，采用了混聚（mash-up）的方式，因此，一个页面的展示可能会同时
获取多个数据源的数据，而获取数据源是一个耗时的操作，我们更希望当一个数据源数据获取完毕后
马上显示对应页面信息，而不是等所有的数据源都获取完数据才展示页面，以免用户长时间面对空白
页面。因此，我们需要高效地利用单个CPU时间，更好地并发这些松耦合任务，使CPU足够忙碌，从而
最大化程序的吞吐量。所以我们真正想做的是并发执行这些耗时的操作，从而让CPU避免等待数据
的返回而可以继续执行其他操作。
#### 关于并发和并行的区别
- 并行是真正意义上的并行，即多个CPU同时执行多个任务。两个任务在时间上和空间上是同时执行的。
- 而并发并不是真正意义上的并行。并发是利用了CPU的高处理性能，对多个任务在同一个时间段
内分片执行，因此，并发对用户的最终效果是一个CPU在同时在执行多个任务。并发对I/O密集型的
任务比较有效，是因为这类任务的主要时间是在等待数据，因此我们可以利用CPU的处理能力在短时间
内处理多个任务。
### Future接口
Future接口在java5中被引入，目的是为耗时的操作
```
public void futureBeforeJava8() {
    ExecutorService executor = Executors.newCachedThreadPool();
    Future<Double> future = executor.submit(new Callable<Double>() {
        @Override
        public Double call() throws Exception {
            return doSomeLongComputation();
        }
    });
    doSomethingElse();

    try {
        Double result = future.get(5, TimeUnit.SECONDS);
    } catch (ExecutionException ee) {
        // 计算抛出一个异常
    } catch (InterruptedException ie) {
        // 当前线程在等待过程中被中断
    } catch (TimeoutException te) {
        // 在Future对象完成之前超过最长等待时间
    }
}

public void doSomethingElse() {}

public double doSomeLongComputation() {
    try {
        Thread.sleep(2 * 1000L);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 1.0;
}
```
