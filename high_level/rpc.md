

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

#### wait-free & lock-free


