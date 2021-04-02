### BIO

阻塞IO实现Socket服务器和客户端

Server:

```java
public class Server {
    private static ExecutorService executorService = Executors.newFixedThreadPool(10);

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8888);
        Socket accept = null;
        while (true) {
            accept = serverSocket.accept();
            executorService.submit(new serverHandler(accept));
        }
    }

    private static class serverHandler implements Runnable {
        Socket socket = null;

        public serverHandler(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            String buf = null;
            BufferedReader br = null;
            PrintWriter pw = null;

            try {
                br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                pw = new PrintWriter(new OutputStreamWriter(socket.getOutputStream()));

                System.out.println("接收到请求，来自: " + socket.getInetAddress().toString() + ":" + socket.getPort());
                System.out.println("服务端就绪");

                //read
                while (true) {
                    buf = br.readLine();
                    System.out.println(Thread.currentThread().getName() + " 客户端信息: " + buf);

                    if (buf.equals("close")) {
                        break;
                    }

                    //write
                    System.out.println(Thread.currentThread().getName() + " 服务端返回响应信息: 响应" + buf);
                    pw.println("响应信息" + buf);
                    pw.flush();
                }

                //close
                System.out.println("客户端关闭！");
                br.close();
                pw.close();
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

Client:

```java
public class Client {
    public static void main(String[] args) throws IOException {
        Socket clientSocket = null;
        BufferedReader br = null;
        PrintWriter pw = null;
        Scanner sc = new Scanner(System.in);
        String buf = null;

        try {

            //connect
            clientSocket = new Socket("localhost", 8888);
            br = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            pw = new PrintWriter(new OutputStreamWriter(clientSocket.getOutputStream()));
            System.out.println("===============客户端就绪================");

            while (true) {
                System.out.print("请输入客户端发送的信息: ");
                buf = sc.nextLine();
                pw.println(buf);
                pw.flush();
                if (buf.equals("close")) {
                    break;
                }

                buf = br.readLine();
                System.out.println("服务端响应信息: " + buf);

                if (buf.equals("close")) {
                    break;
                }
            }

            br.close();
            pw.close();
            sc.close();
            clientSocket.close();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### NIO

#### Buffer

buffer实际上是一个容器，一个连续数组，它通过几个变量来保存这个数据的当前位置状态：
1.capacity：容量，缓冲区能容纳元素的数量
2.position：当前位置，是缓冲区中下一次发生读取和写入操作的索引，当前位置通过大多数读写操作向前推进
3.limit：界限，是缓冲区中最后一个有效位置之后下一个位置的索引

```java
.flip()        //将limit设置为position，然后position重置为0，返回对缓冲区的引用
.clear()        //清空调用缓冲区并返回对缓冲区的引用
```

##### 使用Buffer

```java
//给Buffer分配空间，以字节为单位
//创建一个ByteBuffer对象并且指定内存大小
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

向Buffer中写入数据
//数据从Channel到Buffer
channel.read(byteBuffer);
//数据从Client到Buffer
byteBuffer.put(...);

//从Buffer中读取数据
//数据从Buffer到Channel
channel.write(byteBuffer);
//数据从Buffer到Server
byteBuffer.get(...);
```

#### Channel

channel一共有四种：

FileChannel：作用于IO文件流
DatagramChannel：作用于UDP协议
SocketChannel：作用于TCP协议
ServerSocketChannel：作用于TCP协议

##### 使用Channel

```java
//打开一个ServerSocketChannel通道
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//关闭ServerSocketChannel通道
serverSocketChannel.close();
//循环监听SocketChanne
while(true){
    SocketChannel socketChannel = serverSocketChannel.accept();
    //将此通道设置为非阻塞
    clientChannel.configureBlocking(false);
}
```

`clientChannel.configureBlocking(false);`语句是将此通道设置为非阻塞，也就是异步自由控制阻塞或非阻塞便是NIO的特性之一

#### Selector

选择器是NIO的核心，它是channel的管理者
通过执行select()阻塞方法，监听是否有channel准备好
一旦有数据可读，此方法的返回值是SelectionKey的数量

所以服务端通常会死循环执行select()方法，直到有channl准备就绪，然后开始工作
每个channel都会和Selector绑定一个事件，然后生成一个SelectionKey的对象
需要注意的是：
channel和Selector绑定时，channel必须是非阻塞模式
而FileChannel不能切换到非阻塞模式，因为它不是套接字通道，所以FileChannel不能和Selector绑定事件

在NIO中一共有四种事件：
1.SelectionKey.OP_CONNECT：连接事件
2.SelectionKey.OP_ACCEPT：接收事件
3.SelectionKey.OP_READ：读事件
4.SelectionKey.OP_WRITE：写事件

##### SelectionKey

SelectionKey是通道和选择器交互的核心组件

```java
//在SocketChannel上绑定一个Selector，并注册为连接事件
SocketChannel clientChannel = SocketChannel.open();
clientChannel.configureBlocking(false);
clientChannel.connect(new InetSocketAddress(port));
clientChannel.register(selector, SelectionKey.OP_CONNECT);

```

核心在register()方法，它返回一个SelectionKey对象
来检测channel事件是那哪种事件可以使用以下方法：

```java
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

服务端便是通过这些方法 在轮询中执行相对应操作

当然通过Channel与Selector绑定的key也可以反过来拿到他们

```java
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector();
```

在Channel上注册事件时，我们也可以顺带绑定一个Buffer：

```java
clientChannel.register(key.selector(), SelectionKey.OP_READ,ByteBuffer.allocateDirect(1024));
```

或者绑定一个Object：

```java
selectionKey.attach(Object);
Object anthorObj = selectionKey.attachment();
```

NIO实现Socket服务器和客户端

NioServer:

```java
public class NioServer {
    private Selector selector;          //创建一个选择器
    private final static int port = 8686;
    private final static int BUF_SIZE = 10240;

    private void initServer() throws IOException {
        //创建通道管理器对象selector
        this.selector= Selector.open();

        //创建一个通道对象channel
        ServerSocketChannel channel = ServerSocketChannel.open();
        channel.configureBlocking(false);       //将通道设置为非阻塞
        channel.socket().bind(new InetSocketAddress(port));       //将通道绑定在8686端口

        //将上述的通道管理器和通道绑定，并为该通道注册OP_ACCEPT事件
        //注册事件后，当该事件到达时，selector.select()会返回（一个key），如果该事件没到达selector.select()会一直阻塞
        channel.register(selector,SelectionKey.OP_ACCEPT);

        while (true){       //轮询
            selector.select();          //这是一个阻塞方法，一直等待直到有数据可读，返回值是key的数量（可以有多个）
            Set keys = selector.selectedKeys();         //如果channel有数据了，将生成的key访入keys集合中
            Iterator iterator = keys.iterator();        //得到这个keys集合的迭代器
            while (iterator.hasNext()){             //使用迭代器遍历集合
                SelectionKey key = (SelectionKey) iterator.next();       //得到集合中的一个key实例
                iterator.remove();          //拿到当前key实例之后记得在迭代器中将这个元素删除，非常重要，否则会出错
                if (key.isAcceptable()){         //判断当前key所代表的channel是否在Acceptable状态，如果是就进行接收
                    doAccept(key);
                }else if (key.isReadable()){
                    doRead(key);
                }else if (key.isWritable() && key.isValid()){
                    doWrite(key);
                }else if (key.isConnectable()){
                    System.out.println("连接成功！");
                }
            }
        }
    }

    public void doAccept(SelectionKey key) throws IOException {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        System.out.println("ServerSocketChannel正在循环监听");
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        clientChannel.register(key.selector(),SelectionKey.OP_READ);
    }

    public void doRead(SelectionKey key) throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(BUF_SIZE);
        long bytesRead = clientChannel.read(byteBuffer);
        while (bytesRead>0){
            byteBuffer.flip();
            byte[] data = byteBuffer.array();
            String info = new String(data).trim();
            System.out.println("从客户端发送过来的消息是："+info);
            byteBuffer.clear();
            bytesRead = clientChannel.read(byteBuffer);
        }
        if (bytesRead==-1){
            clientChannel.close();
        }
    }

    public void doWrite(SelectionKey key) throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.allocate(BUF_SIZE);
        byteBuffer.flip();
        SocketChannel clientChannel = (SocketChannel) key.channel();
        while (byteBuffer.hasRemaining()){
            clientChannel.write(byteBuffer);
        }
        byteBuffer.compact();
    }

    public static void main(String[] args) throws IOException {
        NioServer server = new NioServer();
        server.initServer();
    }
}
```

NioClient:

```java
public class NioClient {
    private Selector selector;
    private final static int port = 8686;
    private final static int BUF_SIZE = 10240;
    private static ByteBuffer byteBuffer = ByteBuffer.allocate(BUF_SIZE);

