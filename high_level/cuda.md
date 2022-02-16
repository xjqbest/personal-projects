
### grid和block的配置

 - block大小是32的整数倍：sm上线程执行的基本单元是warp，大小为32.（因为warp大小是32，避免inactive thread浪费sm资源）
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


#### warp相关的优化

 - 避免inactive的thread：如果block所含线程数目不是warp大小的整数倍，那么多出的那些thread所在的warp中，会剩余一些inactive的thread，即使这部分thread是inactive的，也会消耗SM资源，这点是编程时应避免的。
 -  Warp Divergence：因为所有同一个warp中的thread必须执行相同的指令，那么如果这些线程在遇到控制流语句时，如果进入不同的分支，那么同一时刻除了正在执行的分支外，其余分支都被阻塞了，十分影响性能。为了获得最好的性能，就需要避免同一个warp存在不同的执行路径。
 -  最大化active warp的数目：active warp可以被分为三类，SM中warp调度器每个cycle会挑选active warp送去执行，一个被选中的warp称为Selected warp，没被选中，但是已经做好准备被执行的称为Eligible warp，没准备好要被执行的称为Stalled warp。为了最大化GPU利用率，我们必须最大化active warp的数目。
 -  Latency Hiding：指令从开始到结束消耗的clock cycle称为指令的latency。当每个cycle都有eligible warp被调度时，计算资源就会得到充分利用。
 -  Bank Conflict：当多个地址请求落在同一个bank中就会发生bank conflict，从而导致请求多次执行。


### kernel性能调节

Achieved Occupancy = 每个SM在每个cycle能够达到的最大active warp数目占总warp的比例。

有更多的block，device一般会达到更多active warp，也就是更高的Occupancy。

Global Load Throughput：高load throughput有可能是一种假象，如果需要的数据在memory中存储格式未对齐不连续，会导致许多额外的不必要的load操作。

Global Memory Load Efficiency：是指我们确切需要的global load throughput与实际得到global load memory的比值。

一般来讲，最佳配置既不是拥有最高achieved Occupancy也不是最高load throughput的。所以不存在唯一metric来优化计算性能，我么需要从众多metric中寻求一个平衡。
