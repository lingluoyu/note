### 阻塞队列

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作支持阻塞的插入和移除方法。

1. 支持阻塞的插入方法：当队列满时，队列会阻塞插入元素的线程，直到队列不满。
2. 支持阻塞的移除方法：在队列为空时，获取元素的线程会等待队列变为非空

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

在阻塞队列不可用时，这两个附加操作提供了4种处理方式，如下所示：

| 方法/处理方式 | 抛出异常  | 返回特殊值 | 阻塞   | 超时退出             |
| ------------- | --------- | ---------- | ------ | -------------------- |
| 插入          | add(e)    | offer(e)   | put(e) | offer(e, time, unit) |
| 移除          | remove()  | poll()     | take() | poll(time, unit)     |
| 检查          | element() | peek()     | 不可用 | 不可用               |

- 抛出异常：当队列满时，如果再往队列里插入元素，会抛出IllegalArgumentException异常。当队列空时，从队列里获取元素会抛出NoSuchElementException异常。
- 返回特殊值：当往队列插入元素时，会返回元素是否插入成功，成功返回true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回`null`。
- 一直阻塞：当阻塞队列满时，如果生产者线程往队列里`put`元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者从队列里`take`元素，队列会阻塞住消费者线程，直到队列不为空。

> **tips**:如果是无界阻塞队列，队列不可能会出现满的情况，所以使用put或offer方法永远不会被阻塞，而且使用offer方法时，该方法永远返回true。

阻塞队列一共有7种：

![BlockingQueue](https://gitee.com/LoopSup/image/raw/master/img/BlockingQueue.png)

#### ArrayBlockingQueue

基于数组实现有界的阻塞队列(循环数组)，此队列按照先进先出的原则对元素进行排序。

ArrayBlockingQueue 构造方法：

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
......
    // 默认使用非公平锁
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and the specified access policy.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    // 参数fair用于设置线程是否公平访问队列。所谓公平访问是指阻塞的线程，可以按照阻塞的先后顺序访问队列，即先阻塞线程先访问队列。非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格，有可能先阻塞的线程最后才访问队列。为了保证公平性，通常会降低吞吐量。
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity, the specified access policy and initially containing the
     * elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @param c the collection of elements to initially contain
     * @throws IllegalArgumentException if {@code capacity} is less than
     *         {@code c.size()}, or less than 1.
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);

        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }
......
}
```

put、take方法：

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    // 队列不为空时，通知消费者获取元素
    notEmpty.signal();
}
```
1. 首先判断添加的元素是否非空，是空的会抛出异常。
2. 给put方法上锁
3. 当集合元素数量和集合的长度相等时，调用put方法的线程将会被放入notFull队列上等待
4. 如果不相等，则enqueue()，也就是让e进入队列，并唤醒notEmpty队列。

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    // 队列未满，通知生产者放入元素
    notFull.signal();
    return x;
}
```

1. 给take方法上锁
2. 若队列中元素为0，则将调用take方法的线程放入notEmpty队列等待
3. 若不为空，则dequeue()，即从阻塞队列中取数据，并唤醒notFull队列。

#### LinkedBlockingQueue

基于链表实现

LinkedBlockingQueue构造方法：

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    
    private transient Node<E> last;//尾节点

    transient Node<E> head;//头节点

    private final AtomicInteger count = new AtomicInteger();//计算当前阻塞队列中的元素个数 

    private final int capacity;//容量

    //获取并移除元素时使用的锁，如take, poll, etc
    private final ReentrantLock takeLock = new ReentrantLock();

    //notEmpty条件对象，当队列没有数据时用于挂起执行删除的线程
    private final Condition notEmpty = takeLock.newCondition();

    //添加元素时使用的锁如 put, offer, etc
    private final ReentrantLock putLock = new ReentrantLock();

    //notFull条件对象，当队列数据已满时用于挂起执行添加的线程
    private final Condition notFull = putLock.newCondition();
......
    // 默认容量为最大容量
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of the
     * given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    // 将整个集合放入队列中，默认容量同样是最大容量。
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
......
}
```

LinkedBlockingQueue拥有两个全局锁，一个负责放元素，一个负责取元素。