    private void initClient() throws IOException {
        this.selector = Selector.open();
        SocketChannel clientChannel = SocketChannel.open();
        clientChannel.configureBlocking(false);
        clientChannel.connect(new InetSocketAddress(port));
        clientChannel.register(selector, SelectionKey.OP_CONNECT);
        while (true) {
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                if (key.isConnectable()) {
                    doConnect(key);
                } else if (key.isReadable()) {
                    doRead(key);
                }
            }
        }
    }

    public void doConnect(SelectionKey key) throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        if (clientChannel.isConnectionPending()) {
            clientChannel.finishConnect();
        }
        clientChannel.configureBlocking(false);
        String info = "服务端你好!!";
        byteBuffer.clear();
        byteBuffer.put(info.getBytes("UTF-8"));
        byteBuffer.flip();
        clientChannel.write(byteBuffer);
//      clientChannel.register(key.selector(),SelectionKey.OP_READ);
        clientChannel.close();
    }

    public void doRead(SelectionKey key) throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        clientChannel.read(byteBuffer);
        byte[] data = byteBuffer.array();
        String msg = new String(data).trim();
        System.out.println("服务端发送消息：" + msg);
        clientChannel.close();
        key.selector().close();
    }

    public static void main(String[] args) throws IOException {
        NioClient client = new NioClient();
        client.initClient();
    }
}
```

