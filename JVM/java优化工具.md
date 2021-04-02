### 基础故障处理工具
#### jps：虚拟机进程状况工具

jps（JVM Process Status Tool）可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）。

jps命令格式：

```bash
jps [ options ] [ hostid ]
```

jps执行样例：

```bash
jps -l -m -v
2893 sun.tools.jps.Jps -l -m -v -Denv.class.path=/usr/local/java/lib/ -Dapplication.home=/usr/local/java -Xms8m
```

jps还可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，参数hostid为RMI注册表中注册的主机名。

jps工具主要选项

| 选项 | 作用                                                 |
| :--: | ---------------------------------------------------- |
|  -q  | 只输出LVMID，省略主类的名称                          |
|  -m  | 输出虚拟机进程机动是传递给主类main()函数的参数       |
|  -l  | 输出主类的全名，如果进程执行的是JAR包，则输出JAR路径 |
|  -v  | 输出虚拟机进程启动时的JVM参数                        |

#### jstat：虚拟机统计信息监视工具

jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程[1]虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。

jstat命令格式：

```bash
jstat [ option vmid [interval[s|ms] [count]] ]
```

如果是本地虚拟机进程，VMID与LVMID是一致的；如果是远程虚拟机进程，那VMID的格式应当是：

```bash
[protocol:][//]lvmid[@hostname[:port]/servername]
```

参数interval和count代表查询间隔和次数，如果省略这2个参数，说明只查询一次。

选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况。

|       选项        | 作用                                                         |
| :---------------: | ------------------------------------------------------------ |
|      -class       | 监视类加载、卸载数量、总空间以及类装载所耗费的时间           |
|        -gc        | 监视Java堆状况，包括Eden区、两个Survivor区、老年代、永久代等的容量，已用空间，垃圾收集时间合计等信息 |
|    -gccapacity    | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间 |
|      -gcutil      | 监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比 |
|     -gccause      | 与 -gcutil功能一样，但是会额外输出导致上一次垃圾收集产生的原因 |
|      -gcnew       | 监视新生代垃圾收集状况                                       |
|  -gcnewcapacity   | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间 |
|      -gcold       | 监视老年代垃圾收集状况                                       |
|  -gcoldcapacity   | 监视内容与-gcold基本相同，输出主要关注使用到的最大、最小空间 |
|  -gcpermcapacity  | 输出永久代使用到的最大、最小空间                             |
|     -compiler     | 输出即时编译器编译过的方法、耗时等信息                       |
| -printcompilation | 输出已经被即时编译的方法                                     |

这些参数中最常用的参数是gcutil，下面是该参数的输出介绍以及一个简单例子（监视一台刚刚启动的
GlassFish v3服务器的内存状况）：

```bash
S0  — Heap上的 Survivor space 0 区已使用空间的百分比 
S1  — Heap上的 Survivor space 1 区已使用空间的百分比 
E   — Heap上的 Eden space 区已使用空间的百分比 
O   — Heap上的 Old space 区已使用空间的百分比 
P   — Perm space 区已使用空间的百分比 
YGC — 从应用程序启动到采样时发生 Young GC 的次数 
YGCT– 从应用程序启动到采样时 Young GC 所用的时间(单位秒) 
FGC — 从应用程序启动到采样时发生 Full GC 的次数 
FGCT– 从应用程序启动到采样时 Full GC 所用的时间(单位秒) 
GCT — 从应用程序启动到采样时用于垃圾回收的总时间(单位秒) 
   
实例使用1： 
   
jstat -gcutil 25444 
   
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT 
   
 0.00   0.00   6.20  41.42  47.20    16     0.105    3     0.472   0.577 
 
 #查询结果表明：这台服务器的新生代Eden区（E，表示Eden）使用了6.2%的空间，2个Survivor区（S0、S1，表示Survivor0、Survivor1）里面都是空的，老年代（O，表示Old）和永久代（P，表示Permanent）则分别使用了41.42%和47.20%的空间。程序运行以来共发生Minor GC（YGC，表示YoungGC）16次，总耗时0.105秒；发生Full GC（FGC，表示Full GC）3次，总耗时（FGCT，表示Full GCTime）为0.472秒；所有GC总耗时（GCT，表示GC Time）为0.577秒。
```

#### jinfo：Java配置信息工具

jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数。

使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，使用jinfo的-flag选项查询未被显式指定的参数的系统默认值。还可以使用-sysprops选项把虚拟机进程的System.getProperties()的内容打印出来。

JDK 6之后，加入了在运行期修改部分参数值的能力（可以使用-flag[+|-]name或者-flag name=value在运行期修改一部分运行期可写的虚拟机参数值）。

jinfo命令格式：

```bash
jinfo [ option ]  pid
```

jinfo可以查看运行时参数：

```bash
jinfo -flag MaxTenuringThreshold 31518
-XX:MaxTenuringThreshold=15
```

jinfo还可以在运行时修改参数值：

```bash
> jinfo -flag PrintGCDetails 31518
-XX:-PrintGCDetails
> jinfo -flag +PrintGCDetails 31518
> jinfo -flag PrintGCDetails 31518
-XX:+PrintGCDetails
```

