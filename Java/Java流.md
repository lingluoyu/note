### 字节流和字符流

**字节（Byte）和字符（Character）**

- **1 byte = 8 bit**
- **1 char = 2 byte = 16 bit** (Java默认UTF-16编码)

**字节流**：以 8 位（即 1 byte，8 bit）作为一个数据单元，数据流中最小的数据单元是字节。

**字符流**：以 16 位（即 1 char，2 byte，16 bit）作为一个数据单元，数据流中最小的数据单元是字符， Java 中的字符是 Unicode 编码，一个字符占用两个字节。

InputStream是所有字节输入流的祖先，而OutputStream是所有字节输出流的祖先。 

Reader是所有读取字符串输入流的祖先，而writer是所有输出字符串的祖先。

InputStream，OutputStream，Reader，writer都是抽象类，所以不能直接new 。

字节流的InputStream和OutputStream是一切的基础，需要对字节流进行特殊解码才能得到字符流。Java中负责将字节流转为字符流的工具是：

> **InputStreamReader**
> **OutputStreamWriter**

字节流和字符流的区别？

- 读写单位不同：字节流以字节（8 bit）为单位，字符流以字符为单位，根据码表映射字符，一次可能读多个字节。
- 处理对象不同：字节流能处理所有类型的数据（如图片、avi 等），而字符流只能处理字符类型的数据。
- **字节流没有缓冲区**，是直接输出的，而**字符流是输出到缓冲区**的。因此在输出时，字节流不调用 colse() 方法时，信息已经输出了，而字符流只有在调用 close() 方法关闭缓冲区时，信息才输出。要想字符流在未关闭时输出信息，则需要手动调用 flush() 方法。

使用字节流好还是字符流好？
使用字节流更好。所有的文件在硬盘或在传输时都是以字节的方式进行的，包括图片等都是按字节的方式存储的，而字符是只有在内存中才会形成，所以在开发中，字节流使用较为广泛。

字节流是最基本的，所有的InputStream和OutputStream的子类都是,主要用在处理二进制数据，它是按字节来处理的 但实际中很多的数据是文本，又提出了字符流的概念，它是按虚拟机的encode来处理，也就是要进行字符集的转化。

这两个之间通过 InputStreamReader,OutputStreamWriter来关联，实际上是通过byte[]和String来关联，在实际开发中出现的汉字问题实际上都是在字符流和字节流之间转化不统一而造成的，在从字节流转化为字符流时，实际上就是byte[]转化为String时， public String(byte bytes[], String charsetName) 有一个关键的参数字符集编码，通常我们都省略了，那系统就用操作系统的lang 而在字符流转化为字节流时，实际上是String转化为byte[]时， byte[]    String.getBytes(String charsetName) 也是一样的道理，至于java.io中还出现了许多其他的流，按主要是为了提高性能和使用方便， 如BufferedInputStream，PipedInputStream等

### 节点流和处理流

根据是否直接处理数据，Java io可分为节点流和处理流，节点流是真正直接处理数据的；处理流是装饰加工节点流的。

**节点流**

- 文件流：FileInputStream，FileOutputStrean，FileReader，FileWriter，它们都会直接操作文件，直接与 OS 底层交互。因此他们被称为节点流 ，注意：使用这几个流的对象之后，需要关闭流对象，因为 java 垃圾回收器不会主动回收。不过在 Java7 之后，可以在 try() 括号中打开流，最后程序会自动关闭流对象，不再需要显示地 close。
- 数组流：ByteArrayInputStream，ByteArrayOutputStream，CharArrayReader，CharArrayWriter，对数组进行处理的节点流。
- 字符串流：StringReader，StringWriter，其中 StringReader 能从 String 中读取数据并保存到 char 数组。
- 管道流：PipedInputStream，PipedOutputStream，PipedReader，PipedWrite，对管道进行处理的节点流。

**处理流**

处理流是对一个已存在的流的连接和封装，通过所封装的流的功能调用实现数据读写。如 BufferedReader。

**注意**：处理流的构造方法总是要带一个其他的流对象做参数。

常用处理流（通过关闭处理流里面的节点流来关闭处理流）

- 缓冲流 ：BufferedImputStrean，BufferedOutputStream，BufferedReader ，BufferedWriter，需要父类作为参数构造，增加缓冲功能，避免频繁读写硬盘，可以初始化缓冲数据的大小，由于带了缓冲功能，所以就写数据的时候需要使用 flush 方法，另外，BufferedReader 提供一个 readLine( ) 方法可以读取一行，而 FileInputStream 和 FileReader 只能读取一个字节或者一个字符，因此 BufferedReader 也被称为行读取器。
- 转换流：InputStreamReader，OutputStreamWriter，要 inputStream 或 OutputStream 作为参数，实现从字节流到字符流的转换，我们经常在读取键盘输入（System.in）或网络通信的时候，需要使用这两个类。
- 数据流：DataInputStream，DataOutputStream，提供将基础数据类型写入到文件中，或者读取出来。