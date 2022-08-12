## CompletableFuture

### 前言

这个当时我看的时候直接略过，因为完全看不懂它在干什么。

CompleteableFuture 其实是相对于 Future 的一个优化。Future有以下缺点：

- 不能手动结束计算，如果你使用  Future 运行子线程调用远程 API 来获取某款产品的最新价格，服务器突然宕机了，此时如果你想手动结束计算，而是想返回上次缓存中的价格，这是 Future 做不到的。
- 调用 get() 会阻塞，Future 提供的 get() 方法可以拿到返回值，但是它却是同步阻塞的。
- Future 之间的整合调用很麻烦，比如链式的去执行
- 整合多个Future 结果比较麻烦
- 没有异常处理

其实主要就是 get() 的同步阻塞问题，如果要做异步编程，好的思路应该是 Future 执行结束后去回调主程序的方法。

### 函数式接口

CompletableFuture 中大量结合了函数式接口，所以首先来介绍一些接口

#### Runnable

无参数，无返回值

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

#### Function

Function<T, R> 接受一个参数，并且有返回值

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

#### Consumer

Consumer接受一个参数，没有返回值

```java
@FunctionalInterface
public interface Consumer<T> {   
    void accept(T t);
}
```

#### Supplier

Supplier没有参数，有一个返回值

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

#### BiConsumer

BiConsumer<T, U> 接受两个参数（Bi， 英文单词词根，代表两个的意思，同理还有BiFunction），没有返回值

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
```

![image-20220805111503291](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220805111503291.png)

### 类结构

CompletableFuture 实现了 Future 接口，和 CompletionStage 接口

Future 中不好去管理各个 Future 的执行以及执行结果汇总，而 CompletableFuture 就是做了这些事

![image-20220805135205013](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220805135205013.png)

![image-20220805140312685](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220805140312685.png)

**CompletableFuture** 大约有50种不同处理串行，并行，组合以及处理错误的方法。

#### 串行

以 then 开头的方法，接上 run, accept,apply,compose 表示的就是串行的流程了

```java
CompletableFuture<Void> thenRun(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
  
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
  
CompletableFuture<Void> thenAccept(Consumer<? super T> action) 
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
  
<U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)  
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn)
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor)
```

#### 聚合 And 关系

含有 combine with， both with 的就是 and 的聚合关系了

```java
<U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)

<U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync( CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor)
  
CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

#### 聚合 Or 关系

Either 表示两者中的一个，自然也就是 or的体现了

```java
<U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(、CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn, Executor executor)

CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)

CompletableFuture<Void> runAfterEither(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

#### 异常处理

```java
CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn, Executor executor)
        
CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action, Executor executor)
        
       
<U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor)
```

异常处理的 exceptionlly 就是类似 try/catch

whenComplete 、handle 相当于 try/finally

不同的是 whenComplete 没有返回值，而 handle 有返回值。

#### 小结

不难发现上面的方法都是三个一组的，里面有两个是异步的（Async）的变体

```java
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

### 实战

#### runAsync

没啥好说的，异步计算

```java
@Test
public void runAsync(){
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        System.out.println("运行在一个单独的线程当中");
    });
    try {
        TimeUnit.SECONDS.sleep(4);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("主线程结束");
}

// 打印
运行在一个单独的线程当中
主线程结束
```

#### supplyAsync

这个就是有返回结果的

```java
@Test
public void supplyAsync(){
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        System.out.println("运行在一个单独的线程当中");
        return "supplyAsyncResult";
    });
    try {
        System.out.println(future.get());
    } catch (InterruptedException | ExecutionException e) {
        throw new IllegalStateException(e);
    }
}
// 打印
运行在一个单独的线程当中
supplyAsyncResult
```

因为 get() 是阻塞的，所以我不用让主线程加休眠时间等待了。

#### get() 方法

这个 CompleteFuture 有重载，使用和 Future.get() 类似，如果没有别的操作，也会阻塞获取

```java
@Test
public void testCompletableFuture(){
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
    try {
        String s = completableFuture.get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
    System.out.println("end");
}
```

跑这个 test，会一直卡住，无法结束，因为这个 get() 肯定是返回不了的。

