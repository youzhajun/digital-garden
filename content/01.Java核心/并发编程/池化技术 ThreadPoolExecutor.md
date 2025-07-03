---
title: 池化技术 ThreadPoolExecutor
draft: false
tags:
  - java
  - java核心
  - 并发编程
date: 2022-11-22
---
 
# ThreadPoolExecutor
## 介绍

[[ExecutorService]] 下的实现类，它为我们提供了一种灵活且强大的方式来管理和执行异步任务。简单来说，它就是一个线程池的实现，旨在解决频繁创建和销毁线程带来的性能开销以及资源管理问题。

`ThreadPoolExecutor`通过“池化”技术，预先创建一定数量的线程，当有任务到来时，直接从池中获取空闲线程来执行任务，任务完成后线程不会销毁，而是返回池中等待下一个任务。这样就大大减少了线程创建和销毁的开销，提高了程序的响应速度和吞吐量。

## 构造方法

```java
public ThreadPoolExecutor(int corePoolSize,  
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

- `corePoolSize` 核心线程数
	- 即使线程处于空闲状态，核心线程也会一直存在。
	- 当提交一个新任务时，如果当前运行的线程少于`corePoolSize`，即使有空闲线程，也会创建一个新线程来执行任务（除非队列已满）。
	- 可以理解为线程池中的“常驻线程”。
- `maximumPoolSize` 最大线程数
	- 线程池中允许存在的最大线程数。
	- 当任务队列已满，并且当前运行的线程数小于`maximumPoolSize`时，线程池会创建新线程来处理任务。
	- 当线程数达到`maximumPoolSize`，并且任务队列也已满时，新的任务将会被拒绝（根据拒绝策略）。
- `keepAliveTime` (空闲线程存活时间)
	- 当线程池中的线程数量超过`corePoolSize`时，这些“额外”的空闲线程在等待任务的`keepAliveTime`时间后，如果还没有新任务，就会被终止，直到线程数量降到`corePoolSize`。
	- `unit`参数指定了`keepAliveTime`的时间单位（如`TimeUnit.SECONDS`）。
- `unit` (时间单位):
	- `keepAliveTime`的单位。`TimeUnit`有许多常量如天、小时、分钟、秒、毫秒等。
- `workQueue` 任务队列：
	- 用于存放等待执行的任务的阻塞队列。
	- 当核心线程都在忙碌时，新提交的任务会进入这个队列等待。
	- 常用的阻塞队列类型有：
		- **`ArrayBlockingQueue`:** 基于数组的有界阻塞队列，FIFO。
		- **`LinkedBlockingQueue`:** 基于链表的有界（默认无界）阻塞队列，FIFO。通常建议指定容量，否则可能导致OOM。
		- **`SynchronousQueue`:** 一个不存储元素的阻塞队列。每个插入操作必须等待一个对应的移除操作，反之亦然。提交任务后，必须有线程立即消费，否则就会创建新线程。
		-  **`PriorityBlockingQueue`:** 支持优先级的无界阻塞队列。
- `threadFactory` (线程工厂):
	- 用于创建新线程的工厂。
	- 可以通过自定义`ThreadFactory`来为线程设置名称、优先级、守护状态等，方便调试和监控。
	- 如果不指定，会使用`Executors.defaultThreadFactory()`。
- `handler` (拒绝策略):
	- 当线程池无法处理新提交的任务时（比如线程数已达到`maximumPoolSize`且任务队列已满），会根据拒绝策略来处理。可以使用`ThreadPoolExecutor`内置的拒绝策略，也可以自己实现`RejectedExecutionHandler`接口实现自定义拒绝策略，`ThreadPoolExecutor`内置的拒绝策略有：
		-  **`AbortPolicy` (默认):**  抛出`RejectedExecutionException`异常。
		- **`CallerRunsPolicy`:**  在调用者线程中执行任务。
		- **`DiscardPolicy`:** 直接丢弃任务，不做任何处理。
		- **`DiscardOldestPolicy`:** 丢弃队列中最老的任务，然后尝试重新提交当前任务。


## 工作流程

当一个任务被提交到 `ThreadPoolExecutor` 时，大致流程如下：
1. **核心线程数判断** ： 如果当前运行的线程数小于 `corePoolSize` 则创建并启动一个新线程来执行任务。
2. **任务队列判断**： 如果当前运行的线程数大于或等于`corePoolSize`，且任务队列未满，则将任务放入任务队列等待执行。
3. **最大线程判断：** 如果任务队列已满，且当前运行的线程数小于`maximumPoolSize`，则创建并启动一个新线程来执行任务。
4. **拒绝策略：** 如果任务队列已满，且当前运行的线程数等于`maximumPoolSize`，则根据拒绝策略来处理该任务。

> 举个例子

```java
	// 核心线程数2，最大线程数5，空闲线程存活时间1分钟，使用LinkedBlockingQueue作为任务队列 
	ThreadPoolExecutor executor = new ThreadPoolExecutor( 
		2, // corePoolSize 
		5, // maximumPoolSize 
		1, // keepAliveTime 
		TimeUnit.MINUTES, // unit 
		new LinkedBlockingQueue<>(10), // workQueue (容量为10) 
		Executors.defaultThreadFactory(), // threadFactory 
		new ThreadPoolExecutor.AbortPolicy() // handler 
	);
```

根据上述的构造方法构造出来的线程池，此时往线程池中提交任务，假设池中已经有2个任务正在执行，现在要提交第三个任务，要走以下流程：
1. **核心线程数判断** ： 当前运行的线程数2 = `corePoolSize`2 ，不会启动新线程来执行任务。
2. **任务队列判断**： 当前运行的线程数2 = `corePoolSize`2 ， 任务队列 `workQueue` 的容量为10 ，且现在有0个排队任务，任务队列未满，则将任务放入任务队列等待执行。
3. **最大线程判断：** 不进行此步骤
4.  **拒绝策略：** 不进行此步骤

## 线程池状态

- `RUNNING`：接收新任务并处理队列中的任务。
- `SHUTDOWN`：不再接收新任务，但会处理队列中的任务。
- `STOP`：不再接收新任务，也不处理队列中的任务，并且会中断正在执行的任务。
- `TIDYING`：所有任务都已终止，`workerCount`为0，即将调用`terminated()`钩子方法。
- `TERMINATED`：`terminated()`方法已经执行完成

## 应用中的建议

- 根据业务类型选择配置合适的线程池参数
	- **cpu 密集型任务**，`corePoolSize`可以设置为CPU核心数或核心数 + 1。
	- **io 密集型任务**，`corePoolSize`可以适当调大，因为线程在等待IO时不会占用CPU。
- 选择合适的任务队列
	- 任务量小，任务执行时间短，可以使用`SynchronousQueue`。
	- 如果任务量大，可以使用`ArrayBlockingQueue`或`LinkedBlockingQueue`
- **自定义`ThreadFactory`：** 方便线程命名和调试。
- **自定义`RejectedExecutionHandler`：** 在任务被拒绝时，可以进行日志记录、报警或者降级处理。
- **监控线程池状态：** `ThreadPoolExecutor`提供了例如`getTaskCount()`、`getCompletedTaskCount()`、`getPoolSize()`、`getActiveCount()`等方法，可以用于监控线程池的运行状况。
- **优雅关闭线程池：** 在应用程序关闭时，调用`shutdown()`来确保所有任务都能正常完成。


