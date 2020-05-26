## 参考资料

[https://github.com/steveLauwh/SGI-STL](https://github.com/steveLauwh/SGI-STL)

[https://github.com/karottc/sgi-stl](https://github.com/karottc/sgi-stl)


### vector 的数据结构

```cpp
template <class T, class Alloc=alloc>
class vector {
public:
  typedef T value_type;
  typedef value_type* iterator; // vector 的迭代器是普通指针
protected:
  iterator start;  // 表示目前使用空间的头
  iterator finish; // 表示目前使用空间的尾
  iterator end_of_storage; // 表示目前可用空间的尾
};
```

#### 配置器(allocator)

配置器：负责空间管理、申请释放

SGI STL 的配置器，其名称是 alloc 而不是 allocator，而且不接受任何参数。
每一个容器都已经指定其缺省的空间配置器为 alloc。

类的构造析构与内存分配释放是独立的。


最朴素的空间申请、释放：
```cpp
template <class T>
inline T* allocate(ptrdiff_t size, T*) {
    // 申请size个T类型大小的空间
    T* tmp = (T*)(::operator new((size_t)(size * sizeof(T))));
    if (tmp == 0) {
	  cerr << "out of memory" << endl; 
	  exit(1);
    }
    return tmp;
}
template <class T>
inline void deallocate(T* buffer) {
    ::operator delete(buffer);
}
```

实际的实现会考虑两个问题：
 - 内存不足的情况：循环不断尝试分配，直到成功。用户还可以传入一个handler自定义行为，比如抛异常。
 - 内存碎片：两级配置器，第一级配置器直接使用 malloc() 和 free() 实现；第二级配置器使用 memory pool 内存池管理。

这里顺便复习一下new：

new 操作符的执行过程：
 - 调用operator new分配内存
 - 调用构造函数在operator new返回的内存地址处生成类对象

operator new是一个函数，用来分配内存，类似于malloc

placement new 是c++中对operator new 的重载版本，并不分配内存，而是在一个已经分配好的内存中（栈或者堆中）构造一个新的对象。
示例用法：Foo\* p = new (buff)Foo; 其中buff是实现调用operator new 分配好的内存。


这个是类的构造与析构：
```cpp
// 将初值 __value 设定到指针所指的空间上。
template <class _T1, class _T2>
inline void _Construct(_T1* __p, const _T2& __value) {
  new ((void*) __p) _T1(__value);   // placement new，调用 _T1::_T1(__value);
}

template <class _T1>
inline void _Construct(_T1* __p) {
  new ((void*) __p) _T1();
}

// 接受一个指针，将该指针所指的对象析构掉。
template <class _Tp>
inline void _Destroy(_Tp* __pointer) {
  __pointer->~_Tp();
}
```
