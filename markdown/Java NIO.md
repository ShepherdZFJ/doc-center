# Java NIO

NIO (Non-blocking/New I/O)。Java 中的 NIO 于 Java 1.4 中引入，对应 `java.nio` 包，提供了 `Channel` , `Selector`，`Buffer` 等抽象。NIO 中的 N 可以理解为 Non-blocking，不单纯是 New。它是支持面向缓冲的，基于通道的 I/O 操作方法。 对于高负载、高并发的（网络）应用，应使用 NIO 。

Java 中的 NIO 可以看作是 **I/O 多路复用模型**。也有很多人认为，Java 中的 NIO 属于同步非阻塞 IO 模型。

### **1. BIO 和 NIO 拷贝文件的区别**

下面用拷贝一张图片例子讲述**BIO 和 NIO 拷贝文件的区别**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxjVlhYdHFKVlFKQ3FWVHN3cmJEbzA2eU9VSUhuTnBDQTk1RmVxSkFLQW02dWF4c0lsakpoM0EvNjQw?x-oss-process=image/format,png)

这个时候就要来了解了解操作系统底层是怎么对 IO 和 NIO 进行区别的，我会用尽量通俗的文字带你理解，可能并不是那么严谨。

操作系统最重要的就是内核，它既可以访问受保护的内存，也可以访问底层硬件设备，所以为了保护内核的安全，操作系统将底层的虚拟空间分为了**用户空间**和**内核空间**，其中用户空间就是给用户进程使用的，内核空间就是专门给操作系统底层去使用的。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxpY2lhUXh1eFRrMnpRZ25WcnBpYldxUkJKb1dENVg0VGdGanNDNXBqZU5MRFl6aWFUQmVXUEZkVkNnLzY0MA?x-oss-process=image/format,png)

接下来，有一个 Java 进程希望把小菠萝这张图片从磁盘上拷贝，那么内核空间和用户空间都会有一个缓冲区。

- 这张照片就会从磁盘中读出到内核缓冲区中保存，然后操作系统将内核缓冲区中的这张图片字节数据拷贝到用户进程的缓冲区中保存下来，对应着下面这幅图：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxuN1VqNmFyRWZtcVd0d281dWpPcGx4SmF3SDNpY0tsRG9tS3cxcFBYdERFemFBekRwMkVIelpRLzY0MA?x-oss-process=image/format,png)

- 然后用户进程会希望把缓冲区中的字节数据写到磁盘上的另外一个地方，会将数据拷贝到 Socket 缓冲区中，最终操作系统再将 Socket 缓冲区的数据写到磁盘的指定位置上。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmw2ak1HZE9aMU1aRGljSFZmNExpYVdGTHFpYmhCeWIwWjd1aWFjSnJVR1RnYVBvZTJKZzA3WHM3YWljZy82NDA?x-oss-process=image/format,png)

这一轮操作下来，我们数数经过了几次数据的拷贝？4 次。有 2 次是内核空间和用户空间之间的数据拷贝，这两次拷贝涉及到用户态和内核态的切换，需要CPU参与进来，进行上下文切换。

而另外 2 次是硬盘和内核空间之间的数据拷贝，这个过程利用到 DMA与系统内存交换数据，不需要 CPU 的参与。

导致 IO 性能瓶颈的原因：内核空间与用户空间之间数据过多无意义的拷贝，以及多次上下文切换。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9QbjRTbTBSc0F1Z0Zva1BoQ3RiZkU5MGg3eVZ0RTVUcVVpY2ZjNVd5a1JPRlA3eUpOQ1pjQWNHdVlOeDVYU0RNakhIV3Y3UVg1Q3pmd3AxczJYVjZzYkEvNjQw?x-oss-process=image/format,png)

**在用户空间与内核空间之间的操作，会涉及到上下文的切换，这里需要 CPU 的干预，而数据在两个空间之间来回拷贝，也需要 CPU 的干预，这无疑会增大 CPU 的压力，NIO 是如何减轻 CPU 的压力？运用操作系统的零拷贝技术。**

### **操作系统的零拷贝**

所以，操作系统出现了一个全新的概念，解决了 IO 瓶颈：零拷贝。零拷贝指的是**内核空间与用户空间之间的零次拷贝**。

零拷贝可以说是 IO 的一大救星，操作系统底层有许多种零拷贝机制，我这里仅针对 Java NIO 中使用到的其中一种零拷贝机制展开讲解。

