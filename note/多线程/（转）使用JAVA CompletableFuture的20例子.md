# [使用JAVA CompletableFuture的20例子](https://segmentfault.com/a/1190000013452165)

## 前言

这篇博客回顾JAVA8的`CompletionStage`API以及其在JAVA库中的标准实现`CompletableFuture`。将会通过几个例子来展示API的各种行为。

因为`CompletableFuture`是`CompletionInterface`接口的实现，所以我们首先要了解该接口的契约。它代表某个同步或异步计算的一个阶段。你可以把它理解为是一个为了产生有价值最终结果的计算的流水线上的一个单元。这意味着多个`ComletionStage`指令可以链接起来从而一个阶段的完成可以触发下一个阶段的执行。

除了实现了`CompletionStage`接口，`Completion`还继承了`Future`，这个接口用于实现一个未开始的异步事件。因为能够显式的完成`Future`，所以取名为`CompletableFuture`。

## 1.新建一个完成的CompletableFuture

这个简单的示例中创建了一个已经完成的预先设置好结果的`CompletableFuture`。通常作为计算的起点阶段。

```
static void completedFutureExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message");
    assertTrue(cf.isDone());
    assertEquals("message", cf.getNow(null));
}
```

`getNow`方法会返回完成后的结果（这里就是message），如果还未完成，则返回传入的默认值`null`。

## 2.运行一个简单的异步stage

下面的例子解释了如何创建一个异步运行`Runnable`的stage。

```
static void runAsyncExample() {
    CompletableFuture cf = CompletableFuture.runAsync(() -> {
        assertTrue(Thread.currentThread().isDaemon());
        randomSleep();
    });
    assertFalse(cf.isDone());
    sleepEnough();
    assertTrue(cf.isDone());
}
```

这个例子想要说明两个事情：

1. `CompletableFuture`中以`Async`为结尾的方法将会异步执行
2. 默认情况下（即指没有传入`Executor`的情况下），异步执行会使用`ForkJoinPool`实现，该线程池使用一个后台线程来执行`Runnable`任务。注意这只是特定于`CompletableFuture`实现，其它的`CompletableStage`实现可以重写该默认行为。

## 3.将方法作用于前一个Stage

下面的例子引用了第一个例子中已经完成的`CompletableFuture`，它将引用生成的字符串结果并将该字符串大写。

```
static void thenApplyExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApply(s -> {
        assertFalse(Thread.currentThread().isDaemon());
        return s.toUpperCase();
    });
    assertEquals("MESSAGE", cf.getNow(null));
}
```

这里的关键词是`thenApply`：

1. `then`是指在当前阶段正常执行完成后（正常执行是指没有抛出异常）进行的操作。在本例中，当前阶段已经完成并得到值`message`。
2. `Apply`是指将一个`Function`作用于之前阶段得出的结果

`Function`是阻塞的，这意味着只有当大写操作执行完成之后才会执行`getNow()`方法。

## 4.异步的的将方法作用于前一个Stage

通过在方法后面添加`Async`后缀，该`CompletableFuture`链将会异步执行（使用ForkJoinPool.commonPool()）

```
static void thenApplyAsyncExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
        assertTrue(Thread.currentThread().isDaemon());
        randomSleep();
        return s.toUpperCase();
    });
    assertNull(cf.getNow(null));
    assertEquals("MESSAGE", cf.join());
}
```

## 使用一个自定义的Executor来异步执行该方法

异步方法的一个好处是可以提供一个`Executor`来执行`CompletableStage`。这个例子展示了如何使用一个固定大小的线程池来实现大写操作。

```
static ExecutorService executor = Executors.newFixedThreadPool(3, new ThreadFactory() {
    int count = 1;
    @Override
    public Thread newThread(Runnable runnable) {
        return new Thread(runnable, "custom-executor-" + count++);
    }
});
static void thenApplyAsyncWithExecutorExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
        assertTrue(Thread.currentThread().getName().startsWith("custom-executor-"));
        assertFalse(Thread.currentThread().isDaemon());
        randomSleep();
        return s.toUpperCase();
    }, executor);
    assertNull(cf.getNow(null));
    assertEquals("MESSAGE", cf.join());
}
```

## 6.消费(Consume)前一个Stage的结果

如果下一个Stage接收了当前Stage的结果但是在计算中无需返回值（比如其返回值为void），那么它将使用方法`thenAccept`并传入一个`Consumer`接口。

```
static void thenAcceptExample() {
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture("thenAccept message")
            .thenAccept(s -> result.append(s));
    assertTrue("Result was empty", result.length() > 0);
}
```

`Consumer`将会同步执行，所以我们无需在返回的`CompletableFuture`上执行join操作。