但是可以通过 complete() 方法来中断获取。

```java
@Test
public void testCompletableFuture() {
    CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "future";
    }
                                                                               );
    try {
        // Thread.sleep(2500);
        completableFuture.complete("stop manually");
        String res = completableFuture.get();
        System.out.println(res);

    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
    System.out.println("end");
}
```

主线程执行到 `completableFuture.complete("stop manually");` 时发现 future 还没有返回，就会直接结束，这里打印

```
stop manually
end
```

注释打开后，让主线程休眠2.5s，这时 Future 有返回了，所以 complete 不会执行，这里打印

```
future
end
```

可以看出来 complete 的参数实际上就是给的返回值，如果 future.get()无返回，就直接赋值给 res 了。

上面我们说到 get() 方法实际上是阻塞获取结果的，对于真正的异步处理，我们希望的是可以通过**传入回调函数**，在Future 结束时自动调用该回调函数，这样，我们就不用等待结果。

结合上面的 supplyAsync ，异步执行之后设置一个回调方法，可以接 then...的操作，实际上里面写的就是回调的代码。下面来看一些 then 的操作。

#### thenApply

thenApply 接受的参数就是 上一步返回的的结果，并且有一个返回值。

```java
@Test
public void supplyAsync2(){
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        log.info("运行在一个单独的线程当中");
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return "r1";
    }).thenApply(a -> {
        log.info("回调1");
        return a + "回调1";
    }).thenApply(a -> {
        log.info("回调2");
        return a + "回调2";
    });
    try {
        log.info("start");
        log.info(future.get());
        log.info("end");
    } catch (InterruptedException | ExecutionException e) {
        throw new IllegalStateException(e);
    }
}
```

打印结果：

```java
2022-08-09 13:41:42.720  INFO 34884 --- [           main] com.example.demo2.ThreadTest             : start
2022-08-09 13:41:42.720  INFO 34884 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 运行在一个单独的线程当中
2022-08-09 13:41:45.724  INFO 34884 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 回调1
2022-08-09 13:41:45.724  INFO 34884 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 回调2
2022-08-09 13:41:45.724  INFO 34884 --- [           main] com.example.demo2.ThreadTest             : r1回调1回调2
2022-08-09 13:41:45.725  INFO 34884 --- [           main] com.example.demo2.ThreadTest             : end
```

> 有地方说串行的操作不一定使用的同一线程，我倒是没有测出来，不过关系不大

#### thenAccept

如果回调中没有返回值，就可以使用它

```java
@Test
public void supplyAsyncThenAccept(){
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        log.info("运行在一个单独的线程当中");
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return "r1";
    }).thenAccept( a -> {
        log.info(a + "回调");
    });
    try {
        log.info("start");
        future.get();
        log.info("end");
    } catch (InterruptedException | ExecutionException e) {
        throw new IllegalStateException(e);
    }
}
```

```java
2022-08-09 14:00:07.468  INFO 32212 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 运行在一个单独的线程当中
2022-08-09 14:00:07.468  INFO 32212 --- [           main] com.example.demo2.ThreadTest             : start
2022-08-09 14:00:10.474  INFO 32212 --- [onPool-worker-1] com.example.demo2.ThreadTest             : r1回调
2022-08-09 14:00:10.474  INFO 32212 --- [           main] com.example.demo2.ThreadTest             : end
```

future.get() 这里返回值是 Void, 实际上调用没有意义，我这里调用只是阻塞主线程，不让主线程直接执行完成结束。

#### thenRun

这个既没有入参也没有返回，所以他不能拿到前面的执行结果。

```java
@Test
public void supplyAsyncThenRun(){
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        log.info("运行在一个单独的线程当中");
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return "r1";
    }).thenRun(() -> {
        log.info("回调");
    });
    try {
        log.info("start");
        future.get();
        log.info("end");
    } catch (InterruptedException | ExecutionException e) {
        throw new IllegalStateException(e);
    }
}
```

这个和 thenAccept 类似，没有返回值，future.get() 实际上返回 Void，这里只是为了阻塞主流程。