在 Java NIO 中，零拷贝是通过**用户空间和内核空间的缓冲区共享一块物理内存**实现的，也就是说上面的图可以演变成这个样子。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxEcXdQanZ2eUYyZkNpY2hPYVJYWjdsamJkMGFJaDRONmZBaWJ2b2liV0p1bnU1ZEpJbjBUU3FiREEvNjQw?x-oss-process=image/format,png)

这时，无论是用户空间还是内核空间操作自己的缓冲区，本质上都是**操作这一块共享内存**中的缓冲区数据，**省去了用户空间和内核空间之间的数据拷贝操作**。

现在我们重新来拷贝文件，就会变成下面这个步骤：

- 用户进程通过系统调用 read() 请求读取文件到用户空间缓冲区（第一次上下文切换），用户态 -> 核心态，数据从硬盘读取到内核空间缓冲区中（第一次数据拷贝）；
- 系统调用返回到用户进程（第二次上下文切换），此时用户空间与内核空间共享这一块内存（缓冲区），所以不需要从内核缓冲区拷贝到用户缓冲区；
- 用户进程发出 write() 系统调用请求写数据到硬盘上（第三次上下文切换），此时需要将内核空间缓冲区中的数据拷贝到内核的 Socket 缓冲区中（第二次数据拷贝）；
- 由 DMA 将 Socket 缓冲区的内容写到硬盘上（第三次数据拷贝），write() 系统调用返回（第四次上下文切换）；

整个过程就如下面这幅图所示。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxVUUNsVzJidTVPVEt1cmE4VXUwdHM0Z0dDaWNFR01wWFhCV09ZRFlqcmxiY0owdUFldVVGUXhRLzY0MA?x-oss-process=image/format,png)

图中，需要 CPU 参与工作的步骤只有第③个步骤，对比于传统的 IO，CPU 需要在用户空间与内核空间之间参与拷贝工作，需要无意义地占用 2 次 CPU 资源，导致 CPU 资源的浪费。

下面总结一下操作系统中零拷贝的优点：

降低 CPU 的压力：避免 CPU 需要参与内核空间与用户空间之间的数据拷贝工作；

减少不必要的拷贝：避免用户空间与内核空间之间需要进行数据拷贝；

### 2.NIO核心组件

#### 2.1 channel(通道)

在 NIO 中，不再是面向流的 IO 了，而是面向缓冲区，它会建立一个通道（Channel），该通道我们可以理解为铁路，该铁路上可以运输各种货物，而通道上会有一个**缓冲区（Buffer）用于存储真正的数据，缓冲区我们可以理解为一辆火车。**

**通道（铁路）只是作为运输数据的一个连接资源，而真正存储数据的是缓冲区（火车）。即通道负责传输，缓冲区负责存储**

channel的实现有：

- **FileChannel**：操作文件的通道，从文件中读写数据
- **DatagramChannel**：能通过udp读取网络中数据
- **SocketChannel**：能通过tcp读取网络中的 数据
- **ServerSocketChannel**： 可以监听新进来的TCP连接，像Web 服务器一样。对每一个新进来的连接会创建一个SocketChannel

示例代码：FileChannel读数据

```java
package com.shepherd.example.nio.channel;

import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/3/13 22:39
 */
public class FileChannelReadDemo {
    // FileChannel读取数据到buffer中
    public static void main(String[] args) throws Exception {
        //创建FileChannel
        RandomAccessFile aFile = new RandomAccessFile("/Users/shepherdmy/Desktop/nio/test1.txt","rw");
        FileChannel channel = aFile.getChannel();

        //创建Buffer
        ByteBuffer buf = ByteBuffer.allocate(1024);

        //读取数据到buffer中
        int bytesRead = channel.read(buf);
        while(bytesRead != -1) {
            System.out.println("读取了："+bytesRead);
            buf.flip();
            while(buf.hasRemaining()) {
                System.out.println((char)buf.get());
            }
            buf.clear();
            bytesRead = channel.read(buf);
        }
        aFile.close();
        System.out.println("  结束了");
    }
}
```

fileChannel写数据：

```
package com.shepherd.example.nio.channel;

import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/3/13 23:08
 */
public class FileChannelWriteDemo {
    public static void main(String[] args) throws Exception {
        // FileChannel写数据，如果写入文件已经有数据，会被覆盖调
        RandomAccessFile aFile = new RandomAccessFile("/Users/shepherdmy/Desktop/nio/test2.txt","rw");
        FileChannel channel = aFile.getChannel();

        //创建buffer对象
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        String newData = "fileChannel write data: hello world, shepherd";
        buffer.clear();

        //写入内容
        buffer.put(newData.getBytes());

        buffer.flip();

        //FileChannel完成最终实现
        while (buffer.hasRemaining()) {
            channel.write(buffer);
        }

        //关闭
        channel.close();
    }
}
```