## 7.异步执行Comsume

同样，使用Asyn后缀实现：

```
static void thenAcceptAsyncExample() {
    StringBuilder result = new StringBuilder();
    CompletableFuture cf = CompletableFuture.completedFuture("thenAcceptAsync message")
            .thenAcceptAsync(s -> result.append(s));
    cf.join();
    assertTrue("Result was empty", result.length() > 0);
}
```

## 8.计算出现异常时

我们现在来模拟一个出现异常的场景。为了简洁性，我们还是将一个字符串大写，但是我们会模拟延时进行该操作。我们会使用`thenApplyAsyn(Function, Executor)`，第一个参数是大写转化方法，第二个参数是一个延时executor，它会延时一秒钟再将操作提交给`ForkJoinPool`。

```
static void completeExceptionallyExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(String::toUpperCase,
            CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS));
    CompletableFuture exceptionHandler = cf.handle((s, th) -> { return (th != null) ? "message upon cancel" : ""; });
    cf.completeExceptionally(new RuntimeException("completed exceptionally"));
    assertTrue("Was not completed exceptionally", cf.isCompletedExceptionally());
    try {
        cf.join();
        fail("Should have thrown an exception");
    } catch(CompletionException ex) { // just for testing
        assertEquals("completed exceptionally", ex.getCause().getMessage());
    }
    assertEquals("message upon cancel", exceptionHandler.join());
}
```

1. 首先，我们新建了一个已经完成并带有返回值`message`的`CompletableFuture`对象。然后我们调用`thenApplyAsync`方法，该方法会返回一个新的`CompletableFuture`。这个方法用异步的方式执行大写操作。这里还展示了如何使用`delayedExecutor(timeout, timeUnit)`方法来延时异步操作。
2. 然后我们创建了一个handler stage，`exceptionHandler`，这个阶段会处理一切异常并返回另一个消息`message upon cancel`。
3. 最后，我们显式的完成第二个阶段并抛出异常，它会导致进行大写操作的阶段抛出`CompletionException`。它还会触发`handler`阶段。

> API补充：
> `<U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn)`
> 返回一个新的CompletionStage，无论之前的Stage是否正常运行完毕。传入的参数包括上一个阶段的结果和抛出异常。

## 9.取消计算

和计算时异常处理很相似，我们可以通过`Future`接口中的`cancel(boolean mayInterruptIfRunning)`来取消计算。

```
static void cancelExample() {
    CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(String::toUpperCase,
            CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS));
    CompletableFuture cf2 = cf.exceptionally(throwable -> "canceled message");
    assertTrue("Was not canceled", cf.cancel(true));
    assertTrue("Was not completed exceptionally", cf.isCompletedExceptionally());
    assertEquals("canceled message", cf2.join());
}
```

> API补充
> `public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)`
> 返回一个新的CompletableFuture，如果出现异常，则为该方法中执行的结果，否则就是正常执行的结果。

## 10.将Function作用于两个已完成Stage的结果之一

下面的例子创建了一个`CompletableFuture`对象并将`Function`作用于已完成的两个Stage中的任意一个（没有保证哪一个将会传递给Function）。这两个阶段分别如下：一个将字符串大写，另一个小写。

```
static void applyToEitherExample() {
    String original = "Message";
    CompletableFuture cf1 = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s));
    CompletableFuture cf2 = cf1.applyToEither(
            CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
            s -> s + " from applyToEither");
    assertTrue(cf2.join().endsWith(" from applyToEither"));
}
```

> `public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T,U> fn)`
> 返回一个全新的CompletableFuture，包含着this或是other操作完成之后，在二者中的任意一个执行fn

## 11.消费两个阶段的任意一个结果

和前一个例子类似，将`Function`替换为`Consumer`

```
static void acceptEitherExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture cf = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s))
            .acceptEither(CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
                    s -> result.append(s).append("acceptEither"));
    cf.join();
    assertTrue("Result was empty", result.toString().endsWith("acceptEither"));
}
```

## 12.在两个阶段都完成后运行Runnable

注意这里的两个Stage都是同步运行的，第一个stage将字符串转化为大写之后，第二个stage将其转化为小写。

```
static void runAfterBothExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).runAfterBoth(
            CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
            () -> result.append("done"));
    assertTrue("Result was empty", result.length() > 0);
}
```

## 13.用Biconsumer接收两个stage的结果

Biconsumer支持同时对两个Stage的结果进行操作。

```
static void thenAcceptBothExample() {
    String original = "Message";
    StringBuilder result = new StringBuilder();
    CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).thenAcceptBoth(
            CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
            (s1, s2) -> result.append(s1 + s2));
    assertEquals("MESSAGEmessage", result.toString());
}
```

