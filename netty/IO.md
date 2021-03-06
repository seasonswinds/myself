# I/O模型

### Old I/O(BIO)有什么缺点？

- 没有数据缓冲区，性能不好；
- 没有Channel概念，只有输入流、输出流；
- 同步阻塞I/O，容易导致通信线程长时间阻塞；
- 支持的字符集有限，可移植性差；

### Linux网络I/O模型有哪些？

- 阻塞I/O
- 非阻塞I/O
- I/O复用
- 信号驱动I/O
- 异步I/O

### epoll相对于select和poll做了哪些改进？

- 一个进程打开的文件描述符个数不受限制
- I/O效率不会随着FD数目的增加而线性下降
- 使用mmap加速内核与用户空间的消息传递
- API更加简单

# TCP粘包/拆包

### 什么是TCP粘包/拆包机制？

假设发送两个数据包D1和D2
- 接收端分两次读取到两个独立数据包——没有粘包/拆包
- 接收端一次收到两个数据包，D1和D2粘在一起——TCP粘包
- 接收端分两次读取到两个数据包，第一次读到了完整的D1包和D2包的一部分，第二次读到D2包的剩余内容——TCP拆包
- 接收端分两次读取到两个数据包，第一次读到了D1的一部分内容，第二次读到了D1包的剩余部分和D2包整包——TCP拆包
- 接收端分多次才能将D1、D2接收完——多次拆包

### TCP粘包/拆包发生的原因是什么？

- 应用程序写入字节大小大于套接口发送缓冲区大小；
- 进行MSS大小的TCP分段；
- 以太网帧的payload大于MTU进行IP分片；

### 如何解决TCP粘包/拆包问题？

- 消息定长，如果消息不够长，补空格；

- 在包尾添加分隔符；

  - LineBasedFrameDecoder

    > 根据'\n' or '\r\n'判断消息是否结束

  - DelimiterBasedFrameDecoder

    > 根据指定分隔符判断消息是否结束

- 将消息分为消息头和消息体，消息头记录消息总长度

  - FixedLengthFrameDecoder

# 编解码

### Java序列化的目的是什么？

- 网络传输
- 对象持久化

### Java序列化的缺点

- 无法跨语言
- 序列化后码流太大
- 序列化性能差

### 有哪些主流编解码框架？

- Protobuf(PB)

  > 二进制编码

- Thrift

- Marshalling

  > JBoss内部使用

- MessagePack

  

# 零拷贝

### 什么是零拷贝技术？

```
在计算机执行操作时
CPU 不需要先将数据从一个内存区域复制到另一个内存区域
从而可以减少上下文切换以及 CPU 的拷贝时间。
在数据从网络设备到用户程序空间传递的过程中
减少数据拷贝次数，减少系统调用
实现 CPU 的零参与
彻底消除 CPU 在这方面的负载
```
- DMA数据传输技术
- 内存区域映射技术

### 零拷贝技术有什么作用？

- 零拷贝机制可以减少数据在内核缓冲区和用户进程缓冲区之间反复的 I/O 拷贝操作。
- 零拷贝机制可以减少用户进程地址空间和内核地址空间之间因为上下文切换而带来的 CPU 开销。

### 在用户进程和物理内存（磁盘存储器）之间引入虚拟内存主要有什么优点？

- 地址空间：更大、连续的
- 进程隔离。
- 数据保护：每块虚拟内存都有相应的读写属性，这样就能保护程序的代码段不被修改，数据块不能被执行等，增加了系统的安全性。
- 内存映射：有了虚拟内存之后，可以直接映射磁盘上的文件（可执行文件或动态库）到虚拟地址空间。
这样可以做到物理内存延时分配，只有在需要读相应的文件的时候，才将它真正的从磁盘上加载到内存中来，而在内存吃紧的时候又可以将这部分内存清空掉，提高物理内存利用效率，并且所有这些对应用程序都是透明的。
- 共享内存：比如动态库只需要在内存中存储一份，然后将它映射到不同进程的虚拟地址空间中，让进程觉得自己独占了这个文件。
进程间的内存共享也可以通过映射同一块物理内存到进程的不同虚拟地址空间来实现共享。
- 物理内存管理：物理地址空间全部由操作系统管理，进程无法直接分配和回收，从而系统可以更好的利用内存，平衡进程间对内存的需求。