ServerSocketChannel:

```JAVA
package com.shepherd.example.nio.channel;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/3/21 11:39
 */
public class ServerSocketChannelDemo {

    public static void main(String[] args) throws Exception {
        //端口号
        int port = 8888;

        //buffer
        ByteBuffer buffer = ByteBuffer.wrap("hello shepherd nio socket".getBytes());

        //ServerSocketChannel
        ServerSocketChannel ssc = ServerSocketChannel.open();
        //绑定
        ssc.socket().bind(new InetSocketAddress(port));

        //设置非阻塞模式
        ssc.configureBlocking(false);

        //监听有新链接传入
        while(true) {
            SocketChannel sc = ssc.accept();
            if(sc == null) { //没有链接传入
                Thread.sleep(2000);
            } else {
                System.out.println("Incoming connection from: " + sc.socket().getRemoteSocketAddress());
                buffer.rewind(); //指针0
                sc.write(buffer);
                sc.close();
            }
        }
    }
}
```

#### 2.2 buffer(缓冲区)

## **缓冲区（Buffer）** 

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装 成 NIO Buffer 对象，并提供了一组方法，用来方便的访问该块内存。缓冲区实际上是 一个容器对象，更直接的说，其实就是一个数组，在 NIO 库中，所有数据都是用缓冲 区处理的。在读取数据时，它是直接读到缓冲区中的; 在写入数据时，它也是写入到 缓冲区中的;任何时候访问 NIO 中的数据，都是将它放到缓冲区中。而在面向流 I/O 系统中，所有数据都是直接写入或者直接将数据读取到 Stream 对象中

缓冲区是**存储数据**的区域，在 Java 中，缓冲区就是数组，为了可以操作不同数据类型的数据，Java 提供了许多不同类型的缓冲区，**除了布尔类型以外**，其它基本数据类型都有对应的缓冲区数组对象。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxydjc0ZVIwRE9FRGljOWtKMktwU3RpY2hhR2RycEg4VktDZ3RnajFoUFlYQUl1emliaWFpY3ZETlZ2Zy82NDA?x-oss-process=image/format,png)

使用 Buffer 读写数据，一般遵循以下四个步骤:

(1)写入数据到 Buffer

(2)调用 flip()方法

(3)从 Buffer 中读取数据

(4)调用 clear()方法或者 compact()方法

当向 buffer 写入数据时，buffer 会记录下写了多少数据。一旦要读取数据，需要通过 flip()方法将 Buffer 从写模式切换到读模式。在读模式下，可以读取之前写入到 buffer 的所有数据。一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有 两种方式能清空缓冲区:调用 clear()或 compact()方法。clear()方法会清空整个缓冲 区。compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起 始处，新写入的数据将放到缓冲区未读数据的后面。

示例如下：

```java
public class BufferDemo {

    @Test
    public void buffer01() throws Exception {
        //FileChannel
        RandomAccessFile aFile =
                new RandomAccessFile("Users/shepherdmy/Desktop/nio/test1.txt","rw");
        FileChannel channel = aFile.getChannel();

        //创建buffer，大小
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        //读
        int bytesRead = channel.read(buffer);

        while(bytesRead != -1) {
            //read模式
            buffer.flip();

            while(buffer.hasRemaining()) {
                System.out.println((char)buffer.get());
            }
            buffer.clear();
            bytesRead = channel.read(buffer);
        }

        aFile.close();
    }


    @Test
    public void buffer02() throws Exception {

//        //创建buffer
//        IntBuffer buffer = IntBuffer.allocate(8);
//
//        //buffer放
//        for (int i = 0; i < buffer.capacity(); i++) {
//            int j = 2*(i+1);
//            buffer.put(j);
//        }
//
//        //重置缓冲区
//        buffer.flip();
//
//        //获取
//        while(buffer.hasRemaining()) {
//            int value = buffer.get();
//            System.out.println(value+" ");
//        }

        // 1、获取Selector选择器
        Selector selector = Selector.open();

        // 2、获取通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 3.设置为非阻塞
        serverSocketChannel.configureBlocking(false);

        // 4、绑定连接
        serverSocketChannel.bind(new InetSocketAddress(9999));

        // 5、将通道注册到选择器上,并制定监听事件为：“接收”事件
        serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);

    }

}

```

#### 2.3 Selector

