# Java IO

### 1.IO体系

#### 1.1 IO简介

IO 的操作方式常分为：同步阻塞BIO、同步非阻塞NIO、异步非阻塞AIO

Java IO流是一个庞大的生态环境，其内部提供了很多不同的输入流和输出流，细分下去还有字节流和字符流，甚至还有缓冲流提高 IO 性能，转换流将字节流转换为字符流······看到这些就已经对 IO 产生恐惧了，在日常开发中少不了对文件的 IO 操作，虽然 apache 已经提供了 `Commons IO` 这种封装好的组件，但面对特殊场景时，我们仍需要自己去封装一个高性能的文件 IO 工具类，本文将会解析 Java IO 中涉及到的各个类，以及讲解如何正确、高效地使用它们。

#### 1.2 什么是IO流

知识科普：我们知道任何一个文件都是以二进制形式存在于设备中，计算机就只有 `0` 和 `1`，你能看见的东西全部都是由这两个数字组成，你看这篇文章时，这篇文章也是由01组成，只不过这些二进制串经过各种转换演变成一个个文字、一张张图片跃然屏幕上。

而流就是将这些二进制串在各种设备之间进行传输，如果你觉得有些抽象，我举个例子就会好理解一些：

> “
>
> 下图是一张图片，它由01串组成，我们可以通过程序把一张图片拷贝到一个文件夹中，
>
> 把图片转化成二进制数据集，把数据一点一点地传递到文件夹中 , 类似于水的流动 , 这样整体的数据就是一个数据流
>
> ”

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmx6N0kwOGxiTjFxNWlhSk1jMUx3Y25jWFE4clEwNnBWazdsUmRneDQ5ZGpxQ0JNQ0xjcTZBWm9nLzY0MA?x-oss-process=image/format,png)

IO 流读写数据的特点：

- 顺序读写。读写数据时，大部分情况下都是按照顺序读写，读取时从文件开头的第一个字节到最后一个字节，写出时也是也如此（RandomAccessFile 可以实现随机读写, NIO的channel既可以RandomAccessFile获取）
- 字节数组。读写数据时本质上都是对字节数组做读取和写出操作，即使是字符流，也是在字节流基础上转化为一个个字符，所以字节数组是 IO 流读写数据的本质。

#### 1.3 流的分类

根据数据流向不同分类：输入流 和 输出流

- 输入流：从磁盘或者其它设备中将数据输入到进程中
- 输出流：将进程中的数据输出到磁盘或其它设备上保存

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxKUkdpYVA4SHltSHZDQW95dGljd0NtOEhTWFJpY21Ja0lWeU5GaWFEN3dQSnV4bDFpYjg2MlNyaWNBVFEvNjQw?x-oss-process=image/format,png)

1

图示中的硬盘只是其中一种设备，还有非常多的设备都可以应用在IO流中，例如：打印机、硬盘、显示器、手机······

根据处理数据的基本单位不同分类：字节流 和 字符流

- 字节流：以字节（8 bit）为单位做数据的传输
- 字符流：以字符为单位（1字符 = 2字节）做数据的传输

> “
>
> 字符流的本质也是通过字节流读取，Java 中的字符采用 Unicode 标准，在读取和输出的过程中，通过以字符为单位，查找对应的码表将字节转换为对应的字符。
>
> ”

**为什么 I/O 流操作要分为字节流操作和字符流操作呢？**

回答：字符流是由 Java 虚拟机将字节转换得到的，问题就出在这个过程还算是非常耗时，并且，如果我们不知道编码类型就很容易出现乱码问题。所以， I/O 流就干脆提供了一个直接操作字符的接口，方便我们平时对字符进行流操作。

面对字节流和字符流，很多读者都有疑惑：什么时候需要用字节流，什么时候又要用字符流？

我这里做一个简单的概括，你可以按照这个标准去使用：

字符流只针对字符数据进行传输，所以如果是文本数据，优先采用字符流传输；除此之外，其它类型的数据（图片、音频等），最好还是以字节流传输。

根据这两种不同的分类，我们就可以做出下面这个表格，里面包含了 IO 中最核心的 4 个顶层抽象类：



| 数据流向 / 数据类型 | 字节流       | 字符流 |
| :------------------ | :----------- | :----- |
| 输入流              | InputStream  | Reader |
| 输出流              | OutputStream | Writer |

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9saWJZUnV2VUxUZFdCRjhqcFRaQU9WQXBBcm1UVXJhRmxOTXZpYkdOOUFVYVdrOVFRQWVNcHIybGttSU5oUzNxZVIzWkwwc2liT1J3YmwzWXNKZHB1NFVwQS82NDA?x-oss-process=image/format,png)

看到 `Stream` 就知道是字节流，看到 `Reader / Writer` 就知道是字符流。

这里还要额外补充一点：Java IO 提供了字节流转换为字符流的转换类，称为转换流。

| 转换流 / 数据类型        | 字节流与字符流之间的转换 |
| :----------------------- | :----------------------- |
| （输入）字节流 => 字符流 | InputStreamReader        |
| （输出）字符流 => 字节流 | OutputStreamWriter       |

### 2.三种不同实现的IO操作方式

以一个经典的烧开水的例子通俗地讲解它们之间的区别

