
#### 面试问题

 - aibox相关问题
 - hdfs读取慢：通过读其他副本。慢的也可以创建多个task，取最先完成的。
 - 特征准入优化：bloom filter
 
 <img src="./images/1.png" width="80%" height="80%">

以及counting bloom filter，统计映射到某k个位置的数量。insert时候++，delete时候--

#### 编程题

 - 给定一个整数数组，返回最长递增子数组的长度。（动态规划）

```cpp
// f[n] = max_{0}_{n - 1}(f[i] + 1, if array[n] > array[i])
int get_max(const vector<int>& array) {
    if(array.size() <= 1) {
        return array.size();
    }
    vector<int> f(array.size(), 1);
    int ans = 1;
    for(int i = 1; i < array.size(); ++i) {
        for(int j = 0; j < i; ++j) {
            if (array[j] < array[i]) {
                f[i] = max(f[i], f[j] + 1);
                ans = max(ans, f[i]);
            }
        }
    }
    return ans;
}
```