Selector 一般称 为选择器 ，也可以翻译为 多路复用器 。它是 Java NIO 核心组件中 的一个，用于检查一个或多个 NIO Channel(通道)的状态是否处于可读、可写。如此可以实现单线程管理多个 channels,也就是可以管理多个网络链接。

 选择器是提升 IO 性能的灵魂之一，它底层利用了多路复用 IO机制，让选择器可以监听多个 IO 连接，根据 IO 的状态响应到服务器端进行处理。通俗地说：**选择器可以监听多个 IO 连接，而传统的 BIO 每个 IO 连接都需要有一个线程去监听和处理**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxkTWtPMkNXOGlidGF5OExod3FpYUlhb2VVanlHRTA1UFVmeU9yQjVmeUNiRVBHbkVSdWtTYWlhUUEvNjQw?x-oss-process=image/format,png)

图中很明显的显示了在 BIO 中，每个 Socket 都需要有一个专门的线程去处理每个请求，而在 NIO 中，只需要一个 Selector 即可监听各个 Socket 请求，而且 Selector 并不是阻塞的，所以**不会因为多个线程之间切换导致上下文切换带来的开销。**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmx2ZVFibFZEUmpNWjllbU1zeHRWYU9MZEZWUXRGV0RueEU3M2h3WlFwTTJnazE2bnY2clhpYUFBLzY0MA?x-oss-process=image/format,png)

在 Java NIO 中，选择器是使用 Selector 类表示，Selector 可以接收各种 IO 连接，在 IO 状态准备就绪时，会通知该通道注册的 Selector，Selector 在**下一次轮询**时会发现该 IO 连接就绪，进而处理该连接。

示例：

```java
package com.shepherd.example.nio.channel.nio.selector;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/3/21 13:54
 */
public class SelectorDemo {
    //服务端代码
    @Test
    public void serverDemo() throws Exception {
        //1 获取服务端通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        //2 切换非阻塞模式
        serverSocketChannel.configureBlocking(false);

        //3 创建buffer
        ByteBuffer serverByteBuffer = ByteBuffer.allocate(1024);

        //4 绑定端口号
        serverSocketChannel.bind(new InetSocketAddress(8080));

        //5 获取selector选择器
        Selector selector = Selector.open();

        //6 通道注册到选择器，进行监听
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        //7 选择器进行轮询，进行后续操作
        while(selector.select()>0) {
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            //遍历
            Iterator<SelectionKey> selectionKeyIterator = selectionKeys.iterator();
            while(selectionKeyIterator.hasNext()) {
                //获取就绪操作
                SelectionKey next = selectionKeyIterator.next();
                //判断什么操作
                if(next.isAcceptable()) {
                    //获取连接
                    SocketChannel accept = serverSocketChannel.accept();

                    //切换非阻塞模式
                    accept.configureBlocking(false);

                    //注册
                    accept.register(selector,SelectionKey.OP_READ);

                } else if(next.isReadable()) {
                    SocketChannel channel = (SocketChannel) next.channel();

                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                    //读取数据
                    int length = 0;
                    while((length = channel.read(byteBuffer))>0) {
                        byteBuffer.flip();
                        System.out.println(new String(byteBuffer.array(),0,length));
                        byteBuffer.clear();
                    }

                }

                selectionKeyIterator.remove();
            }
        }
    }

    //客户端代码
    @Test
    public void clientDemo() throws Exception {
        //1 获取通道，绑定主机和端口号
        SocketChannel socketChannel =
                SocketChannel.open(new InetSocketAddress("127.0.0.1",8080));

        //2 切换到非阻塞模式
        socketChannel.configureBlocking(false);

        //3 创建buffer
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        //4 写入buffer数据
        byteBuffer.put(new Date().toString().getBytes());

        //5 模式切换
        byteBuffer.flip();

        //6 写入通道
        socketChannel.write(byteBuffer);

        //7 关闭
        byteBuffer.clear();
    }

    public static void main(String[] args) throws IOException {
        //1 获取通道，绑定主机和端口号
        SocketChannel socketChannel =
                SocketChannel.open(new InetSocketAddress("127.0.0.1",8080));

        //2 切换到非阻塞模式
        socketChannel.configureBlocking(false);

        //3 创建buffer
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        Scanner scanner = new Scanner(System.in);
        while(scanner.hasNext()) {
            String str = scanner.next();
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            //4 写入buffer数据
            byteBuffer.put((sdf.format(new Date())+"===>"+str).getBytes());

            //5 模式切换
            byteBuffer.flip();

            //6 写入通道
            socketChannel.write(byteBuffer);

            //7 关闭
            byteBuffer.clear();
        }

    }
}

```

### 3.redis中I/O 多路复用模型的应用

