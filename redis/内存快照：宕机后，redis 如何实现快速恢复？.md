## 内存快照：宕机后，redis 如何实现快速恢复？

​	我们都知道redis的持久化有AOF方法，这个方法的好处就是每次执行只需要记录操作命令，需要持久化的数据量不大。一般来说只要咱们不使用always持久化策略，就不会对性能造成太大影响

​	但是就是因为记录的是操作命令，而不是实际的命令，所以使用AOF方法进行故障回复的时候，需要逐一把操作日志都执行以便。如果操作日志非常多，redis就会恢复的很缓慢，影响到正常使用，这当然不是理想的结果，那么有没有既保证可靠性，还能在宕机时实现快速回复的其他方法呢？那就是 『内存快照』，快照就是把内存中的数据在某一时刻的状态记录。就相当于照片，把某一时刻记录下来。

​	对redis来说，他实现类似照片效果的方式，就是把某一时刻的状态以文件的时形式写到磁盘上，也就是快照，这样一来，即使宕机，快照文件也不会丢失，数据的可靠性也就得到了保证。这个快照文件就成为RDB文件，期中RDB就是redis daabase 的缩写。

​	和AOF相比，RDB记录的是某一时刻的数据，并不是操作。所以，在数据恢复时，我们可以直接把RDB文件读入内存，很快的完成恢复，听起来很不错，但是有两个问题：

- 对哪些数据做快照？这关系到快照的执行效率问题
- 做快照时，数据还能被增删改吗？这关系到redis是否被阻塞，能否同事正常处理请求

举个例子，我们在拍照的时候，通常关注两个问题

- 如何取景？也就是说我们要把哪些人、哪些物拍到照片中；
- 在按快门前，要提醒朋友不要乱动，否则拍出来的照片就模糊了

你看，这两个问题是不是非常重要呢？那么，接下来，我们就具体来聊一聊。先说『取景的问题』，也就是我们对哪些数据做快照。

### 给哪些数据做快照

redis的数据都存储在内纯当中，为了提供所有数据的可靠性保证，它执行的是全量快照，也就是说，把内存中所有的数据都记录到磁盘，这就类似于给100个人拍何莹，把每个人都拍进照片里，这样做的好处是，一次性记录了所有的数据，一个都不能少。

当你一个人拍照的时候，只用协调一个人就够了，但是，拍100人的大合影，却需要协调100个人的位置、状态等等，遮挡人会更费时费力。同样给内存的全量数据做快照，把他们全都写入磁盘也会话费很多时间，而且，全量数据越多，RDB 文件就越大。往磁盘上谢数据的时间开销就越大

对于 redis 而言，他的单线程模型就决定了我们要尽量避免所有会阻塞主线程的操作，所以。针对任何操作，我们都会提一个灵魂之问：『它会阻塞主线程吗』RDB 文件的生成是否会阻塞主线程，这就关系到是否会降低 redis 的性能

redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave

- save：在主线程执行，会导致阻塞；
- bgsave：创建一个子进程，专门用户写入 RDB 文件，避免了主线程的阻塞，这也是 redis RDB 文件生成的默认配置

我们肯定会选择 bgsave 来执行全量快照，这既提供了数据的可靠性保证，也避免了对 redis 性能的影响



接下来，我们要关注的问题就是，在对内存数据做快照时候，这些数据还能懂吗？也就是说，这些数据还能够被修改吗？这个问题非常重要，因为，如果数据被修改，那就意味着 redis 还能正常处理写操作，否则，所有写操作都得等到快照做完了才能执行，性能一下子就降低了



### 快照时数据能修改吗？

在给别人拍照时，一旦对方动了，那么这张照片就拍糊了，我们就需要重拍，所以我们当然希望数据保持不动。



举个例子，我们在给时刻 T 的内存做快照，假设内存数据是4GB，磁盘写入的带宽是0.2GB/s，简单来说，至少需要20s（4/0.2 = 20）才能做完，如果在 t+5s 时，一个还没有写入磁盘的内存数据 A，被修改成了 A' ，那么就会破坏快照的完整性，因为 A'不是时刻 T 的状态，因此，和拍照类似，我们在做快照的时候也不希望数据『动』，也就是不能被修改。



但是，如果快照执行期间数据不能被修改，是会有潜在问题点。对于刚刚的例子来说，在做快照的20s 时间里。如果这4GB 的数据都不能被修改，redis 就不能处理这些数据的写操作，那无疑会给业务服务造成巨大的影响。



你可能会想到，可以用 bgsave 避免阻塞啊，这里我要说一下常见的误区了，避免阻塞和正常处理写操作不是一回事。此事，主线程的确没有足额色，可以正常接收请求，但是，为了保证快照完整性，他只能处理读操作，因为不能修改正在执行的快照数据。



为了快照而暂停写操作，肯定是不能接受的。所以这个时候，redis 就会借助操作系统提供的复制技术（copy-on-write，COW），在执行快照的同事，正常处理写操作



简单来说，bgsave 子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据，bgsave 子进程运行后，开始读取主线程的内存数据，并把他们写入 RDB 文件。



此时，如果主线程对这些数据也都是读操作（例如下图中的键值对 A），那么，这块数据就会被复制一份，生成改数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，而在这个过程中，主线程仍然可以直接修改原来的数据

<img src="https://user-images.githubusercontent.com/19903677/115989862-05ed3600-a5f3-11eb-80c9-dfb8d9bd6c89.png" style="zoom:55%;" />



