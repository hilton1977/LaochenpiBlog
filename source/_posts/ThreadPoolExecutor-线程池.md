---
title: ThreadPoolExecutor 线程池
tags: []
categories: []
toc: false
date: 2019-11-26 16:52:31
---

### ThreadPoolExecutor 线程池
在阿里开发手册里使用线程池特别提到了禁止使用 __Executors__ 创建线程池，通过源码可知 _ Executors_ 提供的几种默认线程池其内部都是通过new ThreadPoolExcutor()进行创建，由于项目场景不一样在各种情况下默认提供的线程池都有一定缺陷

``` java
 public ThreadPoolExecutor(int corePoolSize, 
                              int maximumPoolSize,  
                              long keepAliveTime,  
                              TimeUnit unit,  
                              BlockingQueue<Runnable> workQueue, 
                              ThreadFactory threadFactory, 
                              RejectedExecutionHandler handler ) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

##### 参数解释
|参数|意义|
|-|-|
|corePoolSize|核心线程池数 核心线程即使空闲也会存在除非设置了 allowCoreThreadTimeOut |
|maximumPoolSize|最大线程池数|
|keepAliveTime|线程空闲时间|
|unit|线程空闲时间单位|
|workQueue|线程等待队列 ArrayBlockQueue、LinkedBlockQueue|
|threadFactory|线程创建工厂类|
|handler|饱和拒绝策略类 实现 RejectedExecutionHandler 接口自定义策略|


##### 预配置线程池
- Executors.newCachedThreadPool()
``` java
  public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
```
maximumPoolSize 设置为 Interger.MAX_VALUE 说明这是一个几乎无限大小的线程池，SynchornousQueue 无缓冲无存储数据队列每当请求过来会直接创建线程，当消费速度小于生产速度创建过多线程不能及时回收导致 OOM

- Executors.newSingleThreadExecutor()
``` java
return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
```
与 newCachedThreadPool 不一样的是 maximumPoolSize 为 1，但是所用队列为 LinkedBlockingQueue 无界链表阻塞队列默认大小为 Interger.MAX_VALUE，当消费速度小于生产速度队列里的待消费线程堆积过多导致 OOM

- Executors.newFixedThreadPool(int nThreads)
``` java
return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
```
虽然 maximumPoolSize、corePoolSize 可自行设置，但是由于使用 LinkedBlockingQueue 默认大小问题一样会导致队列堆积产生 OOM

### 线程池执行过程
![线程池执行过程](/images/ThreadPoolExcutor.jpg)

##### 源码分析
``` java
//提交任务
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
	// 获取当前工作线程数量是否小于核心线程池容量 
	// 小于则创建初始化核心线程 
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
	// 超过核心线程池数则进行判断任务队列是否已满 检测线程池是否处于运行状态且任务队列添加成功
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
	    // 再次检查建线程池状态是否为运行 
	    // 运行状态且工作线程数为0：执行 addWorker()
	    // 非运行状态：执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
	//队列已满 直接创建新的线程失败则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
}

//添加任务
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 判断队列 线程池状态
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
		
            for (;;) {
                int wc = workerCountOf(c);
		// 大于默认线程池最大容量或大于核心线程池或最大线程池限制
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
		// CAS 比较替换自增更新线程池数   
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
	// 将传入的线程进行 Worker 包装  
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
		// 加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 获取线程池状态并判断
                    int rs = runStateOf(ctl.get());
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
			// 添加待执行 HashSet<Worker>
                        workers.add(w);
			// 更新最大线程池数
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
		    // 解锁
                    mainLock.unlock();
                }
                if (workerAdded) {
		    // 任务添加成功 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
	    // 启动任务失败 移除任务
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
}
```
##### 线程状态
|状态|解释|
|-|-|-|
|RUNNING|运行状态，可处理新任务并执行队列中的任务|
|SHUTDOWN|关闭状态，不接受新任务但处理队列中的任务|
|STOP|停止状态，不接受新任务不处理队列中任务且中断执行中的任务|
|TIDYING|整理状态，所有任务已结束，workerCount = 0，将执行 terminated() 方法|
|TERMINATED|结束状态，terminated() 方法执行完毕|

### 线程池参数

#### ThreadFactory 线程工厂类
通过实现 ThreadFactory 接口可做自定义线程生成工厂，可以自定义设置线程的name、group、优先级等，默认线程工厂创建的线程为非守护线程且有相同的优先级。
``` java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
	    // 设置非守护线程
            if (t.isDaemon())
                t.setDaemon(false);
	    // 设置默认优先级
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```


#### 线程队列
线程队列用于存放提交的任务，队列实际的最大容量 = maximumPoolSize+ queueSize，根据业务场景采用不同的线程队列可最大限度的发挥线程池的性能

- SynchronousQueue 无缓冲无容量不存储元素的阻塞等待队列，每次提交任务需要等待消费配对才能进行下一个任务否则一直阻塞
- ArrayBlockingQueue 有界缓存等待队列可以设置队列的容量，当线程池大于 coolPoolSize 则缓存到队列中直到饱和，合理的控制队列大小避免 OOM
- LinkedBlockingQueue 无解缓存等待队列其容量大小为 Interger.Max ，线程池 maximumPoolSize 参数直接无效

#### 拒绝策略
- 当线程池任务线程小于 corePoolSize 时，则会创建核心线程执行任务不会从队列中取空闲线程
- 当线程池任务大于 corePoolSize 时，则会从线程队列中取出空闲现场执行任务
- 当线程池任务大于 corePoolSize　且线程队列中无空闲现场，则会创建新的线程执行任务直到创建线程数大于　maximumPoolSize，线程池便会执行 RejectedExecutionHandler 拒绝策略

#### 默认策略

|名称|策略|
|-|-|
|AbortPolicy|线程池满达到上限啧抛出异常|
|DiscardOldestPolicy|线程池满达到上限丢弃最老的一个任务即最先被丢入缓存队列的任务|
|DiscardPolicy|线程池满达到上限直接丢弃任务|

根据业务的需求可以实现 RejectedExecutionHandler 接口自定义拒绝的策略