#### jmap：Java内存映像工具

jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文件）。可以通过此工具打印出某个Java进程内存内的所有对象大小和数量。

jmap的作用并不仅仅是为了获取堆转储快照，它还可以查询finalize执行队列、Java堆和方法区的
详细信息，如空间使用率、当前用的是哪种收集器等。

jmap命令格式：

```bash
jmap [ option ] vmid
```

option选项如下：

| 选项           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| -dump          | 生成Java堆转出快照。格式为 `-dump:[live,]format=b,file=<filename>`，其中live子参数说明是否只dump出存活的对象 |
| -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在Linux/Solaris平台下有效 |
| -heap          | 显示Java堆详细信息，如使用哪种回收器、参数配置、分代状况等。只在Linux/Solaris平台下有效 |
| -histo         | 显示堆中对象统计信息，包括类、实例数量、合计容量             |
| -permstat      | 以ClassLoader为统计口径显示永久代内存状态。只在Linux/Solaris平台下有效 |
| -F             | 当虚拟机进程堆-dump选项没有响应时，可使用这个选项强制生成dump快照。只在Linux/Solaris平台下有效 |

使用jmap生成dump文件
```
> jmap -dump:format=b,file=heap.hprof 31531
Dumping heap to /Users/caojie/heap.hprof ...
Heap dump file created
```
获得堆快照文件之后，我们可以使用多种工具对文件进行分析，例如jhat，visual vm等。
#### jhat：虚拟机堆转储快照分析工具
JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat可以将堆中的对象以HTML的形式显示出来，包括对象的数量、大小等，默认端口7000。

使用jhat分析dump文件

```
> jhat heap.hprof
Reading from heap.hprof...
Dump file created Tue Nov 11 06:02:05 CST 2014
Snapshot read, resolving...
Resolving 8781 objects...
Chasing references, expect 1 dots.
Eliminating duplicate references.
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```
jhat在分析完成之后，使用HTTP服务器展示其分析结果，在浏览器中访问http://127.0.0.1:7000/即可得到分析结果。
#### jstack：Java堆栈跟踪工具
jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因。线程出现停顿时通过jstack来查看各个线程的调用堆栈，就可以获知没有响应的线程到底在后台做些什么事情，或者等待着什么资源。

jstack命令格式：

```
jstack [ option ] vmid
```

option选项如下：

| 选项 | 作用                                         |
| ---- | -------------------------------------------- |
| -F   | 当正常输出的请求不被响应时，强制输出线程堆栈 |
| -l   | 除堆栈外，显示关于锁的附加信息               |
| -m   | 如果调用到本地方法的话，可以显示C/C++的堆栈  |

使用jstack查看线程堆栈

```bash
jstack -l 3500
2010-11-19 23:11:26
Full thread dump Java HotSpot(TM) 64-Bit Server VM (17.1-b03 mixed mode):
"[ThreadPool Manager] - Idle Thread" daemon prio=6 tid=0x0000000039dd4000 nid= 0xf50 in Object.wait() [0x000000003c96f000]
    java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
        at java.lang.Object.wait(Object.java:485)
        at org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor.run (Executor. java:106)
        - locked <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
     Locked ownable synchronizers:
        - None
```

#### jstatd

jstatd命令是一个RMI服务器程序，它的作用相当于代理服务器，建立本地计算机与远程监控工具的通信，jstatd服务器能够将本机的Java应用程序信息传递到远程计算机。

#### hprof
hprof工具可以用于监控Java应用程序在运行时的CPU信息和堆信息。

### 可视化故障处理工具

#### JConsole：Java监视与管理控制台

JConsole（Java Monitoring and Management Console）是一款基于JMX（Java Manage-ment Extensions）的可视化监视、管理工具。它的主要功能是通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整。

#### VisualVM：多合-故障处理工具

[Oracle官方文档https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html](https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html)

Visual VM是一个功能强大的多合一故障诊断和性能监控的可视化工具，它集成了多种性能统计工具的功能，使用Visual VM可以替代jstat、jmap、jhat、jstack等工具。在命令行输入jvisualvm即可启动visualvm。

#### MAT内存分析工具
MAT是一款功能强大的Java堆内存分析器，可以用于查找内存泄露以及查看内存消耗情况。

[MAT官方文档http://help.eclipse.org/luna/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html](http://help.eclipse.org/luna/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)

MAT中有浅堆和深堆的概念，浅堆是指一个对象结构所占用的内存大小，深堆是指一个对象被GC回收后可以真正释放的内存大小。

通过MAT，可以列出所有垃圾回收的根对象，Java系统的根对象可能是以下类：系统类，线程，Java局部变量，本地栈等等。在MAT中还可以很清楚的看到根对象到当前对象的引用关系链。

MAT还可以自动检测内存泄露，单击菜单上的Leak Suspects命令，MAT会自动生成一份报告，这份报告罗列了系统内可能存在内存泄露的问题点。

在MAT中，还可以自动查找并显示消耗内存最多的几个对象，这些消耗大量内存的大对象往往是解决系统性能问题的关键所在。