## 14.将Bifunction同时作用于两个阶段的结果

如果`CompletableFuture`想要合并两个阶段的结果并且返回值，我们可以使用方法`thenCombine`。这里的计算流都是同步的，所以最后的`getNow()`方法会获得最终结果，即大写操作和小写操作的结果的拼接。

```
static void thenCombineExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original).thenApply(s -> delayedUpperCase(s))
            .thenCombine(CompletableFuture.completedFuture(original).thenApply(s -> delayedLowerCase(s)),
                    (s1, s2) -> s1 + s2);
    assertEquals("MESSAGEmessage", cf.getNow(null));
}
```

## 15.异步将Bifunction同时作用于两个阶段的结果

和之前的例子类似，只是这里用了不同的方法：即两个阶段的操作都是异步的。那么`thenCombine`也会异步执行，及时它没有Async后缀。

```
static void thenCombineAsyncExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original)
            .thenApplyAsync(s -> delayedUpperCase(s))
            .thenCombine(CompletableFuture.completedFuture(original).thenApplyAsync(s -> delayedLowerCase(s)),
                    (s1, s2) -> s1 + s2);
    assertEquals("MESSAGEmessage", cf.join());
}
```

## 16.Compose CompletableFuture

我们可以使用`thenCompose`来完成前两个例子中的操作。

```
static void thenComposeExample() {
    String original = "Message";
    CompletableFuture cf = CompletableFuture.completedFuture(original).thenApply(s -> delayedUpperCase(s))
            .thenCompose(upper -> CompletableFuture.completedFuture(original).thenApply(s -> delayedLowerCase(s))
                    .thenApply(s -> upper + s));
    assertEquals("MESSAGEmessage", cf.join());
}
```

## 17.当多个阶段中有有何一个完成，即新建一个完成阶段

```
static void anyOfExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApply(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture.anyOf(futures.toArray(new CompletableFuture[futures.size()])).whenComplete((res, th) -> {
        if(th == null) {
            assertTrue(isUpperCase((String) res));
            result.append(res);
        }
    });
    assertTrue("Result was empty", result.length() > 0);
}
```

## 18.当所有的阶段完成，新建一个完成阶段

```
static void allOfExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApply(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()]))
        .whenComplete((v, th) -> {
            futures.forEach(cf -> assertTrue(isUpperCase(cf.getNow(null))));
            result.append("done");
        });
    assertTrue("Result was empty", result.length() > 0);
}
```

## 19.当所有阶段完成以后，新建一个异步完成阶段

```
static void allOfAsyncExample() {
    StringBuilder result = new StringBuilder();
    List messages = Arrays.asList("a", "b", "c");
    List<CompletableFuture> futures = messages.stream()
            .map(msg -> CompletableFuture.completedFuture(msg).thenApplyAsync(s -> delayedUpperCase(s)))
            .collect(Collectors.toList());
    CompletableFuture allOf = CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()]))
            .whenComplete((v, th) -> {
                futures.forEach(cf -> assertTrue(isUpperCase(cf.getNow(null))));
                result.append("done");
            });
    allOf.join();
    assertTrue("Result was empty", result.length() > 0);
}
```

## 20.真实场景

下面展示了一个实践CompletableFuture的场景：

1. 先通过调用`cars()`方法异步获得`Car`列表。它将会返回一个`CompletionStage<List<Car>>`。`cars()`方法应当使用一个远程的REST端点来实现。
2. 我们将该Stage和另一个Stage组合，另一个Stage会通过调用`rating(manufactureId)`来异步获取每辆车的评分。
3. 当所有的Car对象都填入评分后，我们调用`allOf()`来进入最终Stage，它将在这两个阶段完成后执行
4. 在最终Stage上使用`whenComplete()`，打印出车辆的评分。

```
cars().thenCompose(cars -> {
    List<CompletionStage> updatedCars = cars.stream()
            .map(car -> rating(car.manufacturerId).thenApply(r -> {
                car.setRating(r);
                return car;
            })).collect(Collectors.toList());
    CompletableFuture done = CompletableFuture
            .allOf(updatedCars.toArray(new CompletableFuture[updatedCars.size()]));
    return done.thenApply(v -> updatedCars.stream().map(CompletionStage::toCompletableFuture)
            .map(CompletableFuture::join).collect(Collectors.toList()));
}).whenComplete((cars, th) -> {
    if (th == null) {
        cars.forEach(System.out::println);
    } else {
        throw new RuntimeException(th);
    }
}).toCompletableFuture().join();
```

## 参考资料

> [Java CompletableFuture 详解](http://colobu.com/2016/02/29/Java-CompletableFuture/)
> [Guide To CompletableFuture](http://www.baeldung.com/java-completablefuture)