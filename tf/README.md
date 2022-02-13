

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





