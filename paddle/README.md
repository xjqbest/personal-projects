

### fleet shuffle排查（效果）

数据量 -> 前向反向计算 -> 梯度更新、feasign数、show／click -> 对比shuffle方式

<img src="./images/shuffle.png" width="70%" height="70%">


### fleet 内存

每个样本是 [slot:feasign]\*的格式

原始数据结构
```cpp
struct slot {
  std::vector<float> float_feasign_;
  std::vector<uint64_t> uint64_feasign_;
  std::vector<size_t> offset_;
};

// 一个样本，假如slot个数是408，那么就是
std::vector<slot> sample(408);
```
现在假设每条样本都是空的，一共100w个样本，由于每个样本

### fleet 速度优化


### fleet dataset流程

<img src="./images/dataset.png" width="70%" height="70%">