```java
2022-08-10 09:26:58.094  INFO 23224 --- [           main] com.example.demo2.ThreadTest             : start
2022-08-10 09:26:58.094  INFO 23224 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 运行在一个单独的线程当中
2022-08-10 09:27:01.107  INFO 23224 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 回调
2022-08-10 09:27:01.107  INFO 23224 --- [           main] com.example.demo2.ThreadTest             : end
```

#### Async变体

Apply，Accept，Run 每一个其实都有 Async 变体，用 Apply 示范一下：

```java
@Test
public void supplyAsyncThenAcceptAsync(){
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        log.info("运行在一个单独的线程当中");
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return "r1";
    }).thenAcceptAsync( a -> {
        log.info(a + "回调");
    }, executorService);
    try {
        log.info("start");
        future.get();
        log.info("end");
    } catch (InterruptedException | ExecutionException e) {
        throw new IllegalStateException(e);
    }
}
```

这里为了方便观察，使用了 `thenAcceptAsync(Consumer<? super T> action,Executor executor)` 给它新设置一个线程池（executorService，自定义的一个线程池）执行，否则可能还是用了同一线程。

```java
2022-08-10 09:36:55.633  INFO 18428 --- [           main] com.example.demo2.ThreadTest             : start
2022-08-10 09:36:55.633  INFO 18428 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 运行在一个单独的线程当中
2022-08-10 09:36:58.636  INFO 18428 --- [ecutorService-0] com.example.demo2.ThreadTest             : r1回调
2022-08-10 09:36:58.636  INFO 18428 --- [           main] com.example.demo2.ThreadTest             : end
```

#### thenCompose

我们在回调函数，比如 thenApply 中，又去调用了 CompletableFuture.supplyAsync 去返回，这样会发生 CompletableFuture 结果的嵌套，比如：

```java
public void supplyAsyncThenAcceptCompletableFuture(){
    CompletableFuture<CompletableFuture<String>> future = CompletableFuture.supplyAsync(() -> {
        log.info("r1");
        return "r1";
    }).thenApply(a -> CompletableFuture.supplyAsync(() -> {
        log.info(a + "r2");
        return a + "r2";
    }));
}
```

其实我们呢只需要最后一步的的 CompletableFuture ，这时就可用 thenCompose 了。

```java
@Test
public void supplyAsyncThenCompose() throws ExecutionException, InterruptedException {
    CompletableFuture<String> r1 = CompletableFuture.supplyAsync(() -> {
        log.info("r1");
        return "r1";
    }).thenCompose(a -> CompletableFuture.supplyAsync(() -> {
        log.info(a + "r2");
        return a + "r2";
    }));
    String res = r1.get();
    log.info(res);
}
```

#### thenCombine

如果要聚合两个独立的 Future 结果，使用 thenCombine

```java
@Test
public void supplyAsyncThenCombine() throws ExecutionException, InterruptedException {
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("r1");
        return "r1";
    });
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("r2");
        return "r2";
    });
    log.info("start");
    CompletableFuture<String> stringCompletableFuture = future1.thenCombine(future2, (r1, r2) -> r1 + r2);
    log.info("combine complete");
    String res = stringCompletableFuture.get();
    log.info(res);
}
```

thenCompose 不太一样，它需要两个参数，一是需要 combine 的future，二是一个 BiFunction。

BiFunction中第一个参数是调用方的返回值，第二个参数是被 combine 的 future 的返回值。BiFunction 中的返回值就是最终的 completeFuture.get() 的结果。

#### allOf/anyOf

对于两个 CompletableFuture 组合可以使用 thenCombine，但是多个组合就不好搞了，但是可以使用allOf 和 anyOf 可以组合任意多个 CompletableFuture。

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
    return andTree(cfs, 0, cfs.length - 1);
}