这既保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响。



到这里，我们就解决了对『对哪些数据做快照』以及『做快照数据能否修改』这两大问题：redis 会使用 bgsave 对当前内存中的所有数据做快照，这个操作是子进程在后台完成的，这就允许主线程同时可以修改数据



### 多久做一次快照？

#### 可以每秒做一次快照吗？

我们都知道照相有个技术叫做『连拍』，所谓的『连拍』就是连续的做快照。这样一来，快照的间隔时间变得很短，即使某一时刻发生宕机，因为上一时刻快照刚执行，丢失的数据也不会太多，但是，这其中的快照间隔时间久很关键了



如下图，我们现在 T0时刻做了一个快照，然后就在 T0+t时刻做了一次快照，在这期间，数据块 5和 9 被修改了，如果按照 T0时刻的快照进行修复，此时，数据块 5 和 9 的修改至因为没有快照记录，就无法恢复了

<img src="https://user-images.githubusercontent.com/19903677/115990068-e60a4200-a5f3-11eb-8e73-cbe2ce19bfc0.png" style="zoom:55%;" />

所以，想要尽可能的回复数据，t 值就要尽可能的小，t 越小。就越想『连拍』。那么 t 值可以小到什么程度呢？譬如是不是可以每秒做一次快照？毕竟，每次快照都是由 bgsave 子进程在后台执行，也不会阻塞主线程。



这种想法其实是错误的，虽然 bgsave 执行时不会阻塞主线程，但是，如此频繁的执行全量快照，也会带量两方面的开销。

一方面，频繁的将全量的数据写入磁盘，会给磁盘带来很大的压力，多个快照竞争有限的磁盘宽带，前一个快照还没做完，后一个又开始做了，很容易造成恶性循环

另一方面，bgsave 子进程需要通过 fork 操作从主线程创建出来，虽然，子进程在创建之后不会在阻塞主线程，但是，fork 这个创建本身会阻塞主线程，而且主线程的内存越大，阻塞的时间越长。如果频繁的 fork 初 bgsave 子进程，这就回频繁阻塞主线程了，那么有什么其他好方法吗？

此时，我们可以做增量快照，所谓增量快照，就是值：做了一次全量快照后，后续的快照只对修改的数据进行快照记录，这样可以避免每次全量快照的开销。



在第一次做完全量快照之后，T1和 T2时刻如果在做快照，我们只需要将被修改的数据写入快照文件就行了，但是，这么多的前提是，我们需要记住哪些数据被修改了。不要小瞧『记住』这个功能，他需要我们使用额外的元数据信息去记录哪些数据被修改了，这会带来额外的空间开销问题。如下图

<img src="https://user-images.githubusercontent.com/19903677/115990280-e9ea9400-a5f4-11eb-9c68-e4f32373572c.png" style="zoom:55%;" />

如果我们对每一个键值对的修改，都做个记录，那么，如果有1W 个被修改的键值对，我们就需要有1W条额外的记录，而且，有时候，键值对非常小，比如只有32字节，而记录他被修改的元数据信息，可能需要8字节，这样的话，为了『记住』修改，引入额外的空间开销比较大。这对于内存资源宝贵的 redis 来说，有些得不偿失。

到这里，其实可以发现，虽然跟 AOF 相比，快照恢复速度快，但是，快照的频率不好把握，如果 频率太低，两次快照间一旦党籍，就可能有比较多的数据丢失，如果频率太高，又会产生额外的开销，那么还有什么办法既能利用 RDB 快照的快速回复，又能以较小的开销做到尽量少丢数据呢？



redis 4.0提出了一个混合使用 AOF 日志和内存快照的方法，简单来说，内存快照以一定的频率执行，在两次快照之间，使用 aof 日志记录这期间的所有命令操作。

这样一来，快照不用很频繁的执行，这就避免了频繁 fork 对主线程的影响，而且，AOF 日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。

如下图：

T1和 T2时刻的修改，用 AOF 日志记录，等到第二次全量快照时，就可以青口梅那个 AOF 日志，因为此事的修改都已经记录到了快照中了，恢复时就不再用日志了。

<img src="https://user-images.githubusercontent.com/19903677/115990431-cecc5400-a5f5-11eb-9c1c-c70b043d9350.png" style="zoom:55%;" />



这个方法即能享受到 RDB 文件快速回复的好处，又能享受到 AOF 只记录操作命令的简单优势，有点『鱼和熊掌可以兼得』的感觉



### 总结：

内存快照可以快速的恢复内存中的数据，就是把 RDB 文件直接读入内存，这就避免了 AOF 需要顺序、逐一重新执行命令带来的抵消性能问题。

不过内存快照也有他的局限性。它拍的是内存的『大合影』，不 可避免的会耗时耗力。虽然，redis 涉及了 bgsave 和写时复制方式，尽可能减少了内存快照对正常读写的影响，但是，频繁的快照仍然不太能接受的，而混合使用 RDB 和 AOF，正好可以取两者职场，避两者之短，以较小的性能开销保证数据可靠性和性能

最后，

- 数据不能丢失时，内存快照和 AOF 的混合使用是一个很好的选择
- 如果允许分钟级别的数据丢失，可以只使用 RDB
- 如果只用 AOF，有限使用 everysec 的配置选项，因为他在可靠性和性能之间取了平衡
