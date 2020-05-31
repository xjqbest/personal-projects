
### 机器学习编程框架

ELF：map-reduce框架。对用户提供的主要抽象就是分布式数据流。用户同对一个数据源对象进行一些变换后得到一些新的数据流。最后通过用户将数据流的尾端点加入执行计划。并最终执行来得到结果。

提供了分布式kv存取的操作，但是不是像pull／push这种灵活操作，只能在数据流上做lookup和aggregate，像是加强版的reduce。

Hippo：容错、动态扩缩容。提供了通用的参数服务器。有调度模块、worker模块、server模块。它的目标是更快（数据按需分配），更省（动态扩缩容），更稳定（容错）。