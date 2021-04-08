### BIO（传统Socket）
1. read：从socket读取数据。
2. decode：解码，因为网络上的数据都是以byte的形式进行传输的，要想获取真正的请求，必定需要解码。
3. compute：计算，也就是业务处理，你想干啥就干啥。
4. encode：编码，同理，因为网络上的数据都是以byte的形式进行传输的，也就是socket只接收byte，所以必定需要编码。


```
public static void main(String[] args) {
    try {
        ServerSocket serverSocket = new ServerSocket(9696);
        Socket socket = serverSocket.accept();
        new Thread(() -> {
            try {
                byte[] byteRead = new byte[1024];
                socket.getInputStream().read(byteRead);

                String req = new String(byteRead, StandardCharsets.UTF_8);//encode
                // do something

                byte[] byteWrite = "Hello".getBytes(StandardCharsets.UTF_8);//decode
                socket.getOutputStream().write(byteWrite);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
弊端：需要开启大量线程

### NIO
1. 一个线程可以监听多个Socket，不再是一夫当关，万夫莫开；
2. 基于事件驱动：等发生了各种事件，系统可以通知我，我再去处理。

### Reactor

```
public class Client {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("localhost", 9090));
            new Thread(() -> {
                while (true) {
                    try {
                        InputStream inputStream = socket.getInputStream();
                        byte[] bytes = new byte[1024];
                        inputStream.read(bytes);
                        System.out.println(new String(bytes, StandardCharsets.UTF_8));
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();

            while (true) {
                Scanner scanner = new Scanner(System.in);
                while (scanner.hasNextLine()) {
                    String s = scanner.nextLine();
                    socket.getOutputStream().write(s.getBytes());
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
![image](https://user-gold-cdn.xitu.io/2020/3/25/17110c9504654736?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
这是最简单的Reactor模型，可以看到有多个客户端连接到Reactor，Reactor内部有一个dispatch（分发器）。

有连接请求后，Reactor会通过dispatch把请求交给Acceptor进行处理，有IO读写事件之后，又会通过dispatch交给具体的Handler进行处理。

此时一个Reactor既然负责处理连接请求，又要负责处理读写请求，一般来说处理连接请求是很快的，但是处理具体的读写请求就要涉及到业务逻辑处理了，相对慢太多了。Reactor正在处理读写请求的时候，其他请求只能等着，只有等处理完了，才可以处理下一个请求。

### 单Reactor多线程模型
![image](https://user-gold-cdn.xitu.io/2020/3/25/17110c95048a2009?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
Reactor还是既要负责处理连接事件，又要负责处理客户端的写事件，不同的是，多了一个线程池的概念。

当客户端发起连接请求后，Reactor会把任务交给acceptor处理，如果客户端发起了写请求，Reactor会把任务交给线程池进行处理，这样一个服务端就可以同时为N个客户端服务了。

缺点：一个Reactor还是既然负责连接请求，又要负责读写请求，连接请求是很快的，而且一个客户端一般只要连接一次就可以了，但是会发生很多次写请求，如果可以有多个Reactor，其中一个Reactor负责处理连接事件，多个Reactor负责处理客户端的写事件就好了，这样更符合单一职责，所以主从Reactor模型诞生了。

### 主从Reactor模型
![image](https://user-gold-cdn.xitu.io/2020/3/25/17110c950aa2bb48?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
mainReactor只负责连接请求，而subReactor 只负责处理客户端的写事件。

### Reactor模型结构图
![image](https://user-gold-cdn.xitu.io/2020/3/25/17110c950e4a66db?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
* Synchronous Event Demultiplexer：同步事件分离器，用于监听各种事件，调用方调用监听方法的时候会被阻塞，直到有事件发生，才会返回。对于Linux来说，同步事件分离器指的就是IO多路复用模型，比如epoll，poll 等， 对于Java NIO来说， 同步事件分离器对应的组件就是selector，对应的阻塞方法就是select。
* Handler：本质上是文件描述符，是一个抽象的概念，可以简单的理解为一个一个事件，该事件可以来自于外部，比如客户端连接事件，客户端的写事件等等，也可以是内部的事件，比如操作系统产生的定时器事件等等。
* Event Handler：事件处理器，本质上是回调方法，当有事件发生后，框架会根据Handler调用对应的回调方法，在大多数情况下，是虚函数，需要用户自己实现接口，实现具体的方法。
* Concrete Event Handler： 具体的事件处理器，是Event Handler的具体实现。
* Initiation Dispatcher：初始分发器，实际上就是Reactor角色，提供了一系列方法，对Event Handler进行注册和移除；还会调用Synchronous Event Demultiplexer监听各种事件；当有事件发生后，还要调用对应的Event Handler。