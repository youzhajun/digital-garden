---
title: CompletableFuture
draft: false
tags:
  - 并发编程
date: 2025-05-21
---
可以简单理解为任务包装执行器，类似于前端的 promise

## 📝 主旨内容

业务开发中总会有执行异步任务的场景，从而加快执行进度，JDK8 以后提供了 `CompletableFuture` 。 举个例子, 主线程上有两个查询任务，两者之间毫不关联：

```java
public void testCompletableInfo() throws InterruptedException, ExecutionException {
    long startTime = System.currentTimeMillis();

    //调用用户服务获取用户基本信息
    CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() ->
            //模拟查询商品耗时500毫秒
    {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "用户A";
    });

    //调用商品服务获取商品基本信息
    CompletableFuture<String> goodsFuture = CompletableFuture.supplyAsync(() ->
            //模拟查询商品耗时500毫秒
    {
        try {
            Thread.sleep(400);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "商品A";
    });

    System.out.println("获取用户信息:" + userFuture.get());
    System.out.println("获取商品信息:" + goodsFuture.get());

    //模拟主程序耗时时间
    Thread.sleep(600);
    System.out.println("总共用时" + (System.currentTimeMillis() - startTime) + "ms");
}
```

## `CompletableFuture` 创建方式

`CompletableFuture` 提供了4个静态方法用来创建 `CompletableFuture` 。

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
  return asyncSupplyStage(asyncPool, supplier);
}

public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                   Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

public static CompletableFuture<Void> runAsync(Runnable runnable) {
  return asyncRunStage(asyncPool, runnable);
}

