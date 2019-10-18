---
title: 学习 CompletableFuture
tags:
  - 多线程
categories:
  - Java
toc: false
date: 2019-09-17 11:28:10
---

![](/images/java.jpg)
> java 8 函数编程写爽了，发现线程异步调用也可以用函数编程来写逻辑上更清晰代码更简洁明了，特地记录下学习笔记

## CompletableFuture
CompletableFuture 属于 Future Api 功能上的拓展，它实现了`Future`接口和`CompletableStage`接口简化了异步编程的复杂性并可使用函数编程进行编码，能适应项目中各种场景需求灵活编写。`CompletableStage`提供了线程阶段性调用当现场完成某一些阶段继续进行某一些操作。

#### 创建
CompletableFuture 的创建常用 supplyAsync() 和 runAsync() 区别 supplyAsync()用于需要返回值，其中`Executor`参数可传入线程池以避免浪费资源，默认使用`ForkJoinPool`线程池，建议传入线程池便于自行控制线程池大小等各类设置。

``` java
/**
 *  返回值为String异步调用
 */
Completable<String> hasResult = CompletableFuture.supplyAsync(() -> {
	代码块
	return 结果;
})

/**
 *  无返回值异步调用
 */
Completable<Void> noResult = CompletableFuture.runAsync(() -> {
	代码块
})
```
 
#### 转换和运行
CompletableFuture.get()会阻塞线程直到线程调用完毕返回结果，如果我想将结果进行其他逻辑处理必须要等待结果完成，对于异步构建我们可以使用回调方法不必等待结果返回

- __thenApply()__  有返回值转化，可以获取上一步 CompletableFuture 结果进行处理转化，类似于stream 中的 map() 方法
- __thenCompose()__ 类似于 thenApply() 用于多个 CompletableFutre 结果处理，类似于 stream 中的 flatMap() 方法
- __thenRun()__    无返回值转化，不可获取上一步 CompletableFuture 结果
- __thenAccept()__ 无返回值转化，可以获取上一步 CompletableFuture 结果
- __thenCombine()__ 组合2个 独立 CompletableFuture 并发运行当2个都完成后进行回调结果转化
- __thenAcceptBoth()__ 类似于thenCombine(),但无返回值转化
- __allOf()__ 多个 CompletableFuture 并发运行无返回值，所有结果需要通过特殊处理来获取
- __anyOf()__ 多个 CompletableFuture 并发运行第一个完成的结果
- __applyToEither__ 两个 CompletableFuture 取第一个完成的结果进行处理并返回
- __acceptEither__ 两个 CompletableFuture 取第一个完成的结果进行处理无返回



> 回调方法带有Async的方法为异步调用为独立的线程与主线程不在同一个线程中，且可传入线程池对象或使用默认的`ForkJoinPool`

``` java
/**
 *  thenApply 类型转化
 */
CompletableFuture threadOne = Completable.supplyAsync(() -> ｛
	代码块
	return 结果;
｝).thenApply(s->{
	代码块(这里可以引用到s)
	return 结果;
}).thenAccept(s->{
	代码块
}).thenRun(()->{
	代码块
})；

/**
 *  thenCompose 用于 CompletableFuture 之间链接
 */
CompletableFuture compose = CompletableFuture.supplyAsync(() -> {
	代码块
	return 结果;
}).thenCompose(s -> CompletableFuture.supplyAsync(()->{
	代码块(这里可以引用到s)
	return 结果;
}))


/**
 *  thenCombine 两个 CompletableFuture 并发运行后结果回调
 */
CompletableFuture combineOne = CompletableFuture.supplyAsync(() -> ｛
	代码块
	return 结果;
});

CompletableFuture combineTwo = CompletableFuture.supplyAsync(() -> ｛
	代码块
	return 结果;
});

CompletableFuture combineThread = combineOne.thenCombine(combineTwo, (o, o2) -> {
	代码块
	return o + o2;
});

/**
 *  allOf 多个 CompletableFuture 全部并行处理完毕后进行处理
 */
CompletableFuture allOne = CompletableFuture.runAsync(() -> {
	代码块
})

CompletableFuture allTwo = CompletableFuture.runAsync(() -> {
	代码块
})

CompletableFuture allOf = CompletableFuture.allOf(allOne,allTwo);

```

#### 完成结束
获取异步调用结果使用一下函数
- __get()__ 阻塞方法等待直到线程执行完毕，会抛出checked异常需要`catch`或者`throws`
- __get(lont timeOut,TimeUnit unit)__  带有超时时间的阻塞方法
- __getNow(T valueIfAbsent)__ 立刻获取结果如果没有则返回 `valueIfAbsent`值
- __join()__ 阻塞方法与get()不同的是会抛出`unchecked`异常
- __complete(String value)__ 线程直接完成并返回`value`值

#### 异常补偿
在线程链式调用中如果某一步发生异常后续的所有调用都将不会执行，为了更好的处理可以引入异常补偿，根据实际的业务需求处理异常

-__whenComplete()__ 当完成调用可对有异常情况进行处理
-__handle()__ 相当于whenComplete()+结果转化