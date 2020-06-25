

### 一些概念

#### work stealing

mn线程库，要解决有些worker忙，有些worker闲。

整个的bthread调度顺序为：
 - local run queue
 - local remote queue
 - other workers’ run queue
 - other workers’remote queue
 
bthread会启动多个worker线程，每个worker线程对应一个TaskGroup。每个TaskGroup内有一个wait-free的runq和mutex protected remote_runq，其中remote_runq用于bthread外创建的task。

整个进程会有4个ParkingLot，用于worker间任务通知。当某个worker idle的时候，就wait在对应的ParkingLot上，其他worker有任务入队的时候，会唤醒一个ParkingLot上的waiter。之所以设置4个ParkingLot是为了避免一次入队唤醒多个等待的worker。

worker一直循环取task然后执行，有可能wait在某个parking lot上等待其他的唤醒。还有一种调度的情况是在bthread执行过程中调用bthread_yield让出当前的worker 。
 
通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。 
 
优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务。


#### cache bouncing

为了以较低的成本大幅提高性能，现代CPU都有cache。百度内常见的Intel E5-2620拥有32K的L1 dcache和icache，256K的L2 cache和15M的L3 cache。其中L1和L2cache为每个核独有，L3则所有核共享。为了保证所有的核看到正确的内存数据，一个核在写入自己的L1 cache后，CPU会执行Cache一致性算法把对应的cacheline(一般是64字节)同步到其他核。这个过程并不很快，是微秒级的，相比之下写入L1 cache只需要若干纳秒。当很多线程在频繁修改某个字段时，这个字段所在的cacheline被不停地同步到不同的核上，就像在核间弹来弹去，这个现象就叫做cache bouncing。

cache bouncing使访问频繁修改的变量的开销陡增，甚至还会使访问同一个cacheline中不常修改的变量也变慢，这个现象是false sharing。按cacheline对齐能避免false sharing，但在某些情况下，我们甚至还能避免修改“必须”修改的变量。bvar便是这样一个例子，当很多线程都在累加一个计数器时，我们让每个线程累加私有的变量而不参与全局竞争，在读取时我们累加所有线程的私有变量。虽然读比之前慢多了，但由于这类计数器的读多为低频展现，慢点无所谓。而写就快多了，从微秒到纳秒，几百倍的差距使得用户可以无所顾忌地使用bvar，这便是设计bvar的目的。

如何尽可能避免CPU的同步cache：多线程间尽量避免共享内存

#### wait-free & lock-free

（1）wait-free：不管OS如何调度线程，每个线程始终在做有用的事情。

（2）lock-free：不管OS如何调度线程，至少有一个线程在做有用的事情。

因此，如果有了锁，有可能拿到锁的线程去做IO，等待；其他线程又依赖这个锁，整个线程没有做有用的事情，因此有锁一定不是lock-free，更不可能是wait-free。


#### Streaming RPC

在一些应用场景中， client或server需要向对方发送大量数据，这些数据非常大或者持续地在产生以至于无法放在一个RPC的附件中。比如一个分布式系统的不同节点间传递replica或snapshot。client/server之间虽然可以通过多次RPC把数据切分后传输过去，但存在如下问题：
 - 如果这些RPC是并行的，无法保证接收端有序地收到数据，拼接数据的逻辑相当复杂。
 - 如果这些RPC是串行的，每次传递都得等待一次网络RTT+处理数据的延时，特别是后者的延时可能是难以预估的。

为了让大块数据以流水线的方式在client/server之间传递， 我们提供了Streaming RPC这种交互模型。Streaming RPC让用户能够在client/service之间建立用户态连接，称为Stream,  同一个TCP连接之上能同时存在多个Stream。 Stream的传输数据以消息为基本单位， 输入端可以源源不断的往Stream中写入消息， 接收端会按输入端写入顺序收到消息。


#### brpc连接方式

