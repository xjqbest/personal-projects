

## tf.data

[https://www.tensorflow.org/guide/data_performance](https://www.tensorflow.org/guide/data_performance)

数据读取和解析实现在c++端。

优化如下：
 - prefetch: 数据预加载。（使用了pipeline的方式）
 - interleave: 并行预处理数据。（IO并行）
 - map：
 - cache: 把数据cache在内存




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
