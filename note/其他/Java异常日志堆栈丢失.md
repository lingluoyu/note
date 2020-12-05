### 原因

JVM内部同一个方法被调用多次的时候，会被JIT编译器进行优化。

> The compiler in the server VM now provides correct stack backtraces for all "cold" built-in exceptions. For performance purposes, when such an exception is thrown a few times, the method may be recompiled. After recompilation, the compiler may choose a faster tactic using preallocated exceptions that do not provide a stack trace. To disable completely the use of preallocated exceptions, use this new flag: -XX:-OmitStackTraceInFastThrow.

在Server模式下的JVM编译器提供了一种方式可以让我们回溯异常的堆栈信息，但是出于性能的因素，当类似的异常抛出多次的时候，异常方法可以被重新编译，重新编译后，编译器会采用更快的策略使用预分配缓存区的异常，并且不再提供堆栈信息。

如果不想使用预分配缓存的异常使用-XX:-OmitStackTraceInFastThrow标记。

### 异常丢失如何排查

- 试着隔离一两台机器，重启两台机器观察情况
- -XX:-OmitStackTraceInFastThrow可以控制不进行异常堆栈优化，**如果关闭，就需要预防产生“日志风暴”，否则，一旦高频应用出现异常可能很快用满服务器磁盘。**