public static CompletableFuture<Void> runAsync(Runnable runnable,
                                               Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```

其中，两个 `supplyAsync` 支持返回值，`runAsync` 执行任务没有返回值。

## `CompletableFuture` 获取结果

```java
public T get()

public T get(long timeout, TimeUnit unit)

public T getNow(T valueIfAbsent)

public T join()
```

- 其中 `get()` 提供超时处理，如果在指定时间内获取不到内容则会抛出异常。
- `getNow(valueIfAbsent)` 立即获取结果不阻塞，结果计算已经完成返回结果或者异常，如果没有完成则返回入参`valueIfAbsent`
- `join()` 方法不会抛出异常

```java
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() ->
//模拟查询耗时500毫秒
  {
      try {
          Thread.sleep(500);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "用户A";
  });
System.out.println("获取用户信息:" + userFuture.get(300, TimeUnit.MILLISECONDS));

// 抛出异常
java.util.concurrent.TimeoutException
```

```java
@Test
    public void test3() throws InterruptedException, ExecutionException, TimeoutException {
        long startTime = System.currentTimeMillis();

        CompletableFuture<Integer> testFuture = CompletableFuture.supplyAsync((() -> 1 / 0));

        System.out.println(testFuture.join()); 
        // join 方法获取结果方法里不会抛异常，但是执行结果会抛异常，抛出的异常为 CompletionException
        
    }
```

## 异步回调

多个任务执行，某一个任务开始需要任务A的结果，可以称之为回调方法，也可以理解为前端`promise` 的链式调用。大体有以下几种情况：

- 不依赖上一个任务的返回值，只是希望上一个任务结束后再进行此任务。`thenRun`、`thenRunAsync`
- 依赖上一个任务的返回值，但是此任务没有返回值。`thenAccept`、`thenAcceptAsync`
- 依赖上一个任务的返回值，此任务也会有返回值。`thenApply`、`thenApplyAsync`
- 某个任务异常时，执行回调方法。`exceptionlly`
- 某个任务执行完成后，执行的回调方法，无返回值 `whenComplete`
- 某个任务执行完成后，执行的回调方法，有返回值 `handle`

```java

@Test
public void test4() throws InterruptedException, ExecutionException, TimeoutException {
    CompletableFuture<Void> task1 = CompletableFuture.runAsync(() -> {
        System.out.println("吃饭");
    });

    CompletableFuture<Void> task2 = task1.thenRun(() -> {
        System.out.println("睡觉");
    });
    task2.get();
}

@Test
public void test5() throws InterruptedException, ExecutionException, TimeoutException {
  CompletableFuture<String> task1 = CompletableFuture.supplyAsync(() -> {
      System.out.println("吃饭");
      return "吃完饭啦";
  });

  CompletableFuture<Void> task2 = task1.thenAccept((s) -> {
      System.out.println(s);
      System.out.println("睡觉");
  });
  task2.get();
}
```

任务不论是正常还是异常都会调用 `whenComplete` 这个回调函数。它的核心特点是 **不改变任务的结果或异常状态**，仅作为一个观察者（Side Effect）存在。

- 正常情况下：whenComplete 返回结果与上级任务一致，异常为nul
- 异常情况下：whenComplete 返回结果为nul，异常为上级任务的异常

```java
@Test
public void testCompletableWhenComplete() throws ExecutionException, InterruptedException {
    CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> {

        if (Math.random() < 0.5) {
            throw new RuntimeException("出错了");
        }
        System.out.println("正常结束");
        return 0.11;

    }).whenComplete((aDouble, throwable) -> {
        if (aDouble == null) {
            System.out.println("whenComplete aDouble is null");
        } else {
            System.out.println("whenComplete aDouble is " + aDouble);
        }
        if (throwable == null) {
            System.out.println("whenComplete throwable is null");
        } else {
            System.out.println("whenComplete throwable is " + throwable.getMessage());
        }
    });
    System.out.println("最终返回的结果 = " + future.get());
}
```

程序取了随机数，要么成功正常运行，要么抛出异常。

当成功时，打印结果为：

```java
正常结束
whenComplete aDouble is 0.11
whenComplete throwable is null
最终返回的结果 = 0.11
```

当异常时，打印结果为

```java
RuntimeException("出错了");
whenComplete throwable is "出错了"
```

## 多任务组合

并且关系（都结束了再怎么怎么样）、或关系（有一个结束了再怎么怎么样）

`thenCombine` / `thenAcceptBoth` / `runAfterBoth` 表示当任务1、2都执行结束后再执行任务3.

区别在于：

- `thenAfterBoth` 不会把执行结果当做方法入参，且没有返回值
- `thenAcceptBoth` 会将两个任务的执行结果作为方法入参，传递到指定方法中，且无返回值
- `thenCombine` 会将两个任务的执行结果作为方法入参，传递到指定方法中，且有返回值

```java
@Test
public void testCompletableThenCombine() throws ExecutionException, InterruptedException {
    //创建线程池
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    //开启异步任务1
    CompletableFuture<Integer> task = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务1，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 1;
        System.out.println("异步任务1结束");
        return result;
    }, executorService);

    //开启异步任务2
    CompletableFuture<Integer> task2 = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务2，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 1;
        System.out.println("异步任务2结束");
        return result;
    }, executorService);

    //任务组合
    CompletableFuture<Integer> task3 = task.thenCombineAsync(task2, (f1, f2) -> {
        System.out.println("执行任务3，当前线程是：" + Thread.currentThread().getId());
        System.out.println("任务1返回值：" + f1);
        System.out.println("任务2返回值：" + f2);
        return f1 + f2;
    }, executorService);

    Integer res = task3.get();
    System.out.println("最终结果：" + res);
}
```

```java
异步任务1，当前线程是：17
异步任务1结束
异步任务2，当前线程是：18
异步任务2结束
执行任务3，当前线程是：19
任务1返回值：2
任务2返回值：2
最终结果：4
```

或者关系，`applyToEither` / `acceptEither` / `runAfterEither` 都表示：**「两个任务，只要有一个任务完成，就执行任务三」**。

- **「`runAfterEither`」**：不会把执行结果当做方法入参，且没有返回值
- **「`acceptEither`」**: 会将已经执行完成的任务，作为方法入参，传递到指定方法中，且无返回值
- **「`applyToEither`」**：会将已经执行完成的任务，作为方法入参，传递到指定方法中，且有返回值

```java
@Test
public void testCompletableEitherAsync() {
    //创建线程池
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    //开启异步任务1
    CompletableFuture<Integer> task = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务1，当前线程是：" + Thread.currentThread().getId());

        int result = 1 + 1;
        System.out.println("异步任务1结束");
        return result;
    }, executorService);

    //开启异步任务2
    CompletableFuture<Integer> task2 = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务2，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 2;
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("异步任务2结束");
        return result;
    }, executorService);

    //任务组合
    task.acceptEitherAsync(task2, (res) -> {
        System.out.println("执行任务3，当前线程是：" + Thread.currentThread().getId());
        System.out.println("上一个任务的结果为："+res);
    }, executorService);
}
```

多任务组合，将多个任务合并成一个任务，**「`allOf`」**：等待所有任务完成。**「`anyOf`」**：只要有一个任务完成。

```java
@Test
public void testCompletableAallOf() throws ExecutionException, InterruptedException {
    //创建线程池
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    //开启异步任务1
    CompletableFuture<Integer> task = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务1，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 1;
        System.out.println("异步任务1结束");
        return result;
    }, executorService);

    //开启异步任务2
    CompletableFuture<Integer> task2 = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务2，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 2;
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("异步任务2结束");
        return result;
    }, executorService);

    //开启异步任务3
    CompletableFuture<Integer> task3 = CompletableFuture.supplyAsync(() -> {
        System.out.println("异步任务3，当前线程是：" + Thread.currentThread().getId());
        int result = 1 + 3;
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("异步任务3结束");
        return result;
    }, executorService);

    //任务组合
    CompletableFuture<Void> allOf = CompletableFuture.allOf(task, task2, task3);

    //等待所有任务完成
    allOf.get();
    //获取任务的返回结果
    System.out.println("task结果为：" + task.get());
    System.out.println("task2结果为：" + task2.get());
    System.out.println("task3结果为：" + task3.get());
}
```

```java
@Test
public void testCompletableAnyOf() throws ExecutionException, InterruptedException {
    //创建线程池
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    //开启异步任务1
    CompletableFuture<Integer> task = CompletableFuture.supplyAsync(() -> {
        int result = 1 + 1;
        return result;
    }, executorService);

    //开启异步任务2
    CompletableFuture<Integer> task2 = CompletableFuture.supplyAsync(() -> {
        int result = 1 + 2;
        return result;
    }, executorService);

    //开启异步任务3
    CompletableFuture<Integer> task3 = CompletableFuture.supplyAsync(() -> {
        int result = 1 + 3;
        return result;
    }, executorService);

    //任务组合
    CompletableFuture<Object> anyOf = CompletableFuture.anyOf(task, task2, task3);
    //只要有一个有任务完成
    Object o = anyOf.get();
    System.out.println("完成的任务的结果：" + o);
}
```