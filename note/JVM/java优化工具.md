## JDK命令行工具
### jmap
jmap是JDK自带的堆分析工具Java Memory Map，可以通过此工具打印出某个Java进程内存内的所有对象大小和数量；建议在测试环境中使用jmap -histo:live命令查询，执行此命令会触发一次Full GC
```
> jmap -dump:format=b,file=heap.hprof 31531
Dumping heap to /Users/caojie/heap.hprof ...
Heap dump file created
```
获得堆快照文件之后，我们可以使用多种工具对文件进行分析，例如jhat，visual vm等。
### jhat
jhat是JDK自带的堆分析工具Java Heap Analyse Tool，可以将堆中的对象以HTML的形式显示出来，包括对象的数量、大小等，默认端口7000。

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
### jps
jps用于列出Java的进程，jps可以增加参数，-m用于输出传递给Java进程的参数，-l用于输出主函数的完整路径，-v可以用于显示传递给jvm的参数。

```
jps -l -m -v
2893 sun.tools.jps.Jps -l -m -v -Denv.class.path=/usr/local/java/lib/ -Dapplication.home=/usr/local/java -Xms8m
```
### jstat
jstat是一个可以用于观察Java应用程序运行时信息的工具，它的功能非常强大，可以通过它查看堆信息的详细情况，它的基本使用方法为：

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```
选项option可以由以下值组成：

```
jstat -class pid:显示加载class的数量，及所占空间等信息。 
jstat -compiler pid:显示VM实时编译的数量等信息。 
jstat -gc pid:可以显示gc的信息，查看gc的次数，及时间。其中最后五项，分别是young gc的次数，young gc的时间，full gc的次数，full gc的时间，gc的总时间。 
jstat -gccapacity:可以显示，VM内存中三代（young,old,perm）对象的使用和占用大小，如：PGCMN显示的是最小perm的内存使用量，PGCMX显示的是perm的内存最大使用量，PGC是当前新生成的perm内存占用量，PC是但前perm内存占用量。其他的可以根据这个类推， OC是old内纯的占用量。 
jstat -gcnew pid:new对象的信息。 
jstat -gcnewcapacity pid:new对象的信息及其占用量。 
jstat -gcold pid:old对象的信息。 
jstat -gcoldcapacity pid:old对象的信息及其占用量。 
jstat -gcpermcapacity pid: perm对象的信息及其占用量。 
jstat -gcutil pid:统计gc信息统计。 
jstat -printcompilation pid:当前VM执行的信息。 
除了以上一个参数外，还可以同时加上 两个数字，如：jstat -printcompilation 3024 250 6是每250毫秒打印一次，一共打印6次。
```
这些参数中最常用的参数是gcutil，下面是该参数的输出介绍以及一个简单例子：

```
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
   
[root@localhost bin]# jstat -gcutil 25444 
   
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT 
   
 11.63   0.00   56.46  66.92  98.49 162    0.248    6      0.331    0.579 
```
### jinfo
jinfo可以用来查看正在运行的Java应用程序的扩展参数，甚至在运行时修改部分参数，它的基本语法为：

```
jinfo  <option>  <pid>
```
jinfo可以查看运行时参数：

```
jinfo -flag MaxTenuringThreshold 31518
-XX:MaxTenuringThreshold=15
```
jinfo还可以在运行时修改参数值：

```
> jinfo -flag PrintGCDetails 31518
-XX:-PrintGCDetails
> jinfo -flag +PrintGCDetails 31518
> jinfo -flag PrintGCDetails 31518
-XX:+PrintGCDetails
```
### jstack
jstack可用于导出Java应用程序的线程堆栈信息，语法为：

```
jstack -l <pid>
```

### jstatd
jstatd命令是一个RMI服务器程序，它的作用相当于代理服务器，建立本地计算机与远程监控工具的通信，jstatd服务器能够将本机的Java应用程序信息传递到远程计算机。

### hprof
hprof工具可以用于监控Java应用程序在运行时的CPU信息和堆信息。

## Visual VM工具
[Oracle官方文档https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html](https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html)

Visual VM是一个功能强大的多合一故障诊断和性能监控的可视化工具，它集成了多种性能统计工具的功能，使用Visual VM可以替代jstat、jmap、jhat、jstack等工具。在命令行输入jvisualvm即可启动visualvm。

## MAT内存分析工具
MAT是一款功能强大的Java堆内存分析器，可以用于查找内存泄露以及查看内存消耗情况。

[MAT官方文档http://help.eclipse.org/luna/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html](http://help.eclipse.org/luna/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)

MAT中有浅堆和深堆的概念，浅堆是指一个对象结构所占用的内存大小，深堆是指一个对象被GC回收后可以真正释放的内存大小。

通过MAT，可以列出所有垃圾回收的根对象，Java系统的根对象可能是以下类：系统类，线程，Java局部变量，本地栈等等。在MAT中还可以很清楚的看到根对象到当前对象的引用关系链。

MAT还可以自动检测内存泄露，单击菜单上的Leak Suspects命令，MAT会自动生成一份报告，这份报告罗列了系统内可能存在内存泄露的问题点。

在MAT中，还可以自动查找并显示消耗内存最多的几个对象，这些消耗大量内存的大对象往往是解决系统性能问题的关键所在。