redis之所以能这么快：一方面，**Redis 的大部分操作在内存上完成，再加上它采用了高效的数据结构**，例如哈希表和跳表，这是它实现高性能的一个重要原因。另一方面，就是 Redis 采用了**多路复用机制**，使其在网络 IO 操作中能并发处理大量的客户端请求，实现高吞吐率。接下来，我们就重点学习下多路复用机制。

#### 3.1基本 IO 模型与阻塞点

以 Get 请求获取数据为例，为了处理一个 Get 请求，需要监听客户端请求（bind/listen），和客户端建立连接（accept），从 socket 中读取请求（recv），解析客户端发送请求（parse），根据请求类型读取键值数据（get），最后给客户端返回结果，即向 socket 中写回数据（send）

下图显示了这一过程，其中，bind/listen、accept、recv、parse 和 send 属于网络 IO 处理，而 get 属于键值数据操作。既然 Redis 是单线程，那么，最基本的一种实现是在一个线程中依次执行上面说的这些操作。

![](https://static001.geekbang.org/resource/image/e1/c9/e18499ab244e4428a0e60b4da6575bc9.jpg)

但是，在这里的网络 IO 操作中，有潜在的阻塞点，分别是 accept() 和 recv()。当 Redis 监听到一个客户端有连接请求，但一直未能成功建立起连接时，会阻塞在 accept() 函数这里，导致其他客户端无法和 Redis 建立连接。类似的，当 Redis 通过 recv() 从一个客户端读取数据时，如果数据一直没有到达，Redis 也会一直阻塞在 recv()。这就导致 Redis 整个线程阻塞，无法处理其他客户端请求，效率很低。不过，幸运的是，socket 网络模型本身支持非阻塞模式

#### 3.2非阻塞模式

Socket 网络模型的非阻塞模式设置，主要体现在三个关键的函数调用上，如果想要使用 socket 非阻塞模式，就必须要了解这三个函数的调用返回类型和设置模式。接下来，我们就重点学习下它们。在 socket 模型中，不同操作调用后会返回不同的套接字类型。socket() 方法会返回主动套接字，然后调用 listen() 方法，将主动套接字转化为监听套接字，此时，可以监听来自客户端的连接请求。最后，调用 accept() 方法接收到达的客户端连接，并返回已连接套接字。

![](https://static001.geekbang.org/resource/image/1c/4a/1ccc62ab3eb2a63c4965027b4248f34a.jpg)

针对监听套接字，我们可以设置非阻塞模式：当 Redis 调用 accept() 但一直未有连接请求到达时，Redis 线程可以返回处理其他操作，而不用一直等待。但是，你要注意的是，调用 accept() 时，已经存在监听套接字了。虽然 Redis 线程可以不用继续等待，但是总得有机制继续在监听套接字上等待后续连接请求，并在有请求时通知 Redis。

类似的，我们也可以针对已连接套接字设置非阻塞模式：Redis 调用 recv() 后，如果已连接套接字上一直没有数据到达，Redis 线程同样可以返回处理其他操作。我们也需要有机制继续监听该已连接套接字，并在有数据达到时通知 Redis。

这样才能保证 Redis 线程，既不会像基本 IO 模型中一直在阻塞点等待，也不会导致 Redis 无法处理实际到达的连接请求或数据。

到此，Linux 中的 IO 多路复用机制就要登场了。

#### 3.3基于多路复用的高性能 I/O 模型

**Linux 中的 IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听套接字和已连接套接字。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果**

下图就是基于多路复用的 Redis IO 模型。图中的多个 FD 就是刚才所说的多个套接字。Redis 网络框架调用 epoll 机制，让内核监听这些套接字。此时，Redis 线程不会阻塞在某一个特定的监听或已连接套接字上，也就是说，不会阻塞在某一个特定的客户端请求处理上。正因为此，Redis 可以同时和多个客户端连接并处理请求，从而提升并发性。

![](https://static001.geekbang.org/resource/image/00/ea/00ff790d4f6225aaeeebba34a71d8bea.jpg)

为了在请求到达时能通知到 Redis 线程，**select/epoll 提供了基于事件的回调机制，即针对不同事件的发生，调用相应的处理函数**。那么，**回调机制是怎么工作的呢？其实，select/epoll 一旦监测到 FD 上有请求到达时，就会触发相应的事件。这些事件会被放进一个事件队列，Redis 单线程对该事件队列不断进行处理。这样一来，Redis 无需一直轮询是否有请求实际发生，这就可以避免造成 CPU 资源浪费。同时，Redis 在对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。因为 Redis 一直在对事件队列进行处理，所以能及时响应客户端请求，提升 Redis 的响应性能**

