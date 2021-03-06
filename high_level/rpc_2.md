
### 线程模型

[https://github.com/apache/incubator-brpc/blob/master/docs/cn/threading_overview.md](https://github.com/apache/incubator-brpc/blob/master/docs/cn/threading_overview.md)

 - 一个连接对应一个线程或进程：线程/进程处理来自绑定连接的消息，连接不断开线程/进程就不退。当连接数逐渐增多时，线程/进程占用的资源和上下文切换成本会越来越大，性能很差。
 - 单线程reactor：由一个event dispatcher等待各类事件，待事件发生后原地调用event handler，全部调用完后等待更多事件，故为"loop"。实质是把多段逻辑按事件触发顺序交织在一个系统线程中。一个event-loop只能使用一个核，故此类程序要么是IO-bound，要么是逻辑有确定的较短的运行时间（比如http server)，否则一个回调卡住就会卡住整个程序，容易产生高延时。
 - N:1线程库：一般是把N个用户线程映射入一个系统线程(LWP)，同时只能运行一个用户线程，调用阻塞函数时才会放弃时间片。N:1线程库与单线程reactor等价，只是事件回调被替换为了独立的栈和寄存器状态，运行回调变成了跳转至对应的上下文。由于所有的逻辑运行在一个系统线程中，N:1线程库不太会产生复杂的race condition，一些编码场景不需要锁。和event loop库一样，由于只能利用一个核，N:1线程库无法充分发挥多核性能。
 - 多线程reactor：一般由一个或多个线程分别运行event dispatcher，待事件发生后把event handler交给一个worker thread执行。由于百度内以SMP机器为主，这种可以利用多核的结构更加合适，多线程交换信息的方式也比多进程更多更简单，所以往往能让多核的负载更加均匀。不过由于cache一致性的限制，多线程reactor模型并不能获得线性于核数的扩展性。
 - M:N线程库：把M个用户线程映射入N个系统线程(LWP)。
 
 多线程reactor的cache bouncing：  
 当event dispatcher把任务递给worker时，用户逻辑不得不从一个核跳到另一个核，相关的cpu cache必须同步过来，这是微秒级的操作，并不很快。如果worker能直接在event dispatcher所在的核上运行就更好了，因为大部分系统（在这个时间尺度下）并没有密集的事件流，尽快运行已有的任务的优先级高于event dispatcher获取新事件。另一个例子是收到response后最好在当前cpu core唤醒发起request的阻塞线程。
 
 M:N线程库优点：  
 - 每个系统线程往往有独立的runqueue，可能有一个或多个scheduler把用户线程分发到不同的runqueue，每个系统线程会优先运行自己runqueue中的用户线程，然后再做全局调度。这当然更复杂，但比全局mutex + condition有更好的扩展性。
 - 虽然M:N线程库和多线程reactor是等价的，但同步的编码难度显著地低于事件驱动，大部分人都能很快掌握同步操作。
 - 不用把一个函数拆成若干个回调，可以使用RAII。
 - 从用户线程A切换为用户线程B时，也许我们可以让B在A所在的核上运行，而让A去其他核运行，从而使更高优先级的B更少受到cache miss的干扰。
 
 
 ### 负载均衡
 
最常见的分流算法是round robin和随机。这两个方法的前提是下游的机器和网络都是类似的，但在目前的线上环境下，特别是混部的产品线中，已经很难成立，因为：
 - 每台机器运行着不同的程序组合，并伴随着一些离线任务，机器的可用资源在持续动态地变化着。
 - 机器配置不同。
 - 网络延时不同。 
 
LoadBalancer是一个读远多于写的数据结构：大部分时候，所有线程从一个不变的server列表中选取一台server。如果server列表真是“不变的”，那么选取server的过程就不用加锁，我们可以写更复杂的分流算法。一个方法是用读写锁，但当读临界区不是特别大时（毫秒级），读写锁并不比mutex快，而实用的分流算法不可能到毫秒级，否则开销也太大了。另一个方法是双缓冲，很多检索端用类似的方法实现无锁的查找过程，它大概这么工作：

 - 数据分前台和后台。
 - 检索线程只读前台，不用加锁。
 - 只有一个写线程：修改后台数据，切换前后台，睡眠一段时间，以确保老前台（新后台）不再被检索线程访问。

这个方法的问题在于它假定睡眠一段时间后就能避免和前台读线程发生竞争，这个时间一般是若干秒。由于多次写之间有间隔，这儿的写往往是批量写入，睡眠时正好用于积累数据增量。

但这套机制对“server列表”不太好用：总不能插入一个server就得等几秒钟才能插入下一个吧，即使我们用批量插入，这个"冷却"间隔多少会让用户觉得疑惑：短了担心安全性，长了觉得没有必要。我们能尽量降低这个时间并使其安全么？

我们需要写以某种形式和读同步，但读之间相互没竞争。一种解法是，读拿一把thread-local锁，写需要拿到所有的thread-local锁。具体过程如下：

 - 数据分前台和后台。
 - 读拿到自己所在线程的thread-local锁，执行查询逻辑后释放锁。
 - 同时只有一个写：修改后台数据，切换前后台，挨个获得所有thread-local锁并立刻释放，结束后再改一遍新后台（老前台）。

我们来分析下这个方法的基本原理：

 - 当一个读正在发生时，它会拿着所在线程的thread-local锁，这把锁会挡住同时进行的写，从而保证前台数据不会被修改。
 - 在大部分时候thread-local锁都没有竞争，对性能影响很小。
 - 逐个获取thread-local锁并立刻释放是为了确保对应的读线程看到了切换后的新前台。如果所有的读线程都看到了新前台，写线程便可以安全地修改老前台（新后台）了。