- ArrayBlockingQueue和LinkedBlockingQueue的区别和联系？
  - 数据存储容器不一样，ArrayBlockingQueue采用数组去存储数据、LinkedBlockingQueue采用链表去存储数据。
  - ArrayBlockingQueue(循环数组)采用数组去存储数据，不会产生额外的对象实例； LinkedBlockingQueue采用链表去存储数据，在插入和删除元素只与一个节点有关，需要去生成一个额外的Node对象，这可能长时间内需要并发处理大批量的数据，对于性能和后期GC会产生影响。
  - ArrayBlockingQueue是有界的，初始化时必须要指定容量；LinkedBlockingQueue默认是无界的，Integer.MAX_VALUE, 当添加速度大于删除速度、有可能造成内存溢出。
  - ArrayBlockingQueue在读和写使用的锁是一样的，即添加操作和删除操作使用的是同一个ReentrantLock，没有实现锁分离；LinkedBlockingQueue实现了锁分离，添加的时候采用putLock、删除的时候采用takeLock,这样能提高队列的吞吐量。

- ArrayBlockingQueue可以使用两把锁提高效率吗？ 
  - 不能，主要原因是ArrayBlockingQueue底层循环数组来存储数据，LinkedBlockingQueue底层 链表来存储数据，链表队列的添加和删除，只是和某一个节点有关，为了防止head和last相互影响，就需要有一个原子性的计数器，每个添加操作先加入队列，计数器+1，这样是为了保证队列在移除的时候， 长度是大于等于计数器的，通过原子性的计数器，使得当前添加和移除互不干扰。对于循环数据来说，当我们走到最后一个位置需要返回到第一个位置，这样的操作是无法原子化，只能使用同一把锁来解决。

#### DelayQueue

DelayQueue是一个基于PriorityQueue的支持延时获取元素的无界阻塞队列。DelayQueue中的元素只有当其延时时间到达，才能够去当前队列中获取到该元素。

实现DelayQueue的三个步骤

- 继承Delayed接口
- 实现getDelay(TimeUnit unit)，该方法返回当前元素还需要延时多长时间，单位是纳秒
- 实现compareTo方法来指定元素的顺序 

```java
/** 控制台输出
* begin time: 2021-04-29T14:43:56.2048297
* time: 2021-04-29T14:44:01.1818432
* time: 2021-04-29T14:44:06.1814344
* time: 2021-04-29T14:44:11.1810123
*/
class Test implements Delayed {
    private long time; //Test实例延时时间

    public Test(long time, TimeUnit unit){
        this.time = System.currentTimeMillis() + unit.toMillis(time);
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return this.time - System.currentTimeMillis();
    }

    @Override
    public int compareTo(Delayed o) {
        long diff = this.time - ((Test)o).time;
        if(diff <= 0){
            return -1;
        }else{
            return 1;
        }
    }

    public static void main(String[] args) {
        DelayQueue<Test> queue = new DelayQueue<>();
        queue.put(new Test(5, TimeUnit.SECONDS));
        queue.put(new Test(10, TimeUnit.SECONDS));
        queue.put(new Test(15, TimeUnit.SECONDS));

        System.out.println("begin time: "+ LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
        for(int i=0; i<3; i++){
            try {
                Test test = queue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("time: "+LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
        }
    }
}
```

DelayQueue运用在以下应用场景：

- 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
- 任务超时处理：比如下单后15分钟内未付款，自动关闭订单。

#### PriorityBlockingQueue

PriorityBlockingQueue是一个支持优先级的无界阻塞队列。默认情况下元素采取自然顺序升序排列。也可以自定义类实现`compareTo()`方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数`Comparator`来进行排序。需要注意的是不能保证同优先级元素的顺序。

#### SynchronousQueue

SynchronousQueue是一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。它支持公平访问队列。默认情况下线程采用非公平性策略访问队列。

SynchronousQueue可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合传递性场景。SynchronousQueue的吞吐量高于LinkedBlockingQueue和ArrayBlockingQueue.

#### LinkedTransferQueue

LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。

- `transfer`方法

  如果当前有消费者正在等待接收元素（消费者使用take()方法或带时间限制的poll()方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer方法会将元素存放在队列的tail节点，并等到该元素被消费者了才返回。

- `tryTransfer`方法

  tryTransfer方法时用来试探生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回fasle。和transfer方法的区别是tryTransfer方法无论消费者是否接收，方法立即返回，而transfer方法是必须等到消费者消费了才返回。

#### LinkedBlockingDeque

LinkedBlockingDeque是一个由链表结构组成的双向阻塞队列。所谓双向队列指的是可以从队列的两端插入和移出元素。双向队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。相比其他的阻塞队列，LinkedBlockingDeque多了`addFirst`、`addLast`、`offerFirst`、`offerLast`、`peekFirst`和`peekLast`等方法，以First单词结尾的方法，表示插入、获取（peek）或移除双端队列的第一个元素。以Last单词结尾的方法，表示插入、获取或移除双向队列的最后一个元素。

