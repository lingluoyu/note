### Runnable/Thread

常规开启线程方式

```java
public class ThreadTest {
    public static void main(String[] args) {
        //Thread方式
        new ThreadDemo().start();
        //Runnable方式
        new Thread(new RunnableDemo()).start();

        //Lamda方式
        new Thread(() -> {
            System.out.println("Lamda 当前线程：" + Thread.currentThread().getName());
        }).start();
    }

    static class ThreadDemo extends Thread{
        @Override
        public void run() {
            System.out.println("Thread 当前线程：" + Thread.currentThread().getName());
        }
    }

    static class RunnableDemo implements Runnable{
        @Override
        public void run() {
            System.out.println("Runnable 当前线程：" + Thread.currentThread().getName());
        }
    }
}
```

输出：

```bash
Thread 当前线程：Thread-0
Runnable 当前线程：Thread-1
Lamda 当前线程：Thread-2
```

Thread和Runnable无法获取线程返回值，线程执行成功还是失败无法得知。

### Callable

`Callable`是`java.util.concurrent`包下的一个接口：

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

`Callable`是一个接口，一个函数式接口，也是个泛型接口。`call()`有返回值，且返回值类型与泛型参数类型相同，且可以抛出异常。`Callable`可以看作是`Runnable`接口的补充。

### Future

`Future`是`java.util.concurrent`包下的一个接口，主要方法定义如下：

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

`Future`可以看作是一个句柄，即`Callable`任务返回给调用方这么一个句柄，通过这个句柄我们可以跟这个异步任务联系起来，可以通过`future`来对任务查询、取消、执行结果的获取，是调用方与异步执行方之间沟通的桥梁。

可以通过`get()`方法获取异步任务的执行结果，**这个方法是阻塞的，直到异步任务执行完成。**

### FutureTask

`Future`是一个接口，我们无法直接使用它，需要通过它的实现类来操作。`FutureTask`就是JDK提供给我们的一个实现类。

```java
public class FutureTask<V> implements RunnableFuture<V> {
    ......
        
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
    
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

    ......
}
```

`FutureTask`实现了`RunnableFuture`接口，同时具有`Runnable`、`Future`的能力，即既可以作为`Future`得到`Callable`的返回值，又可以作为一个`Runnable`。

`FutureTask`使用示例（主线程通过`FutureTask.get()`获取其他线程返回的数据）：

```java
public class FutureTest {
    public static void main(String[] args) {
        try {
            FutureTask<String> thread1Task = new FutureTask<>(new Thread1());
            Thread thread1 = new Thread(thread1Task);
            thread1.start();

            FutureTask<String> thread2Task = new FutureTask<>(new Thread2());
            Thread thread2 = new Thread(thread2Task);
            thread2.start();

            System.out.println(thread1Task.get());
            System.out.println(thread2Task.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static class Thread1 implements Callable {
        @Override
        public String call() throws Exception {
            return "Thread1 当前线程：" + Thread.currentThread().getName();
        }
    }

    static class Thread2 implements Callable {
        @Override
        public String call() throws Exception {
            return "Thread2 当前线程：" + Thread.currentThread().getName();
        }
    }
}
```

运行结果：

```bash
Thread1 当前线程：Thread-0
Thread2 当前线程：Thread-1
```

### CompletableFuture

`CompletableFuture`是Java8新增加的，用于异步编程。异步编程是编写非阻塞的代码，运行的任务在一个单独的线程，与主线程隔离，并且会通知主线程它的进度，成功或者失败。

在这种方式中，主线程不会被阻塞，不需要一直等到子线程完成。主线程可以并行的执行其他任务。

**`CompletableFuture`常用方法**

> runAsync()、supplyAsync()：运行异步计算，其中后者可以返回计算结果
>
> thenApply、thenApplyAsync: 假如任务执行完成后，还需要后续的操作，比如返回结果的解析等等；可以通过这两个方法来完成
>
> thenCompose、thenComposeAsync: 允许你对两个异步操作进行流水线的操作，当第一个操作完成后，将其结果传入到第二个操作中
>
> thenCombine、thenCombineAsync：允许你把两个异步的操作整合；比如把第一个和第二个操作返回的结果做字符串的连接操作
>
> allOf()：在所有任务完成后做一些操作
>
> anyOf()：当任何一个任务完成后，返回一个新的CompletableFuture
>
> exceptionally()：回调处理异常
>
> handle()：处理异常，无论一个异常是否发生它都会被调用。

设想一个场景，需要远程调用两个接口：

