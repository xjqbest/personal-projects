

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
