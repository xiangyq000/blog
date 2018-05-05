---
title: 《深入分析 Java Web 技术内幕》：Java I/O
date: 2017-04-23 16:55:34
tags: 
categories: [读书笔记, Java]
---

## 1. 建立通信链路
当客户端要与服务端通信，客户端首先要创建一个 Socket 实例，操作系统将为这个 Socket 实例分配一个没有被使用的本地端口号，并创建一个包含本地和远程地址和端口号的套接字数据结构，这个数据结构将一直保存在系统中直到这个连接关闭。在创建 Socket 实例的构造函数正确返回之前，将要进行 TCP 的三次握手协议，TCP 握手协议完成后，Socket 实例对象将创建完成，否则将抛出 `IOException` 错误。
<!--more-->

与之对应的服务端将创建一个 `ServerSocket` 实例，`ServerSocket` 创建比较简单只要指定的端口号没有被占用，一般实例创建都会成功，同时操作系统也会为 `ServerSocket` 实例创建一个底层数据结构，这个数据结构中包含指定监听的端口号和包含监听地址的通配符，通常情况下都是“*”即监听所有地址。之后当调用 `accept()` 方法时，将进入阻塞状态，等待客户端的请求。当一个新的请求到来时，将为这个连接创建一个新的套接字数据结构，该套接字数据的信息包含的地址和端口信息正是请求源地址和端口。这个新创建的数据结构将会关联到 `ServerSocket` 实例的一个未完成的连接数据结构列表中，注意这时服务端与之对应的 `Socket` 实例并没有完成创建，而要等到与客户端的三次握手完成后，这个服务端的 `Socket` 实例才会返回，并将这个 `Socket` 实例对应的数据结构从未完成列表中移到已完成列表中。所以 `ServerSocket` 所关联的列表中每个数据结构，都代表与一个客户端的建立的 TCP 连接。

## 2. 数据传输
当连接已经建立成功，服务端和客户端都会拥有一个 `Socket` 实例，每个 `Socket` 实例都有一个 `InputStream` 和 `OutputStream`，正是通过这两个对象来交换数据。同时我们也知道网络 I/O 都是以字节流传输的。当 `Socket` 对象创建时，操作系统将会为 `InputStream` 和 `OutputStream` 分别分配一定大小的缓冲区，数据的写入和读取都是通过这个缓存区完成的。写入端将数据写到 `OutputStream` 对应的 SendQ 队列中，当队列填满时，数据将被发送到另一端 `InputStream` 的 RecvQ 队列中，如果这时 RecvQ 已经满了，那么 `OutputStream` 的 `write` 方法将会阻塞直到 RecvQ 队列有足够的空间容纳 SendQ 发送的数据。值得特别注意的是，这个缓存区的大小以及写入端的速度和读取端的速度非常影响这个连接的数据传输效率。

## 3. NIO
典型的 NIO 代码：

```Java
public void selector() throws IOException {  
        ByteBuffer buffer = ByteBuffer.allocate(1024);  
        Selector selector = Selector.open();  
        ServerSocketChannel ssc = ServerSocketChannel.open();  
        ssc.configureBlocking(false);//设置为非阻塞方式  
        ssc.socket().bind(new InetSocketAddress(8080));  
        ssc.register(selector, SelectionKey.OP_ACCEPT);//注册监听的事件  
        while (true) {  
            Set selectedKeys = selector.selectedKeys();//取得所有key集合  
            Iterator it = selectedKeys.iterator();  
            while (it.hasNext()) {  
                SelectionKey key = (SelectionKey) it.next();  
                if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {  
                    ServerSocketChannel ssChannel = (ServerSocketChannel) key.channel();  
                 SocketChannel sc = ssChannel.accept();//接受到服务端的请求  
                    sc.configureBlocking(false);  
                    sc.register(selector, SelectionKey.OP_READ);  
                    it.remove();  
                } else if   
                ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {  
                    SocketChannel sc = (SocketChannel) key.channel();  
                    while (true) {  
                        buffer.clear();  
                        int n = sc.read(buffer);//读取数据  
                        if (n <= 0) {  
                            break;  
                        }  
                        buffer.flip();  
                    }  
                    it.remove();  
                }  
            }  
        }  
} 
```

调用 `Selector` 的静态工厂创建一个选择器，创建一个服务端的 `Channel` 绑定到一个 `Socket` 对象，并把这个通信信道注册到选择器上，把这个通信信道设置为非阻塞模式。然后就可以调用 `Selector` 的 `selectedKeys` 方法来检查已经注册在这个选择器上的所有通信信道是否有需要的事件发生，如果有某个事件发生时，将会返回所有的 `SelectionKey`，通过这个对象 `Channel` 方法就可以取得这个通信信道对象从而可以读取通信的数据，而这里读取的数据是 `Buffer`，这个 `Buffer` 是我们可以控制的缓冲器。

在上面的这段程序中，是将 Server 端的监听连接请求的事件和处理请求的事件放在一个线程中，但是在实际应用中，我们通常会把它们放在两个线程中，一个线程专门负责监听客户端的连接请求，而且是阻塞方式执行的；另外一个线程专门来处理请求，这个专门处理请求的线程才会真正采用 NIO 的方式，像 Web 服务器 Tomcat 和 Jetty 都是这个处理方式。

通过 `Channel` 获取的 I/O 数据首先要经过操作系统的 `Socket` 缓冲区再将数据复制到 `Buffer` 中，这个的操作系统缓冲区就是底层的 TCP 协议关联的 RecvQ 或者 SendQ 队列，从操作系统缓冲区到用户缓冲区复制数据比较耗性能，`Buffer` 提供了另外一种直接操作操作系统缓冲区的的方式即 `ByteBuffer.allocateDirector(size)`，这个方法返回的 `byteBuffer` 就是与底层存储空间关联的缓冲区，它通过 Native 代码操作非 JVM 堆的内存空间。每次创建或者释放的时候都调用一次 `System.gc()`。