#### ThreadLocal

ThreadLocal提供了get与set等访问接口或方法，这些方法为每个使用该变量的线程都存有一份独立的副本，因此get总是返回由当前执行线程在调用set时设置的最新值。

ThreadLocal对象通常用于防止对可变的单实例变量或全局变量进行共享。

#### ThreadLocal示例

SimpleDateFormat为线程不安全类，通过ThreadLocal为每个线程提供专属的SimpleDateFormat

```java
public class ThreadLocalExample implements Runnable {

    //java8写法
    //private static final ThreadLocal<SimpleDateFormat> formatter = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyyMMdd HHmm"));

    //java8之前
    private static final ThreadLocal<SimpleDateFormat> formatter = new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyyMMdd HHmm");
        }
    };
    
    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(obj, "" + i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        System.out.println("Thread Name=" + Thread.currentThread().getName() + "defaultFormatter =" + formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //formatter pattern is changed here by thread, but it won't reflect to other threads
        formatter.set(new SimpleDateFormat());
        System.out.println("Thread Name=" + Thread.currentThread().getName() + " formatter =" + formatter.get().toPattern());
    }
}
```

输出结果：

```shell
Thread Name=0 defaultFormatter =yyyyMMdd HHmm
Thread Name=1 defaultFormatter =yyyyMMdd HHmm
Thread Name=1 formatter =yy-M-d ah:mm
Thread Name=0 formatter =yy-M-d ah:mm
Thread Name=2 defaultFormatter =yyyyMMdd HHmm
Thread Name=3 defaultFormatter =yyyyMMdd HHmm
Thread Name=2 formatter =yy-M-d ah:mm
Thread Name=4 defaultFormatter =yyyyMMdd HHmm
Thread Name=3 formatter =yy-M-d ah:mm
Thread Name=4 formatter =yy-M-d ah:mm
Thread Name=5 defaultFormatter =yyyyMMdd HHmm
Thread Name=6 defaultFormatter =yyyyMMdd HHmm
Thread Name=7 defaultFormatter =yyyyMMdd HHmm
Thread Name=5 formatter =yy-M-d ah:mm
Thread Name=6 formatter =yy-M-d ah:mm
Thread Name=8 defaultFormatter =yyyyMMdd HHmm
Thread Name=7 formatter =yy-M-d ah:mm
Thread Name=9 defaultFormatter =yyyyMMdd HHmm
Thread Name=9 formatter =yy-M-d ah:mm
Thread Name=8 formatter =yy-M-d ah:mm
```

可以看出每个线程都拥有专属的formatter，Thread0修改formatter不会影响其他线程的formatter

#### ThreadLocal原理

Thread类

```java
public class Thread implements Runnable {
    ......
    //与此线程有关的ThreadLocal值，由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;

    //与此线程有关的inheritableThreadLocals值，由InheritableThreadLocals类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    ......
}
```

Thread 类中有一个 threadLocals 变量和一个 inheritableThreadLocals 变量，变量类型均为 ThreadLocalMap。

ThreadLocalMap 是 ThreadLocal 的静态内部类。

threadLocals 和 inheritableThreadLocals 默认为 null，当我们调用 ThreadLocal 的 set 方法时，会通过 ThreadLocalMap 的 getMap 方法创建它们

```java
//ThreadLocal的set方法
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

//ThreadLocalMap的getMap方法
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

**最终变量放在当前线程的ThreadLocalMap中，ThreadLocal只是对ThreadLocalMap的封装，传递变量到ThreadLocalMap**

ThreadLocal 内部维护了一个类似于 Map 的 ThreadLocalMap，key 为当前对象的 ThreadLocal 对象，value 为 Object 对象（ThreadLocal 对象调用 set 方法设置的值）

```java
public class ThreadLocal<T> {
    ......
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ......
    }
    ......
}
```

#### ThreadLocal内存泄露

ThreadLocalMap 中的 Entry 使用的 key 为 ThreadLocal 的弱引用，而value使用的是强引用。所以，ThreadLocal没有被外部强引用的时候，垃圾回收时，key 会被清理掉，而 value 不会被清理掉。此时 ThreadLocalMap 中就会出现 key 为 null 的 Entry。若不采取措施，value 将永远不会被 GC 回收，就可能会产生内存泄露。ThreadLocalMap 已经考虑到了此种情况，在调用 set() 、 get() 、 remove() 方法时，会清理掉 key 为 null 的记录。

**在使用 ThreadLocal 后，手动调用 remove() 方法**