| 类型 | 烧开水                                                       |
| :--- | :----------------------------------------------------------- |
| BIO  | 一直监测着某个水壶，该水壶烧开水后再监测下一个水壶           |
| NIO  | 每隔一段时间就看看所有水壶的状态，哪个水壶烧开水就去处理哪个水壶 |
| AIO  | 不用监测水壶，每个水壶烧开水后都会主动通知线程说：“我的水烧开了，来处理我吧” |

接下来我详解讲述一下BIO的使用

### 3.BIO (Blocking I/O)

**BIO 属于同步阻塞 IO 模型** 。

同步阻塞 IO 模型中，应用程序发起 read 调用后，会一直阻塞，直到内核把数据拷贝到用户空间。在客户端连接数量不高的情况下，是没问题的。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量，所以才有了后续的NIO,AIO

下面通过文件复制看一下io操作示例：

```java
package com.shepherd.example.bio;

import org.apache.commons.io.FileUtils;
import org.apache.commons.io.IOUtils;

import java.io.*;

/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/3/20 00:48
 */
public class BioDemo {
    public static void main(String[] args) throws IOException {
        String source = "/Users/shepherdmy/baiduYunDownload/72-Elasticsearch核心技术与实战/13丨通过Analyzer进行分词.mp4";
        String destination = "/Users/shepherdmy/Desktop/bio/bak/es.mp4";
        long start = System.currentTimeMillis();
        copyWithFileInputStream(source, destination);
        long end = System.currentTimeMillis();
        System.out.println(end-start);
    }

    public static void copyWithFileInputStream(String source, String destination) throws IOException {
        // 创建输入流，读取文件内容
        InputStream inputStream = new FileInputStream(source);
        OutputStream outputStream = new FileOutputStream(destination);
        byte[] buf = new byte[8192];
        int len;
        while ((len = inputStream.read(buf)) > 0) {
            outputStream.write(buf, 0, len);
        }
        inputStream.close();
        outputStream.close();
    }

    public static void copyWithBufferInputStream(String source, String destination) throws IOException {

        InputStream in = new FileInputStream(source);
        OutputStream out = new FileOutputStream(destination);
        BufferedInputStream bufferedInputStream = new BufferedInputStream(in);
        BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(out);
        // bufferedInputStream默认缓冲区大小为8192
        // BufferedInputStream比FileInputStream多了一个缓冲区，执行read时先从缓冲区读取，当缓冲区数据读完时再把缓冲区填满。
        // 因此，当每次读取的数据量很小时，FileInputStream每次都是从硬盘读入，而BufferedInputStream大部分是从缓冲区读入。读取内存速度比
        // 读取硬盘速度快得多，因此BufferedInputStream效率高。
        // BufferedInputStream的默认缓冲区大小是8192字节。当每次读取数据量接近或远超这个值时，两者效率就没有明显差别了。
        byte[] buf = new byte[8192];
        int len;
        while ((len = bufferedInputStream.read(buf)) > 0) {
            bufferedOutputStream.write(buf, 0, len);
        }
        bufferedInputStream.close();
        bufferedOutputStream.close();
    }
  
      public static void copyWithFileChannel(String source, String destination) throws IOException {
        // 打开文件输入流
        FileChannel inChannel = new FileInputStream(source).getChannel();
        // 打开文件输出流
        FileChannel outChannel = new FileOutputStream(destination).getChannel();
        // 分配 1024 个字节大小的缓冲区
        ByteBuffer buf = ByteBuffer.allocate(8192);
        // 将数据从通道读入缓冲区
        while (inChannel.read(buf) != -1) {
            // 切换缓冲区的读写模式
            buf.flip();
            // 将缓冲区的数据通过通道写到目的地
            outChannel.write(buf);
            // 清空缓冲区，准备下一次读
            buf.clear();
        }
        inChannel.close();
        outChannel.close();

    }

    public static void copyWithUtils(String source, String destination) throws IOException {
        FileUtils.copyFile(new File(source), new File(destination));
    }
}
```

注意：apache给我们提供的IOUtils和FileUtils提供了很多对流和文件的封装方法，我们可以直接，但是底层实现有可能是BIO，也有可能是NIO，如上面的`FileUtils.copyFile()`方法，其核心逻辑是：

```java
    private static void doCopyFile(File srcFile, File destFile, boolean preserveFileDate) throws IOException {
        if (destFile.exists() && destFile.isDirectory()) {
            throw new IOException("Destination '" + destFile + "' exists but is a directory");
        }

        FileInputStream fis = null;
        FileOutputStream fos = null;
        FileChannel input = null;
        FileChannel output = null;
        try {
            fis = new FileInputStream(srcFile);
            fos = new FileOutputStream(destFile);
            input  = fis.getChannel();
            output = fos.getChannel();
            long size = input.size();
            long pos = 0;
            long count = 0;
            while (pos < size) {
                count = size - pos > FILE_COPY_BUFFER_SIZE ? FILE_COPY_BUFFER_SIZE : size - pos;
                pos += output.transferFrom(input, pos, count);
            }
        } finally {
            IOUtils.closeQuietly(output);
            IOUtils.closeQuietly(fos);
            IOUtils.closeQuietly(input);
            IOUtils.closeQuietly(fis);
        }

        if (srcFile.length() != destFile.length()) {
            throw new IOException("Failed to copy full contents from '" +
                    srcFile + "' to '" + destFile + "'");
        }
        if (preserveFileDate) {
            destFile.setLastModified(srcFile.lastModified());
        }
    }
```

使用channel通道进行读写，所以应该是NIO实现。

