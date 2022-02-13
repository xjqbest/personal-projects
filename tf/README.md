

## tf.data

[https://tensorflow.google.cn/guide/data_performance](https://tensorflow.google.cn/guide/data_performance)

数据读取和解析实现在c++端。

优化如下：
 - prefetch: 数据预加载。（使用了pipeline的方式）
 - interleave + num_parallel_calls: 并行extract数据。（IO并行）
 - map + num_parallel_calls：并行transform数据。
 - fusion：多个dataset融合成一个。（map+batch，map+filter等）。
 - cache: 把数据cache在内存，多个epoch复用。
 - Vectorized（向量化）：map前先batch。




interleave例子如下，cycle_length表示有多少元素可以被并行处理，block_length表示处理的每个元素的输出，每次有多大。
```python
a = Dataset.range(1, 6)  # ==> [ 1, 2, 3, 4, 5 ]

# NOTE: New lines indicate "block" boundaries.
a.interleave(lambda x: Dataset.from_tensors(x).repeat(6),
            cycle_length=2, block_length=4)  # ==> [1, 1, 1, 1,
                                             #      2, 2, 2, 2,
                                             #      1, 1,
                                             #      2, 2,
                                             #      3, 3, 3, 3,
                                             #      4, 4, 4, 4,
                                             #      3, 3,
                                             #      4, 4,
                                             #      5, 5, 5, 5,
                                             #      5, 5]
```

使用interleave并行读文件，每个文件每次输出16个样本：
```python
# Preprocess 4 files concurrently, and interleave blocks of 16 records
# from each file.
filenames = ["/var/data/file1.txt", "/var/data/file2.txt",
             "/var/data/file3.txt", "/var/data/file4.txt"]
dataset = tf.data.Dataset.from_tensor_slices(filenames)
def parse_fn(filename):
  return tf.data.Dataset.range(10)
dataset = dataset.interleave(lambda x:
    tf.data.TextLineDataset(x).map(parse_fn, num_parallel_calls=1),
    cycle_length=4, block_length=16)
```

一个使用dataset的示例，可以看出可以将dataset做一些变换（map、interleave、shuffle、batch等）生成新的dataset

<img width="1120" alt="image" src="https://user-images.githubusercontent.com/12492564/153747212-d1e21fee-a956-4ec8-af62-d86cb7032842.png">

## BFCAllocator

为了减少与物理设备频繁的malloc操作，需要使用内存池/显存池。tensorflow设备内存管理模块实现了一个best-fit with coalescing算法（简称bfc算法）。

整个内存空间由一个按基址升序排列的Chunk双向链表来表示，它们的直接前趋和后继必须在地址连续的内存空间。

<img width="546" alt="image" src="https://user-images.githubusercontent.com/12492564/153749119-ef222cec-69f7-4b55-8764-cf9bee467da9.png">

chunk的操作：
 - 申请内存：根据chunk双链表找到一个合适的内存块，如果该内存块的大小是用户申请的大小的二倍以上，那么就将该内存块切分成两块，这就是split操作。返回其中一块给用户，并将该内存块标识为占用。spilt操作会新增一个chunk，所以需要修改chunk双链表以维持前驱和后继关系。（只有在双向链表中不能找到合适的chunk时，extend过程才会被调用。它的调用说明现有的存储池中已经没有可以满足需求的存储区了，需要向物理设备申请，并创建新的chunk，向物理设备申请存储空间时，如果因为一次申请的空间较大而失败，会将请求空间做0.9因子的衰退）。
 - 释放内存：先将该块标记为空闲。然后根据chunk数据结构中的信息找到其前驱和后继内存块。如果前驱和后继块中有空闲的块，那么将刚释放的块和空闲的块合并成一个更大的chunk（这就是merge操作，合并当前块和其前后的空闲块）。再修改双链表结构以维持前驱后继关系。这就做到了内存碎片的回收。

当chunk数量很大时，为了寻找一个合适的内存块而遍历有较大开销，于是设计了bin。

每个bin都有一个size属性，一个bin是一个拥有chunk size >= binsize的空闲chunk的集合。集合中的chunk按照chunk size的升序组织成单链表。bfc算法维护了一个bin的集合：bins。它由多个bin以及从属于每个bin的chunks组成。内存中所有的空闲chunk都由bins管理。

图中每一列表示一个bin，列首方格中的数字表示bin的size。bin size的大小都是256的2^n的倍。每个bin下面挂载了一系列的空闲chunk，每个chunk的chunk size都 >= 所属的bin的bin size，按照chunk size的升序挂载成单链表。

<img width="560" alt="image" src="https://user-images.githubusercontent.com/12492564/153749282-11d7cee3-af9a-421b-ae5b-6e2a00baf867.png">

bins有三个操作：
 - search：给定一个chunk size，从bins中找到大于等于该chunksize的最小的那个空闲chunk。如果bin以数组的形式组织，那么可以从index = chunk size /256 >>2 的那个bin开始查找。
 - insert：将一个空闲的chunk插入到一个bin所挂载的chunk链表中，同时需要维持chunk链表的升序关系。具体流程是直接将chunk插入到index = chunk size /256 >>2的那个bin中即可。
 - delete：将一个空闲的chunk从bins中移除。

### StreamExecutor

对一种并行编程模型的封装。

Stream：处于同一个Stream的操作必须按顺序执行，不同Stream之间的并无顺序关系。

StreamExecutor框架由三个层次组成，从上到下依次为Platform层（平台描述）、StreamExecutor Core层（执行引擎）和LibrarySupport层（基础库）：
 - Platform：指的是计算所使用设备平台的抽象，每种Device对应一种Platform。比如GPU对应的是CudaPlatform，而CPU对应的是HostPlatform等。
 - StreamExecutor Core：该层只向上层暴露StreamExecutor类，而涉及到具体实现的StreamExecutorInterface以及各种具体的实现将由StreamExecutor类统一控制。
 - Library：这一层提供的是各种底层加速库的接入（比如CuDNN、Eigen、mkl、CuBLAS）。Library层将这些基础库统一作为插件（Plugin）来管理。他们通过PluginRegister模块注册。

类图：

<img width="805" alt="image" src="https://user-images.githubusercontent.com/12492564/153757001-ca8c8dc0-25d2-4652-9fa2-a56ecfd489e2.png">

从op出发，一次调用：

<img width="802" alt="image" src="https://user-images.githubusercontent.com/12492564/153756981-74e60fea-1d49-41be-adec-da043e9197ea.png">


### Placement

Soft Placement机制：如果某个Node被显示指定精确放在某Device上，但系统中却没有该Device上的实现版本，那么为了保证程序可用，Soft Placement将发挥作用，它将忽略device type，在系统中按照Device优先级选取另一个可用的实现版本重新改写Placement。

三类特殊的Op类型，对他们可以做一些特殊处理：
 - Generator类Op：入度为0，出度为1的Op。
 - MetaData类Op：直接在Tensor的元数据MetaData上操作，不改变Tensor本身的内容，比如Reshape）。
 - Ref类或Resource类：例如Variable这种可能发生赋值的Op（或者叫左值）。比如Variable，对其assign等操作肯定直接在Variable所在之地执行即可。

Placer是TensorFlow中Placement相关的类。它在尽可能满足用户诉求的前提下，暗中纠正部分不合理的Placement：
 - 规则A：若某个Node是GeneratorNode，将其与Consumer与其放在同一个Device上可以防止无意义的跨Device拷贝。
 - 规则B：若某个Node是MetaDataNode，将其与Producer放在相同的Device上也可以防止无意义的跨Device拷贝。
 - 规则C：若某个Node的输入是Reference type或者是Reource type，那么尽量将其与输入放在同一个Colocation Group中。

Find-Union算法：并查集算法，Placer内最重要的算法。TensorFlow通过Find-Union算法高效地处理了Node的Colocation（打印GraphDef里面node的属性“loc:@xxxx”）问题。简单而言，逻辑上，多个具有相同Colocation Group的Node应该被“并”到同一个组中，从而“查”某个Node的Placement信息时，可以更快速地获取整组的信息。

<img width="731" alt="image" src="https://user-images.githubusercontent.com/12492564/153758859-af99aa67-67d3-40a4-9830-2241b4f726ef.png">

### session

两种线程数：

<img width="798" alt="image" src="https://user-images.githubusercontent.com/12492564/153760625-1d36ba1f-f16f-4e25-beb9-9d9407968dc9.png">

<img width="824" alt="image" src="https://user-images.githubusercontent.com/12492564/153760648-fd232e3f-4971-48e3-8995-9b3831c9b68a.png">

### 通信模块 Rendezvous

ParsedKey：消息传输的唯一标识符。在多组消息同时发送接收时，需要对每一对Send和Recv梳理一个对应关系，即Send端发送的消息与Recv端接收的消息不能有错位。只要让某个消息在被Send前加上一个唯一标识符，而Recv在接收消息前也能够按照某种规则拼出一样的唯一标识符，这个对应关系就解决了。

Table：消息队列的缓存。

（1）本地通信

<img width="665" alt="image" src="https://user-images.githubusercontent.com/12492564/153767377-882f69fa-4304-45e6-82b2-8c5011c95286.png">

value就是参与通信tensor本体。waitor是在确认tensor被接收端完成接收后的处理函数，也就是consumer处理该tensor的函数过程。

<img width="730" alt="image" src="https://user-images.githubusercontent.com/12492564/153767398-f8357be1-1e2e-4a69-b616-21ea9e453efa.png">

Send过程作为Tensor的生产者，它负责将待发送的Tensor送入Table中，并将ParsedKey作为该Item的键。而Recv过程作为消费者，它也会根据自己所需拼出相同的ParsedKey，然后从Table中查看是否已经存在该项。在Send和RecvAsync顺序相对异步的情况下，waitor函数的执行时机只有两种情况，它取决于Send的供给和RecvAsync的需求哪一个先到达。若生产者先到达，那么waiter函数的调用由RecvAsync执行。若消费者的需求先到达，那么waiter函数的调用由Send执行。

（2）跨进程通信

Send方——将Ready的Tensor挂入本地Table。（本地Tensor处于Ready状态后就被放挂了本地Worker的Table中，至此Send过程就全部完成了）

<img width="634" alt="image" src="https://user-images.githubusercontent.com/12492564/153767480-703053dc-d273-4687-8fc5-eeac8bb01bd2.png">

Recv方——向Send方主动发出请求，触发通信过程（主动的拉数据）。Recv方是Tensor的接收方，它的处理过程是：将所需要的Tensor对应的ParsedKey拼出后，主动向Send方主动发出Request，Send方在接收到Request后立即在本地Table中查找方所需要的Tensor，找到后将Tensor封装成Response发送回Recv方。

<img width="804" alt="image" src="https://user-images.githubusercontent.com/12492564/153767495-2d3951c5-a089-41e1-b295-30da100f2318.png">
