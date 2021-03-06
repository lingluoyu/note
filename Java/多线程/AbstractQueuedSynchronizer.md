### AQS主要原理

AQS核心思想，若被请求的资源当前为空闲状态，则将当前请求资源的线程设置为有效的工作线程，并将共享资源状态设置为锁定；当前请求的资源被占用时，其他请求资源的线程需要被阻塞，这种阻塞线程以及被唤醒是获取锁的机制通过  CLH 队列实现，暂时获取不到锁（资源）的线程将被加入到此队列中。

AQS原理图

![AQS-1](https://gitee.com/LoopSup/image/raw/master/img/aqs-1.jpg)

### AQS主要结构

```java
//等待队列头结点
private transient volatile Node head;

// 阻塞的尾节点，新节点插入到最后
private transient volatile Node tail;

//表示资源的状态，state=0表示未占用，state>0表示有线程持有当前锁，state可以大于1，因为锁可重入，重入state+1
//使用 volatile 配合 cas 保证其修改时的原子性，使用了 32bit int 来维护同步状态
private volatile int state;

//当前持有独占锁的线程
private transient Thread exclusiveOwnerThread;
```

AQS等待队列如下图

![aqs-0](https://gitee.com/LoopSup/image/raw/master/img/20201205120210.png)**Node**结构

```java
static final class Node {
    //表示线程以共享的模式等待锁
    static final Node SHARED = new Node();
    //表示线程正在以独占的方式等待锁
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    //表示线程获取锁的请求已经取消了
    //此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    //表示线程已经准备好了，就等资源释放了
    //当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    //表示节点在等待队列中，节点线程等待唤醒
    static final int CONDITION = -2;
    //当前线程处在SHARED情况下，该字段才会使用
    static final int PROPAGATE = -3;
	// =====================================================
    
    //取值为0,1，-1，-2，-3
    //如果这个值 大于0 代表此线程取消了等待
    volatile int waitStatus;

    //前驱节点的引用
    volatile Node prev;

    //后继节点的引用
    volatile Node next;

    //此节点代表的线程
    volatile Thread thread;

    //指向下一个处于CONDITION状态的节点
    Node nextWaiter;    
}
```

**同步状态state**

```java
private volatile int state;
//获取State的值
protected final int getState() {
    return state;
}

//设置State的值
protected final void setState(int newState) {
    state = newState;
}

//使用CAS方式更新State
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

通过修改State字段表示的同步状态来实现多线程的独占模式和共享模式（加锁过程）

![27605d483e8935da683a93be015713f331378](https://gitee.com/LoopSup/image/raw/master/img/20201205121629.png)![3f1e1a44f5b7d77000ba4f9476189b2e32806](https://gitee.com/LoopSup/image/raw/master/img/20201205121639.png)

### ReentrantLock

ReentrantLock 在内部用了内部类 Sync 来管理锁，所以真正的获取锁和释放锁是由 Sync 的实现类来控制的。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
}
```

Sync 有两个实现，分别为 NonfairSync（非公平锁）和 FairSync（公平锁），我们看 FairSync 部分。

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

#### 加锁

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    // 争锁
    final void lock() {
        acquire(1);
    }
    // 来自父类AQS
    // 如果tryAcquire(arg) 返回true, 直接结束。
    // 否则，acquireQueued方法会将线程放入阻塞队列
    public final void acquire(int arg) { // 此时 arg == 1
        // 首先调用tryAcquire(1)，尝试获取锁
        if (!tryAcquire(arg) &&
            // tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
              selfInterrupt();
        }
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    // 尝试直接获取锁，返回值是boolean，代表是否获取到锁
    // 返回true：1.没有线程在等待锁；2.重入锁，线程本来就持有锁，可以直接获取
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state == 0 此时没有线程持有锁
        if (c == 0) {
            // 和非公平锁相比，这里多了一个判断hasQueuedPredecessors：是否有线程在等待
            if (!hasQueuedPredecessors() &&
                // 如果没有线程在等待，CAS尝试获取锁，成功了就获取到锁了
                // 不成功的话，就是刚刚几乎同一时刻有个线程抢先获取了锁
                compareAndSetState(0, acquires)) {

                // 获取到锁，标记获取到当前锁的线程为自己
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 此分支说明是重入锁
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 如果到这里，说明前面的if和else if都没有返回true，说明没有获取到锁
        return false;
    }

    // tryAcquire(arg) 返回false，将进入：
    //        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，

    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    // 此方法的作用是把线程包装成node，同时进入到队列中
    // 参数mode此时是Node.EXCLUSIVE，代表独占模式
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 先尝试将当前node直接加入到队列最后，若失败则执行enq(node)
        Node pred = tail;

        // tail!=null => 队列不为空
        if (pred != null) { 
            // 将当前的队尾节点，设置为自己的前驱 
            node.prev = pred; 
            // 用CAS把自己设置为队尾, 如果成功后，tail == node 了，这个节点成为阻塞队列新的尾巴
            if (compareAndSetTail(pred, node)) { 
                // 进到这里说明设置成功，当前node==tail, 将自己与之前的队尾相连，
                // 上面已经有 node.prev = pred，加上下面这句，也就实现了和之前的尾节点双向连接了
                pred.next = node;
                // 线程入队了，可以返回了
                return node;
            }
        }
        // pred==null(队列是空的) 或者 CAS失败(有线程在竞争入队)
        enq(node);
        return node;
    }

    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    // 采用自旋的方式入队
    // 之前说过，到这个方法只有两种可能：等待队列为空，或者有线程竞争入队，
    // 自旋在这边的语义是：CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 等待队列为空，需要初始化队列
            if (t == null) { // Must initialize
                // 初始化head节点
                if (compareAndSetHead(new Node()))
                    // 给后面用：这个时候head节点的waitStatus==0, 看new Node()构造方法就知道了

                    // 设置tail = head后，继续for循环，下次循环进入else分支
                    tail = head;
            } else {
                // 将当前线程排到队尾，若有线程竞争，循环尝试
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

    // 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // p == head 当前节点排在阻塞队列的第一个节点
                // 然后使用tryAcquire再次尝试获取锁 
                if (p == head && tryAcquire(arg)) {
                    //获取成功，将头结点设置为当前节点所在节点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 当前node不是队头节点或tryAcquire(arg)获取锁失败
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // tryAcquire() 方法抛异常，取消获取锁
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    // 当前线程没有抢到锁，是否需要挂起当前线程？
    // 第一个参数是前驱节点，第二个参数是代表当前线程的节点
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点的 waitStatus == -1，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;

        // 前驱节点 waitStatus大于0，大于0 说明前驱节点取消了排队。
        // 进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                // 将当前节点的prev指向waitStatus<=0的节点
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            // 前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3
            // 用CAS将前驱节点的waitStatus设置为Node.SIGNAL(也就是-1)
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 这个方法返回 false，那么会再走一次 for 循序，
        // 然后再次进来此方法，此时会从第一个分支返回 true
        return false;
    }

    // private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
    // 如果返回true, 说明前驱节点的waitStatus==-1，是正常情况，那么当前线程需要被挂起，等待以后被唤醒
    // 如果返回false, 说明当前不需要被挂起

    // if (shouldParkAfterFailedAcquire(p, node) &&
    //                parkAndCheckInterrupt())
    //                interrupted = true;

    // 如果shouldParkAfterFailedAcquire(p, node)返回true，
    // 执行parkAndCheckInterrupt():

    // 此负责挂起线程
    // 使用LockSupport.park(this)挂起线程，线程将会阻塞在此处，等待被唤醒
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

    // 如果shouldParkAfterFailedAcquire(p, node)返回false的情况
   // 第一次进入此方法，一般都不会返回true的，原因很简单，前驱节点的waitStatus=-1是依赖于后继节点设置的。也就是说，我都还没给前驱设置-1呢，怎么可能是true呢，但是要看到，这个方法是套在循环里的，所以第二次进来的时候状态就是-1了。
    // 为什么shouldParkAfterFailedAcquire(p, node)返回false的时候不直接挂起线程：
    // 是为了应对在经过这个方法后，node已经是head的直接后继节点了。
}
```

#### 解锁

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// ReentrantLock.tryRelease()
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否完全释放锁
    boolean free = false;
    // 可重入锁，如果c==0，可重入锁已释放
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
// 唤醒后继节点
// 参数node是head头结点
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    // 如果head节点当前waitStatus<0, 将其修改为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    // 唤醒后继节点，但是有可能后继节点取消了等待（waitStatus==1）
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后往前找waitStatus<=0的所有节点中排在最前面的
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒加锁过程中parkAndCheckInterrupt()中阻塞的线程
        LockSupport.unpark(s.thread);
}
```

#### 公平锁与非公平锁

公平锁：

```java
if (c == 0) {
    //公平锁相比非公平锁多了一个判断（!hasQueuedPredecessors()）：是否有线程在等待
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```

非公平锁：

```java
if (c == 0) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```

通过同步状态state控制可重入

1. State初始化的时候为0，表示没有任何线程持有锁。
2. 当有线程持有该锁时，值就会在原来的基础上+1，同一个线程多次获得锁是，就会多次+1，这里就是可重入的概念。
3. 解锁也是对这个字段-1，一直到0，此线程对锁释放。

### JUC中AQS相关应用

| 同步工具                   | 同步工具与AQS的关联                                          |
| :------------------------- | :----------------------------------------------------------- |
| ReentrantLock              | 使用AQS保存锁重复持有的次数。当一个线程获取锁时，ReentrantLock记录当前获得锁的线程标识，用于检测是否重复获取，以及错误线程试图解锁操作时异常情况的处理。 |
| Semaphore（信号量）        | 使用AQS同步状态来保存信号量的当前计数。tryRelease会增加计数，acquireShared会减少计数。 |
| CountDownLatch（倒计时器） | 使用AQS同步状态来表示计数。计数为0时，所有的Acquire操作（CountDownLatch的await方法）才可以通过。 |
| Cyclicbarrier（循环栅栏）  | 使用AQS的Condition，让一组线程到达一个屏障时被阻塞，知道最后一个线程到达屏障时，屏障才会打开，所有被阻塞的线程才可继续运行。 |
| ReentrantReadWriteLock     | 使用AQS同步状态中的16位保存写锁持有的次数，剩下的16位用于保存读锁的持有次数。 |
| ThreadPoolExecutor         | Worker利用AQS同步状态实现对独占线程变量的设置（tryAcquire和tryRelease）。 |

### 自定义同步工具

通过继承AbstractQueuedSynchronizer可以很方便的自定义同步工具

```java
public class LiuLock  {

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire (int arg) {
            return compareAndSetState(0, 1);
        }

        @Override
        protected boolean tryRelease (int arg) {
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively () {
            return getState() == 1;
        }
    }
    
    private Sync sync = new Sync();
    
    public void lock () {
        sync.acquire(1);
    }
    
    public void unlock () {
        sync.release(1);
    }
}
```

自定义同步工具使用

```java
public class TestMain {

    static int count = 0;
    static LiuLock liuLock = new LiuLock();

    public static void main (String[] args) throws InterruptedException {

        Runnable runnable = new Runnable() {
            @Override
            public void run () {
                try {
                    liuLock.lock();
                    for (int i = 0; i < 10000; i++) {
                        count++;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    liuLock.unlock();
                }

            }
        };
        Thread thread1 = new Thread(runnable);
        Thread thread2 = new Thread(runnable);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println(count);
    }
}
```