其他特点：

 - 不同的读之间没有竞争，高度并发。
 - 如果没有写，读总是能无竞争地获取和释放thread-local锁，一般小于25ns，对延时基本无影响。如果有写，由于其临界区极小（拿到立刻释放），读在大部分时候仍能快速地获得锁，少数时候释放锁时可能有唤醒写线程的代价。由于写本身就是少数情况，读整体上几乎不会碰到竞争锁。
 - 完成这些功能的数据结构是DoublyBufferedData<>，我们常简称为DBD。baidu-rpc中的所有load balancer都使用了这个数据结构，使不同线程在分流时几乎不会互斥。而其他rpc实现往往使用了全局锁，这使得它们无法写出复杂的分流算法：否则分流代码将会成为竞争热点。

这个结构有广泛的应用场景：
 - reload词典。大部分时候词典都是只读的，不同线程同时查询时不应查询。
 - 可替换的全局callback。像base/logging.cpp支持配置全局LogSink以重定向日志，这个LogSink就是一个带状态的callback。如果只是简单的全局变量，在替换后我们无法直接删除LogSink，因为可能还有都写线程在用。用DBD可以解决这个问题。
 
 
### 一致性哈希

一些场景希望同样的请求尽量落到一台机器上，比如访问缓存集群时，我们往往希望同一种请求能落到同一个后端上，以充分利用其上已有的缓存，不同的机器承载不同的稳定working set。
而不是随机地散落到所有机器上，那样的话会迫使所有机器缓存所有的内容，最终由于存不下形成颠簸而表现糟糕。 我们都知道hash能满足这个要求，比如当有n台服务器时，输入x总是会发送到第hash(x) % n台服务器上。
但当服务器变为m台时，hash(x) % n和hash(x) % m很可能都不相等，这会使得几乎所有请求的发送目的地都发生变化，如果目的地是缓存服务，所有缓存将失效，继而对原本被缓存遮挡的数据库或计算服务造成请求风暴，
触发雪崩。一致性哈希是一种特殊的哈希算法，在增加服务器时，发向每个老节点的请求中只会有一部分转向新节点，从而实现平滑的迁移。这篇论文中提出了一致性hash的概念。

一致性hash满足以下四个性质：

 - 平衡性 (Balance) : 每个节点被选到的概率是O(1/n)。
 - 单调性 (Monotonicity) : 当新节点加入时， 不会有请求在老节点间移动， 只会从老节点移动到新节点。当有节点被删除时，也不会影响落在别的节点上的请求。
 - 分散性 (Spread) : 当上游的机器看到不同的下游列表时(在上线时及不稳定的网络中比较常见),  同一个请求尽量映射到少量的节点中。
 - 负载 (Load) : 当上游的机器看到不同的下游列表的时候， 保证每台下游分到的请求数量尽量一致。
 
所有server的32位hash值在32位整数值域上构成一个环(Hash Ring)，环上的每个区间和一个server唯一对应，如果一个key落在某个区间内， 它就被分流到对应的server上。 


当删除一个server的， 它对应的区间会归属于相邻的server，所有的请求都会跑过去。当增加一个server时，它会分割某个server的区间并承载落在这个区间上的所有请求。单纯使用Hash Ring很难满足我们上节提到的属性，主要两个问题：

 - 在机器数量较少的时候， 区间大小会不平衡。
 - 当一台机器故障的时候， 它的压力会完全转移到另外一台机器， 可能无法承载。

为了解决这个问题，我们为每个server计算m个hash值，从而把32位整数值域划分为n*m个区间，当key落到某个区间时，分流到对应的server上。那些额外的hash值使得区间划分更加均匀，被称为Virtual Node。当删除一个server时，它对应的m个区间会分别合入相邻的区间中，那个server上的请求会较为平均地转移到其他server上。当增加server时，它会分割m个现有区间，从对应server上分别转移一些请求过来。

由于节点故障和变化不常发生， 我们选择了修改复杂度为O(n)的有序数组来存储hash ring，每次分流使用二分查找来选择对应的机器， 由于存储是连续的，查找效率比基于平衡二叉树的实现高。 线程安全性请参照Double Buffered Data章节.

### Memory Management

内存管理总是程序中的重要一环，在多线程时代，一个好的内存分配大都在如下两点间权衡：

 - 线程间竞争少。内存分配的粒度大都比较小，对性能敏感，如果不同的线程在大多数分配时会竞争同一份资源或同一把锁，性能将会非常糟糕，原因无外乎和cache一致性有关，已被大量的malloc方案证明。
 - 浪费的空间少。如果每个线程各申请各的，速度也许不错，但万一一个线程总是申请，另一个线程总是释放，这个方案就显然不靠谱了。所以线程之间总是要共享全局数据的，如何共享就是方案的关键了。

一般用户可以使用tcmalloc、jemalloc等成熟的内存分配方案，但这对于较为底层，关注性能长尾的应用是不够的。多线程框架广泛地通过传递对象的ownership来让问题异步化，如何让分配这些小对象的开销变的更小是值得研究的问题。其中的一个特点较为显著：大多数结构是等长的。这个属性可以大幅简化内存分配的过程，获得比通用malloc更加稳定、快速的性能。baidu-rpc中的ResourcePool<T>和ObjectPool<T>即提供这类分配。
