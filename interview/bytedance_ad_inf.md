



#### 编程题

 - 一面：手写kmeans
 - 二面：[01 矩阵](https://leetcode-cn.com/problems/01-matrix/) 、[二叉树直径](https://leetcode-cn.com/problems/diameter-of-binary-tree/)
 - 三面：[多数元素](https://leetcode-cn.com/problems/majority-element/)、[搜索旋转排序数组 II](https://leetcode-cn.com/problems/search-in-rotated-sorted-array-ii/)

#### 面试问题

 - 哈希表有那些优化，如何降低cache miss
 - 数据模块的实现
 - 分布式训练性能优化(内存、性能)
 - ctr推荐算法（wide&deep、din、dien）
 - itemcf改进：惩罚热点、加权冷启动
 - knn实现

##### 哈希表有那些优化，如何降低cache miss

一个HashMap有两个参数会影响它的性能：初始化大小（capacity）和装载因子（load factor）。

当size > (capacity * loadFactor) 的时候会出现rehash，把原来的所有的元素重新hash到新的数组中。

open addressing碰撞时要继续寻找下一个槽，设装载因子为a，那么期望的探查次数为1/(1-a)。假设element_size = cache_line size / N, 那么对于open addressing liner probing，每N次访问会造成一次cache miss。（比较理想的情况了）

链表的方法在访问链表元素的时候，也会有cache miss的问题。


如何缓解：

 - 保持装载因子不能过大，否则resize。
 - 初始化大小不能过小，否则会频繁resize
 - 元素/类大小做内存对齐（alignas）
 - 环形链表：比如说一个slot，下的value，一般拉链是 1-2-3，如果3特别热，每次查找3都得跳三次，如果1-2-3弄成环，发现3是热点，直接把head指向3