public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {
    return orTree(cfs, 0, cfs.length - 1);
}
```

这两个都是静态方法，接收参数是可变长的 CompletableFuture，区别是

1. allOf 含义是所有 CompletableFuture 结束时才返回。而 anyOf 含义是任一 CompletableFuture 结束时就返回。
2. allOf 返回值是 CompletableFuture<Void>，需要 thenApply，在回调函数中去 get() 。anyOf 返回值类型都可能不同，任意一个， 意味着无法判断是什么类型，所以 anyOf 的返回值是 CompletableFuture<Object>类型
3. allOf 相当于聚合所有的 CompletableFuture，而 anyOf 只返回最快执行的结果，其余的返回结果就被丢弃了。

##### allOf示例

```java
@Test
public void testAllOf() throws ExecutionException, InterruptedException {
    // 获得 CompleteFuture 的List
    List<String> pageList = new ArrayList<>();
    pageList.add("123");
    pageList.add("456");
    pageList.add("789");
    pageList.add("777");
    List<CompletableFuture<String>> collect = pageList.stream().map(this::downloadPage).collect(Collectors.toList());
    // allOf 返回值是 Void
    CompletableFuture<Void> allFuture = CompletableFuture.allOf(collect.toArray(new CompletableFuture[collect.size()]));
    // 利用 thenApply 获得 CompletableFuture<List<String>> 结果
    CompletableFuture<List<String>> listCompletableFuture = allFuture.thenApply(a -> {
        return collect.stream().map(future -> {
            String res = null;
            try {
                res = future.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
            return res;
        }).collect(Collectors.toList());
    });
    // 直接获得结果
    List<String> strings = listCompletableFuture.get();
    log.info(strings.toString());
    // 再利用 thenApply 对其进行筛选
    CompletableFuture<List<String>> resFuture = listCompletableFuture.thenApply(a -> a.stream().filter(item -> item.contains("7")).collect(Collectors.toList()));
    List<String> stringFilters = resFuture.get();
    log.info(stringFilters.toString());
}

private CompletableFuture<String> downloadPage(String url){
    long t = Math.round(Math.random() * 3000);
    return CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(t);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("完成任务： " + url);
        return url + ":end";
    });
}
```

> downloadPage() 模拟下载页面的业务，里面随机休眠 0-3s，返回一个 CompletableFuture<String>

##### anyOf 示例

```java
@Test
public void testAnyOf() throws ExecutionException, InterruptedException {
    // 获得 CompleteFuture 的List
    List<String> pageList = new ArrayList<>();
    pageList.add("123");
    pageList.add("456");
    pageList.add("789");
    pageList.add("777");
    List<CompletableFuture<String>> collect = pageList.stream().map(this::downloadPage).collect(Collectors.toList());
    // anyOf
    CompletableFuture<Object> allFuture = CompletableFuture.anyOf(collect.toArray(new CompletableFuture[collect.size()]));
    String res = (String) allFuture.get();
    log.info(res);
    // Thread.sleep(3500)
}
```

anyOf 是任一 CompletableFuture 完成后就返回，这里就只会返回一个执行完成的，而且只返回最早执行完的，比如上面的只会返回一个。

```java
2022-08-10 16:06:01.585  INFO 9904 --- [onPool-worker-2] com.example.demo2.ThreadTest             : 完成任务： 456
2022-08-10 16:06:01.585  INFO 9904 --- [           main] com.example.demo2.ThreadTest             : 456:end
```

但实际上**其余的几个 completableFuture 还是在执行的**，我让主线程挂起确保他们都能执行完毕，只不过其余的返回值被忽略了。

```java
2022-08-10 16:11:43.100  INFO 6572 --- [onPool-worker-3] com.example.demo2.ThreadTest             : 完成任务： 789
2022-08-10 16:11:43.100  INFO 6572 --- [           main] com.example.demo2.ThreadTest             : 789:end
2022-08-10 16:11:43.146  INFO 6572 --- [onPool-worker-1] com.example.demo2.ThreadTest             : 完成任务： 123
2022-08-10 16:11:44.028  INFO 6572 --- [onPool-worker-2] com.example.demo2.ThreadTest             : 完成任务： 456
2022-08-10 16:11:45.167  INFO 6572 --- [onPool-worker-4] com.example.demo2.ThreadTest             : 完成任务： 777
```

#### exceptionally

```java
@Test
public void testException() throws ExecutionException, InterruptedException {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        log.info("supplyAsync");
        // int t = 10 / 0;
        log.info("supplyAsync-end");
        return "supplyAsync";
    }).thenApply(a -> {
        log.info("apply1");
        //            int t = 10 / 0;
        log.info("apply1-end");
        return a + "apply1";
    }).thenApply(a -> {
        log.info("apply2");
        int t = 10 / 0;
        log.info("apply2-end");
        return a + "apply2";
    }).exceptionally(ex -> {
        log.error(ex.getMessage());
        return "exceptionally";
    });

    log.info(future.get());
}
```

exceptionally 方法指定某个任务执行异常时执行的回调方法，出现异常，将会跳过所有的后续操作，直接捕获异常，将抛出异常作为参数传递到回调方法中，然后返回 exceptionally 的返回值。

```java
2022-08-11 17:00:23.977  INFO 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : supplyAsync
2022-08-11 17:00:23.978  INFO 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : supplyAsync-end
2022-08-11 17:00:23.978  INFO 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : apply1
2022-08-11 17:00:23.978  INFO 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : apply1-end
2022-08-11 17:00:23.978  INFO 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : apply2
2022-08-11 17:00:23.978 ERROR 11788 --- [onPool-worker-1] com.example.demo2.ThreadTest             : java.lang.ArithmeticException: / by zero
2022-08-11 17:00:23.978  INFO 11788 --- [           main] com.example.demo2.ThreadTest             : exceptionally
```

#### whenComplete

whenComplete是当某个任务执行完成后执行的回调方法，会将执行结果或者执行期间抛出的异常传递给回调方法，如果是正常执行则异常为null，如果该任务正常执行，则 get() 方法返回执行结果，如果是执行异常，则get() 方法抛出异常。

```java
@Test
public void testWhenComplete() throws ExecutionException, InterruptedException {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        log.info("supplyAsync");
        int t = 10 / 0;
        log.info("supplyAsync-end");
        return "supplyAsync";
    }).whenComplete((res, ex) -> {
        if(ex != null){
            log.error(ex.getMessage());
        }
    });
    log.info("before get");
    log.info(future.get());
    log.info("after get");
}
```

get 会抛出异常，流程停止。

> whenComplete 接受两个参数，第一个是上个流程的结果，第二个是异常，如果没有异常，异常是 null，需要注意不要 npe。

#### handle

与上面类似，区别只是 whenComplete 没有返回值。 handle 有返回值。

```java
@Test
public void testHandle() throws ExecutionException, InterruptedException{
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        log.info("supplyAsync");
        //            int t = 10 / 0;
        log.info("supplyAsync-end");
        return 1;
    }).handle((res, ex) -> {
        if (ex != null) {
            log.error(ex.getMessage());
            return "handle result";
        }
        return "no handle";
    });
    log.info("before get");
    String s = future.get();
    log.info("res: " + s);
    log.info("after get");
}
```

>whenComplete 与 handle 类似，只不过接收参数一个是 BiComsumer，一个是 BiFunction。意味着 whenComplete 没有返回值。

handle 的返回值返回的是 handle 里面的值，所以这里返回的类型就是 String，而不是前者的 Integer (return 1)，与原始 CompletableFuture 的 result 无关了。

建议 handle 里面 return res,保证两者类型的一致。

#### ForkJoinPool

之前说到所有的方法有三种变体，其中一种 async 参数有自定义的线程池，实际上，如果没有指定，会使用默认的线程池，也就是 ForkJoinPool

```java
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}
private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

ForkJoinPool 的线程数默认是 CPU 的核心数。

> **不要所有业务共用一个线程池**，因为，一旦有任务执行一些很慢的 I/O 操作，就会导致线程池中所有线程都阻塞在 I/O 操作上，从而造成线程饥饿，进而影响整个系统的性能。所以尽量使用指定的线程池。







> [【多图长文】搞定 CompletableFuture，并发异步编程和编写串行程序还有什么区别？](https://mp.weixin.qq.com/s?__biz=Mzg2NzYyNjQzNg==&mid=2247489365&idx=1&sn=b0866f630a79e882b71d95b5bd3ed815&scene=21#wechat_redirect)
>
> [Java8 CompletableFuture 用法全解](https://blog.csdn.net/qq_31865983/article/details/106137777?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-106137777-blog-107577226.pc_relevant_multi_platform_whitelistv1&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-106137777-blog-107577226.pc_relevant_multi_platform_whitelistv1&utm_relevant_index=1)