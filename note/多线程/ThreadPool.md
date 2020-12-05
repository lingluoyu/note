### 线程池
**使用现线程池的理由**
* 1. 降低资源消耗。线程是操作系统十分宝贵的资源，当多个人同时开发一个项目时，在互不知情的情况下，都自己在代码中创建了线程，这样就会导致线程数过多，而且线程的创建和销毁，在操作系统层面，需要由用户态切换到内核态，这是一个费时费力的过程。而使用线程池可以避免频繁的创建线程和销毁线程，线程池中线程可以重复使用。
* 2. 提高响应速度。当请求到达时，由于线程池中的线程已经创建好了，使用线程池，可以省去线程创建的这段时间。
* 3. 提高线程的可管理性。线程是稀缺资源，当创建过多的线程时，会造成系统性能的下降，而使用线程池，可以对线程进行统一分配、调优和监控。 线程池的使用十分简单，但是会用不代表用得好。在面试中，基本不会问线程池应该怎么用，而是问线程池在使用不当时会造成哪些问题，实际上就是考察线程池的实现原理。因此搞明白线程池的实现原理是很有必要的一件事，不仅仅对面试会有帮助，也会让我们在平时工作中避过好多坑。

**实现原理**
![线程池执行任务流程](https://user-gold-cdn.xitu.io/2019/11/19/16e7f863984d32d4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

* 1. 先判断线程池中线程的数量是否超过核心线程数，如果没有超过核心线程数，就创建新的线程去执行任务；如果超过了核心线程数，就进入到下面流程。
* 2. 判断任务队列是否已经满了，如果没有满，就将任务添加到任务队列中；如果已经满了，就进入到下面的流程。
* 3. 再判断如果创建一个线程后，线程数是否会超过最大线程数，如果不会超过最大线程数，就创建一个新的线程来执行任务；如果会，则进入到下面的流程。
* 4. 执行拒绝策略。

**ThreadPoolExecutor**

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
}
```
* corePoolSize：该参数表示的是线程池的核心线程数。当任务提交到线程池时，如果线程池的线程数量还没有达到corePoolSize，那么就会新创建的一个线程来执行任务，如果达到了，就将任务添加到任务队列中。

* maximumPoolSize：该参数表示的是线程池中允许存在的最大线程数量。当任务队列满了以后，再有新的任务进入到线程池时，会判断再新建一个线程是否会超过maximumPoolSize，如果会超过，则不创建线程，而是执行拒绝策略。如果不会超过maximumPoolSize，则会创建新的线程来执行任务。

* keepAliveTime：当线程池中的线程数量大于corePoolSize时，那么大于corePoolSize这部分的线程，如果没有任务去处理，那么就表示它们是空闲的，这个时候是不允许它们一直存在的，而是允许它们最多空闲一段时间，这段时间就是keepAliveTime，时间的单位就是TimeUnit。

* unit：空闲线程允许存活时间的单位，TimeUnit是一个枚举值，它可以是纳秒、微妙、毫秒、秒、分、小时、天。

* workQueue：任务队列，用来存放任务。该队列的类型是阻塞队列，常用的阻塞队列有ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、PriorityBlockingQueue等。
>* ArrayBlockingQueue是一个基于数组实现的阻塞队列，元素按照先进先出（FIFO）的顺序入队、出队。因为底层实现是数组，数组在初始化时必须指定大小，因此ArrayBlockingQueue是有界队列。
>* LinkedBlockingQueue是一个基于链表实现的阻塞队列，元素按照先进先出（FIFO）的顺序入队、出队。因为顶层是链表，链表是基于节点之间的指针指向来维持前后关系的，如果不指链表的大小，它默认的大小是Integer.MAX_VALUE，即，这个数值太大了，因此通常称LinkedBlockingQueue是一个无界队列。当然如果在初始化的时候，就指定链表大小，那么它就是有界队列了。
>* SynchronousQueue是一个不存储元素的阻塞队列。每个插入操作必须得等到另一个线程调用了移除操作后，该线程才会返回，否则将一直阻塞。吞吐量通常要高于LinkedBlockingQueue。
>* PriorityBlockingQueue是一个将元素按照优先级排序的阻塞的阻塞队列，元素的优先级越高，将会越先出队列。这是一个无界队列。

* threadFactory：线程池工厂，用来创建线程。通常在实际项目中，为了便于后期排查问题，在创建线程时需要为线程赋予一定的名称，通过线程池工厂，可以方便的为每一个创建的线程设置具有业务含义的名称。

* handler：拒绝策略。当任务队列已满，线程数量达到maximumPoolSize后，线程池就不会再接收新的任务了，这个时候就需要使用拒绝策略来决定最终是怎么处理这个任务。默认情况下使用AbortPolicy，表示无法处理新任务，直接抛出异常。在ThreadPoolExecutor类中定义了四个内部类，分别表示四种拒绝策略。我们也可以通过实现RejectExecutionHandler接口来实现自定义的拒绝策略。
> AbortPocily：不再接收新任务，直接抛出异常。
> CallerRunsPolicy：提交任务的线程自己处理。
> DiscardPolicy：不处理，直接丢弃。
> DiscardOldestPolicy：丢弃任务队列中排在最前面的任务，并执行当前任务。（排在队列最前面的任务并不一定是在队列中待的时间最长的任务，因为有可能是按照优先级排序的队列）