### 一次普通的磁盘I/O流程是什么？

![磁盘I/O流程图](https://wx3.sinaimg.cn/mw1024/ce7c2972ly1g80zk382lwj20r40jmjt8.jpg)

### 什么是DMA技术？

> DMA 的全称叫直接内存存取（Direct Memory Access），是一种允许外围设备（硬件子系统）直接访问系统主内存的机制。
>
> 系统主内存于硬盘或网卡之间的数据传输可以绕开 CPU 的全程调度。
>
> 目前大多数的硬件设备，包括磁盘控制器、网卡、显卡以及声卡等都支持 DMA 技术。

![使用DBM机制的磁盘I/O流程图](https://wx2.sinaimg.cn/mw1024/ce7c2972ly1g80zx1bjy4j20u00hugmb.jpg)

### 传统I/O的一次读写流程是什么？

![传统I/O的一次读写流程](https://wx3.sinaimg.cn/mw1024/ce7c2972ly1g8101coq5aj20mx0fd74t.jpg)

> 整个过程涉及 2 次 CPU 拷贝、2 次 DMA 拷贝，总共 4 次拷贝，以及 4 次上下文切换 

### 零拷贝的实现方式有哪些？

- 用户态直接 I/O

  > 只能适用于不需要内核缓冲区处理的应用程序
  >
  > 这些应用程序通常在进程地址空间有自己的数据缓存机制，称为自缓存应用程序，例如数据库。

- 减少数据拷贝次数

  - 使用mmap + write替代原来的read + write，减少了一次CPU拷贝

    >  使用 mmap 的目的是将内核中读缓冲区（read buffer）的地址与用户空间的缓冲区（user buffer）进行映射，实现内核缓冲区与应用程序内存的共享。
    >
    > 省去了将数据从内核读缓冲区（read buffer）拷贝到用户缓冲区（user buffer）的过程。
    >
    > 内核读缓冲区（read buffer）仍需将数据拷贝到内核写缓冲区（socket buffer）。
    >
    > ![mmap + write I/O流程](https://wx3.sinaimg.cn/mw1024/ce7c2972ly1g810bgv031j20ml0f13z7.jpg)

  - Sendfile，不仅减少了 CPU 拷贝的次数，还减少了上下文切换的次数

    > 通过 Sendfile 系统调用，数据可以直接在内核空间内部进行 I/O 传输，从而省去了数据在用户空间和内核空间之间的来回拷贝。
    >
    > 与 mmap 内存映射方式不同的是， Sendfile 调用中 I/O 数据对用户空间是完全不可见的。也就是说，这是一次完全意义上的数据传输过程。
    >
    > ![Sendfile I/O流程](https://wx1.sinaimg.cn/mw1024/ce7c2972ly1g810hj5npxj20m50fb3z0.jpg)
    >
    > 整个拷贝过程会发生 2 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝。
    >
    > Sendfile 存在的问题是用户程序不能对数据进行修改，而只是单纯地完成了一次数据传输过程。

  - Sendfile + DMA gather copy，省去了内核空间中仅剩的 1 次 CPU 拷贝操作

    > 在硬件的支持下，Sendfile 拷贝方式不再从内核缓冲区的数据拷贝到 socket 缓冲区，取而代之的仅仅是缓冲区文件描述符和数据长度的拷贝。
    >
    > 这样 DMA 引擎直接利用 gather 操作将页缓存中数据打包发送到网络中即可，本质就是和虚拟内存映射的思路类似。
    >
    > ![Sendfile + DMA gather copy I/O流程](https://wx3.sinaimg.cn/mw1024/ce7c2972ly1g811ap7r21j20ls0faaaq.jpg)
    >
    > 整个拷贝过程会发生 2 次上下文切换、0 次 CPU 拷贝以及 2 次 DMA 拷贝。
    >
    > 同样存在用户程序不能对数据进行修改的问题，而且本身需要硬件的支持，它只适用于将数据从文件拷贝到 socket 套接字上的传输过程
    
  - Splice

    > 在内核空间的读缓冲区（read buffer）和网络缓冲区（socket buffer）之间建立管道（pipeline），从而避免了两者之间的 CPU 拷贝操作
    >
    > ![Splice I/O流程](https://wx4.sinaimg.cn/mw1024/ce7c2972gy1g81nqr6mqoj20lx0f0dgc.jpg)
    >
    > 整个拷贝过程会发生 2 次上下文切换，0 次 CPU 拷贝以及 2 次 DMA 拷贝

- 写时复制技术

  > 写时复制指的是当多个进程共享同一块数据时，如果其中一个进程需要对这份数据进行修改，那么就需要将其拷贝到自己的进程地址空间中。
  >
  > 这种方法在某种程度上能够降低系统开销，如果某个进程永远不会对所访问的数据进行更改，那么也就永远不需要拷贝。
  
- 缓冲区共享（在 Solaris 上实现的 fbuf，Fast Buffer，快速缓冲区）

  > 每个进程都维护着一个缓冲区池，这个缓冲区池能被同时映射到用户空间（user space）和内核态（kernel space），内核和用户共享这个缓冲区池
  >
  > ![FBUF I/O流程](https://wx2.sinaimg.cn/mw1024/ce7c2972gy1g81nunnhkij20np0fedge.jpg)

#### 零拷贝技术对比

|        拷贝方式        | CPU拷贝 | DMA  |       系统调用        | 上下文切换 |
| :--------------------: | :-----: | :--: | :-------------------: | :--------: |
| 传统方式(read + write) |    2    |  2   |     read + write      |     4      |
| 内存映射(mmap + write) |    1    |  2   |     mmap + write      |     4      |
|        sendfile        |    1    |  2   |       sendfile        |     2      |
| sendfile + DMA gather  |    0    |  2   | sendfile + DMA gather |     2      |
|         splice         |    0    |  2   |        splice         |     2      |

### Java NIO是如何使用零拷贝技术的？

> - Channel是全双工的（双向传输），相当于操作系统的内核空间（kernel space）的缓冲区，既可能是读缓冲区（read buffer），也可能是网络缓冲区（socket buffer）。
> - Buffer，相当于操作系统的用户空间（user space）中的用户缓冲区（user buffer），分为堆内存（HeapBuffer）和堆外内存（DirectBuffer），这是通过 malloc() 分配出来的用户态内存。

- mmap
- sendfile

### Netty是如何使用零拷贝技术的？

- Netty 通过 DefaultFileRegion 类对 java.nio.channels.FileChannel 的 tranferTo() 方法进行包装，在文件传输时可以将文件缓冲区的数据直接发送到目的通道（Channel）。
- ByteBuf 可以通过 wrap 操作把字节数组、ByteBuf、ByteBuffer 包装成一个 ByteBuf 对象, 进而避免了拷贝操作。
- ByteBuf 支持 Slice 操作, 因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝。
- Netty 提供了 CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免了各个 ByteBuf 之间的拷贝。

### RocketMQ&Kafka是如何使用零拷贝技术的？

- RocketMQ 选择了 mmap+write ，适用于业务级消息这种小块文件的数据持久化和传输。
-  Kafka 的索引文件使用的是 mmap+write 方式，数据文件使用的是 Sendfile 方式。，适用于系统日志消息这种高吞吐量的大块文件的数据持久化和传输。