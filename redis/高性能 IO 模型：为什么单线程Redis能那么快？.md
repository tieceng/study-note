## 高性能 IO 模型：为什么单线程Redis能那么快？

redis 的单线程只要是只 redis 的网络 IO 和键值对读写是由一个线程来完成的，这也是 redis 对外提供键值存储服务的主要流程。但是 redis 的其他功能，比如持久化、一步删除、集群数据同步等，是由额外的线程来执行的。

所以严格来说 redis 不是单线程。

### 为什么用单线程？为什么单线程能这么快？

我们来看下一下 redis 单线程设计机制以及多路复用机制

### redis 为什么使用单线程？

更好的了解 redis 为什么用单线程，我们要了解多线程的开销

#### 多线程的开销

日常写程序时，使用多线程有一下优点“使用多线程，可以增加系统吞吐率，看也增加系统拓展性”，在合理的资源分配情况下，增加系统中处理请求操作的资源你尸体，进而提升系统能够处理的请求书，即吞吐率。

<img src="https://user-images.githubusercontent.com/19903677/113002322-3bddfc80-91a4-11eb-90e4-2562b78a96c4.png" alt="ab9a384d70578236800ceaa8e58ff45c" style="zoom:25%;" />



注意！如果没有良好的系统设计，实际得到的结果，其实是右图所展示的那样。我们开始增加线程数时，视同吞吐率会增加，但是在进一步增加线程时，系统吞吐率就增长迟缓了，甚至还会出现下降的情况？

为什么出现这种情况呢？一个关键的瓶颈在于。系统中通常会存在被多线程同时访问的共享资源，比如一个共享的数据结构，当有多个线程修改这个资源时，为了保证共享资源的正确性，就需要有额外的机制进行保证，那么就会带来额外的开销。

拿 redis 来说，redis 的 List 数据类型，提供出队（LPOP) 和入队（Lpush）操作，如果使用多线程 A 和 B 一个入队，并且长度+1，一个出队，并且长度-1，为了保证队列长度的正确性，需要 线程 A 和 B 的 Lpush 和 Lpop 串行执行，否则我们就会得到错误的长度结果。这就是**多线程编程模式面临的共享资源的并发访问控制问题**

<img src="https://user-images.githubusercontent.com/19903677/113003081-f79f2c00-91a4-11eb-8292-64bc3a895f18.png" alt="ab9a384d70578236800ceaa8e58ff45c" style="zoom:25%;" />

并发访问控制一直是多线程开发中的一个难点问题，如果没有精细的设计，如果只是简单的采用了一个粗粒度互斥锁，即使增加了线程，大部分线程也在等待获取访问共享资源的互斥锁，并行变串行，系统的吞吐率并没有随着线程的增加而增加。

而且采用多线程一般会引入保护共享资源的并发访问，这降低了代码的易调试性和可维护性。因此 redis 直接采用了单线程的模式

### 单线程 redis 为什么那么快？

一般来说，单线程的处理能力比多线程差很多，但是 redis 却能使用单线程模型达到每秒数十万级别的处理能力，这是为什么呢？

1. redis 操作大部分在内存上完成，在加上他采用了高效的数据结构，例如哈希表和跳表。
2. redis **采用了多路复用机制**，使其在网络 IO 操作中能并发处理大量的客户端请求，实现高吞吐率

首先我们要明白网络操作的基本 IO 模型和潜在的阻塞点，毕竟，redis 采用单线程进行 IO。如果线程被阻塞了，就无法进行多路复用了

#### 基本 IO 模型与阻塞点

一般的请求（get）会有以下过程

1. 监听客户团请求（bind/listen）
2. 和客户端建立连接（accept）
3. 从 socket 中读取请求（recv）
4. 解析客户端发送请求（parse）
5. 根据请求类型读取兼职数据（get）
6. 最后给客户端返回结果，即向 socket 中写回数据（send）

<img src="https://user-images.githubusercontent.com/19903677/113652488-75dc6080-96c6-11eb-8023-35c6f478f6c3.png" style="zoom:25%;" />

上图显示了这一个过程，其中，bind/listen、accept、recv、parse 和 send 属于网络 IO 处理，而 get 属于键值数据处理。