brpc支持以下连接方式：
 - 短连接：每次RPC前建立连接，结束后关闭连接。由于每次调用得有建立连接的开销，这种方式一般用于偶尔发起的操作，而不是持续发起请求的场景。没有协议默认使用这种连接方式，http/1.0对连接的处理效果类似短链接。
 - 连接池：每次RPC前取用空闲连接，结束后归还，一个连接上最多只有一个请求，一个client对一台server可能有多条连接。http/1.1和各类使用nshead的协议都是这个方式。
 - 单连接：进程内所有client与一台server最多只有一个连接，一个连接上可能同时有多个请求，回复返回顺序和请求顺序不需要一致，这是baidu_std，hulu_pbrpc，sofa_pbrpc协议的默认选项。
 
 | - |短链接|连接池|单连接|
 |  ----  | ----  | ----  | ----  |
 |长连接	|否|	是	|是|
 |server端连接数(单client)|	qps\*latency (原理见[little's law](https://en.wikipedia.org/wiki/Little%27s_law))|	qps\*latency	|1|
 |极限qps	|差，且受限于单机端口数|	中等	|高|
 |latency	1.5RTT(connect) + 1RTT + 处理时间|	1RTT + 处理时间|	1RTT + 处理时间|
 |cpu占用	|高, 每次都要tcp connect|	中等, 每个请求都要一次sys write|	低, 合并写出在大流量时减少cpu占用|

#### Cacheline

没有任何竞争或只被一个线程访问的原子操作是比较快的，“竞争”指的是多个线程同时访问同一个cacheline。现代CPU为了以低价格获得高性能，大量使用了cache，并把cache分了多级。百度内常见的Intel E5-2620拥有32K的L1 dcache和icache，256K的L2 cache和15M的L3 cache。其中L1和L2cache为每个核心独有，L3则所有核心共享。一个核心写入自己的L1 cache是极快的(4 cycles, 2 ns)，但当另一个核心读或写同一处内存时，它得确认看到其他核心中对应的cacheline。对于软件来说，这个过程是原子的，不能在中间穿插其他代码，只能等待CPU完成一致性同步，这个复杂的算法相比其他操作耗时会很长，在E5-2620上大约在700ns左右。所以访问被多个线程频繁共享的内存是比较慢的。

要提高性能，就要避免让CPU同步cacheline。这不仅仅和原子指令的性能有关，而是会影响到程序的整体性能。比如像一些临界区很小的场景，使用spinlock效果仍然不佳，问题就在于实现spinlock使用的exchange，fetch_add等指令必须在CPU同步好最新的cacheline后才能完成，看上去只有几条指令，花费若干微秒不奇怪。最有效的解决方法非常直白：尽量避免共享。从源头规避掉竞争是最好的，有竞争就要协调，而协调总是很难的。

 - 一个依赖全局多生产者多消费者队列(MPMC)的程序难有很好的多核扩展性，因为这个队列的极限吞吐取决于同步cache的延时，而不是核心的个数。最好是用多个SPMC或多个MPSC队列，甚至多个SPSC队列代替，在源头就规避掉竞争。
 - 另一个例子是全局计数器，如果所有线程都频繁修改一个全局变量，性能就会很差，原因同样在于不同的核心在不停地同步同一个cacheline。如果这个计数器只是用作打打日志之类的，那我们完全可以让每个线程修改thread-local变量，在需要时再合并所有线程中的值，性能可能有几十倍的差别。
 
做不到完全不共享，那就尽量少共享。在一些读很多的场景下，也许可以降低写的频率以减少同步cacheline的次数，以加快读的平均性能。一个相关的编程陷阱是避免false sharing：这指的是那些不怎么被修改的变量，由于同一个cacheline中的另一个变量被频繁修改，而不得不经常等待cacheline同步而显著变慢了。多线程中的变量尽量按访问规律排列，频繁被其他线程的修改要放在独立的cacheline中。要让一个变量或结构体按cacheline对齐。可以include <base/macros.h>然后使用BAIDU_CACHELINE_ALIGNMENT宏，用法请自行grep一下baidu-rpc的代码了解。

#### Memory fence

仅靠原子累加实现不了对资源的访问控制，即使简单如spinlock或引用计数，看上去正确的代码也可能会crash。这里的关键在于重排指令导致了读写一致性的变化。只要没有依赖，代码中在后面的指令（包括访存）就可能跑到前面去，编译器和CPU都会这么做。这么做的动机非常自然，CPU要尽量塞满每个cycle，在单位时间内运行尽量多的指令。一个核心访问自己独有的cache是很快的，所以它能很好地管理好一致性问题。当软件依次写入a,b,c后，它能以a,b,c的顺序依次读到，哪怕在CPU层面是完全并发运行的。当代码只运行于单线程中时，重排对软件是透明的。但在多核环境中，这就不成立了。如上节中提到的，访存在等待cacheline同步时要花费数百纳秒，最高效地自然是同时同步多个cacheline，而不是一个个做。一个线程在代码中对多个变量的依次修改，可能会以不同的次序同步到另一个线程所在的核心上，CPU也许永远无法保证这个顺序如同TCP那样，有序修改有序读取，因为不同线程对数据的需求顺序是不同的，按需访问是合理的（从而导致同步cacheline的序和写序不同）。如果其中第一个变量扮演了开关的作用，控制对后续变量对应资源的访问。那么当这些变量被一起同步到其他核心时，更新顺序可能变了，第一个变量未必是第一个更新的，其他线程可能还认为它代表着其他变量有效，而去访问了已经被删除的资源，从而导致未定义的行为。比如下面的代码片段：

线程1
```cpp
// ready was initialized to false
p.init();
ready = true;
```

线程2
```
if (ready) {
    p.bar();
}
```

从人的角度，这是对的，因为线程2在ready为true时才会访问p，按线程1的逻辑，此时p应该初始化好了。但对多核机器而言，这段代码难以正常运行：

 - 线程1中的ready = true可能会被编译器或cpu重排到p.init()之前，从而使线程2看到ready为true时，p仍然未初始化。
 - 即使没有重排，ready和p的值也会独立地同步到线程2所在核心的cache，线程2仍然可能在看到ready为true时看到未初始化的p。这种情况同样也会在线程2中发生，比如p.bar()中的一些代码被重排到检查ready之前。

通过这个简单例子，你可以窥见原子指令编程的复杂性了吧。为了解决这个问题，CPU提供了memory fence，让用户可以声明访存指令间的可见性(visibility)关系，boost和C++11对memory fencing做了抽象，总结为如下几种memory order.

 - memory_order_relaxed	没有fencing作用
 - memory_order_consume	后面依赖此原子变量的访存指令勿重排至此条指令之前
 - memory_order_acquire	后面访存指令勿重排至此条指令之前
 - memory_order_release	前面访存指令勿重排至此条指令之后。当此条指令的结果对其他线程可见后，之前的所有指令都可见
 - memory_order_acq_rel	acquire + release语意
 - memory_order_seq_cst	acq_rel语意外加所有使用seq_cst的指令有严格地全序关系
 
有了memory order，上面的例子可以这么更正：

线程1
```cpp
// ready was initialized to false
p.init();
ready.store(true, std::memory_order_release);
```

线程2
```cpp
if (ready.load(std::memory_order_acquire)) {
    p.bar();
}
```

线程2中的acquire和线程1的release配对，确保线程2在看到ready==true时能看到线程1 release之前所有的访存操作。

#### IO

一般有三种操作IO的方式：
 - blocking IO: 发起IO操作后阻塞当前线程直到IO结束，标准的同步IO，如默认行为的posix read和write。
 - non-blocking IO: 发起IO操作后不阻塞，用户可阻塞等待多个IO操作同时结束。non-blocking也是一种同步IO：“批量的同步”。如linux下的poll,select, [epoll](https://blog.csdn.net/songchuwang1868/article/details/89877739)，BSD下的kqueue。
 - asynchronous IO: 发起IO操作后不阻塞，用户得递一个回调待IO结束后被调用。如windows下的OVERLAPPED + IOCP。linux的native AIO只对文件有效。
linux一般使用non-blocking IO提高IO并发度。当IO并发度很低时，non-blocking IO不一定比blocking IO更高效，因为后者完全由内核负责，而read/write这类系统调用已高度优化，效率显然高于一般得多个线程协作的non-blocking IO。但当IO并发度愈发提高时，blocking IO阻塞一个线程的弊端便显露出来：内核得不停地在线程间切换才能完成有效的工作，一个cpu core上可能只做了一点点事情，就马上又换成了另一个线程，cpu cache没得到充分利用，另外大量的线程会使得依赖thread-local加速的代码性能明显下降，如tcmalloc，一旦malloc变慢，程序整体性能往往也会随之下降。而non-blocking IO一般由少量event dispatching线程和一些运行用户逻辑的worker线程组成，这些线程往往会被复用（换句话说调度工作转移到了用户态），event dispatching和worker可以同时在不同的核运行（流水线化），内核不用频繁的切换就能完成有效的工作。线程总量也不用很多，所以对thread-local的使用也比较充分。这时候non-blocking IO就往往比blocking IO快了。不过non-blocking IO也有自己的问题，它需要调用更多系统调用，比如epoll_ctl，由于epoll实现为一棵红黑树，epoll_ctl并不是一个很快的操作，特别在多核环境下，依赖epoll_ctl的实现往往会面临棘手的扩展性问题。non-blocking需要更大的缓冲，否则就会触发更多的事件而影响效率。non-blocking还得解决不少多线程问题，代码比blocking复杂很多。
