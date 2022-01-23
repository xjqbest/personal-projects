
### grid和block的配置

 - block大小是32的整数倍：sm上线程执行的基本单元是warp，大小为32.
 - block不大不小：小可以有更多的block数避免sm空闲，大可以提高吞吐。
 - 根据资源约束来调整：每个sm上的最大block/warp数、最大shared memory、寄存器数。

### [reduction](https://developer.download.nvidia.cn/compute/cuda/1.1-Beta/x86_website/projects/reduction/doc/reduction.pdf)

 - 解决warp divergent:同一个warp中的线程尽量执行同一个指令
 - Shared Memory Bank Conflicts：为了获得高带宽，shared Memory被分成32（对应warp中的thread）个相等大小的内存块，他们可以被同时访问。如果warp访问shared Memory，对于每个bank只访问不多于一个内存地址，那么只需要一次内存传输就可以了，否则需要多次传输
 - 看看如何用起来空闲的thread

### \_\_host\_\_、\_\_global\_\_ 、\_\_device\_\_

\_\_host\_\_：由CPU调用，由CPU执行的函数
\_\_global\_\_：由CPU调用，launch到gpu上执行
\_\_device\_\_：由GPU中一个线程调用的函数

### unified memory

使用cudaMallocManaged来分配内存。

简化了代码编写和内存模型，可以在CPU端和GPU端共用一个指针，不用单独各自分配空间。方便管理，减少了代码量。

### 性能调优

 - 尽量减少分支语句
 - unrolling loops: 减少指令；增加更多的独立操作。