但是，在这里的网络 IO 操作中，有潜在的阻塞点，分别是 accept（）和 recv（）。当 redis 监听到一个客户端有链接请求，但一直未能成功建立起连接时，会阻塞在 accept（）函数这里，导致其他客户端无法和 redis 建立联系，类似的，当 redis 通过 recv（）从一个客户端读取数据时，如果数据一致没有到达，redis 也会一直阻塞在 recv（）。

这就导致 redis 整个线程阻塞，无法处理其他客户端请求，效率很低。那我们来寻早一个非阻塞模式，socket 网络模型。

#### 非阻塞模式

socket 网络模型的非阻塞模式设置，主要体现在三个关键的函数调用上，如果想要使用 socket 非阻塞模式，就必须要了解这三个函数的调用返回型和设置模式。

在 socket 模型中，不同操作调用后会返回不同的套接字类型。socket（）方法会返回主动套接字，然后调用 listen()方法，将主动套接字转化为监听套接字，最后调用 accept（）方法接收到达的客户端连接，返回已连接套接字。



<img src="https://user-images.githubusercontent.com/19903677/113711503-97196d00-9717-11eb-8eea-2dbf908f6abb.png" style="zoom:25%;" />

如上图，针对监听套接字，我们可以设置非阻塞模式：当 redis 调用 accept（）但是一直未有连接请求到达时，redis 线程可以返回处理其他操作，而不用一直等待。但是，你要注意的是，调用 accept（）时，已经存在监听套接字了。

虽然 redis 线程可以不用继续等待，但是总得有机制继续在监听套接字上等待后续连接请求，并在有请求时通知 reids。

类似的，我们也可以针对已连接套接字设置非阻塞模式：redis 调用 recv（）后，如果已连接套接字上一直没有数据到达，redis 线程同样可以返回处理其他操作。我们也需要有机制继续监听该已连接套接字，并在有数据到达时通知 redis。

这样才能保证 redis 线程，既不会像基本 IO 模型中一直在阻塞点等待，也不会导致 redis 无法处理实际到达的链接请求或者数据。那需要什么机制来监听呢？linux 中的 IO 多路复用机制！

#### 基于多路复用的高性能 I/O 模型

linux 中的 IO 多路复用机制是指一个线程处理多个 IO 流，叫 select/epoll 机制。简单来说，在 redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听套机制和已连接套接字。内核会一直监听这些套接字上的链接请求或者数据请求。一旦有请求到达，就会交给 redis 线程处理，这就实现了 redis 线程处理多个 IO 流的效果。

下图就是基于多路复用的 redis IO 模型。图中的多个 FD 就是刚才说的多个套接字。redis 网络框架调用 epoll 机制，让内核监听这些套接字。此时，reids 线程不会阻塞在某个特定的监听或者已连接套接字上，也就是说，不会阻塞在某一个特定的客户端请求上。正因为如此，redis 可以同事和多个客户端连接并处理请求，从而提升并发性。

<img src="https://user-images.githubusercontent.com/19903677/113713866-6129b800-971a-11eb-9bd2-17da886b7ed0.png" style="zoom:25%;" />

为了在请求到达时能通知到 redis 线程，select/epoll 提供了基于事件的毁掉机制，即针对不同事件的发生，调用相应的处理函数。

那么，回调机制是怎么工作的呢？其实，select/epoll 一旦检测到 FD 上游请求到达时，就会触发相应的事件

这些事件会放到一个事件队列。redis 单线程对改时间队列不断处理。这样一来，redis 无需一直轮询是否有请求实际发生，这就避免造成了 CPU 浪费，同时 redis 对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。因为 redis 一直在对事件队列进行处理，所以能及时相应客户端请求，提升 redis 的响应性能。

举个栗子，以连接请求和读数据请求：连接请求对应 accept 事件和 read 事件，redis 分别对这两个事件注册 accept 和 get 回调函数。当 linux 内核监听到有连接请求或者读数据请求时，就会触发 accept 事件和 read 事件，此时，内核就会回调 redis 相应的 accept 和 get 函数进行处理