```java
public interface RemoteApi {
    String call();

    default void delay() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class UserService implements RemoteApi{
    @Override
    public String call() {
        delay();
        return "用户";
    }
}

public class OrderService implements RemoteApi{
    @Override
    public String call() {
        delay();
        return "订单";
    }
}
```

**同步方式调用**：

```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    List<RemoteApi> remoteApis = Arrays.asList(new UserService(), new OrderService());
    List<String> collect = remoteApis.stream().map(RemoteApi::call).collect(Collectors.toList());
    System.out.println(collect);
    long end = System.currentTimeMillis();
    System.out.println("用时:" + (end - start));
}
```

```shell
[用户, 订单]
用时:2073
```

用时超过2秒

**Future实现**：

```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    ExecutorService executorService = Executors.newFixedThreadPool(2);
    List<RemoteApi> remoteApis = Arrays.asList(new UserService(), new OrderService());
    List<Future<String>> futures = remoteApis.stream()
        .map(remoteApi -> executorService.submit(remoteApi::call))
        .collect(Collectors.toList());

    List<String> collect = futures.stream()
        .map(future -> {
            try {
                return future.get();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return null;
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
    System.out.println(collect);
    long end = System.currentTimeMillis();
    System.out.println("用时:" + (end - start));
}
```

```shell
[用户, 订单]
用时:1103
```

用时小于2秒

> 此处分成了两个Stream，如何合在一起用同一个Stream，那么在用`future.get()`的时候会导致阻塞，相当于提交一个任务执行完后才提交下一个任务，这样达不到异步的效果

虽然`Future`达到了我们预期的效果，但是如果需要实现将两个异步的结果进行合并处理很繁琐。

**Java8并行流**：

```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    List<RemoteApi> remoteApis = Arrays.asList(new UserService(), new OrderService());
    List<String> collect = remoteApis.parallelStream().map(RemoteApi::call).collect(Collectors.toList());
    System.out.println(collect);
    long end = System.currentTimeMillis();
    System.out.println("用时:" + (end - start));
}
```

```shell
[用户, 订单]
用时:1070
```

用时小于2秒

> 但是当需要运行的任务数量增加时，Java8并行流的效率就不太理想
>
> 并行流和`CompletableFuture`底层使用的线程池的大小都是CPU的核数`Runtime.getRuntime().availableProcessors()`，因此当线程数超过CPU核心数时，运行效率就会降低

**CompletableFuture**

```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    List<RemoteApi> remoteApis = Arrays.asList(new UserService(), new OrderService());
    List<CompletableFuture<String>> completableFutures = remoteApis.stream().
        map(api -> CompletableFuture.supplyAsync(api::call)).collect(Collectors.toList());
    List<String> collect = completableFutures.stream().map(CompletableFuture::join).collect(Collectors.toList());
    System.out.println(collect);
    long end = System.currentTimeMillis();
    System.out.println("用时:" + (end - start));
}
```

```shell
[用户, 订单]
用时:1100
```

用时小于2秒

**自定义线程池，优化CompletableFuture**

```java
public static void main(String[] args) {
    long start = System.currentTimeMillis();
    List<RemoteApi> remoteApis = Arrays.asList(new UserService(), new OrderService());
    ExecutorService executorService = Executors.newFixedThreadPool(Math.min(remoteApis.size(), 50));
    List<CompletableFuture<String>> completableFutures = remoteApis.stream().
        map(api -> CompletableFuture.supplyAsync(api::call, executorService)).collect(Collectors.toList());
    executorService.shutdown();
    List<String> collect = completableFutures.stream().map(CompletableFuture::join).collect(Collectors.toList());
    System.out.println(collect);
    long end = System.currentTimeMillis();
    System.out.println("用时:" + (end - start));
}
```

> 并行流和`CompletableFuture`两者该如何选择
>
> 1. 如果你的任务是计算密集型的，并且没有I/O操作的话，那么推荐你选择Stream的并行流，实现简单并行效率也是最高的
> 2. 如果你的任务是有频繁的I/O或者网络连接等操作，那么推荐使用`CompletableFuture`，采用自定义线程池的方式，根据服务器的情况设置线程池的大小，尽可能的让CPU忙碌起来

**处理异常**

```java
public class OrderService implements RemoteApi{
    @Override
    public String call() {
        delay();
        if (true) {
            throw new NullPointerException("异常");
        }
        return "订单";
    }
}

List<CompletableFuture<String>> completableFutures = remoteApis.stream().
                map(api -> CompletableFuture.supplyAsync(api::call, executorService).exceptionally(e -> e.getMessage())).collect(Collectors.toList());
```

```shell
[用户, java.lang.NullPointerException: 异常]
用时:1108
```

