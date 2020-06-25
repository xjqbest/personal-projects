



### 读写锁

[https://www.cnblogs.com/XNQC1314/p/9183487.html](https://www.cnblogs.com/XNQC1314/p/9183487.html)

[https://www.cnblogs.com/sylz/p/6030675.html](https://www.cnblogs.com/sylz/p/6030675.html)



### 无锁stack

实现无锁数据结构的基础是CAS（Compare And Set）。比如atomic_compare_exchange的行为类似于原子的执行如下代码
```cpp
if (memcmp(obj, expected, sizeof *obj) == 0)
    memcpy(obj, &desired, sizeof *obj);
else
    memcpy(expected, obj, sizeof *obj);
```

```cpp
// 将obj所指向的值与expected所指向的值进行原子比较，如果相等，则把*desired赋值给*obj。否则，把*obj赋值给*expected。
bool atomic_compare_exchange_weak（volatile A * obj，C * expected，C desired);
```

lockfree优缺点：

 - 优点：不发生死锁。性能通常比lock好一些。
 - 缺点：仍然是锁（只是将锁限制在一个最小的范围内，通常是一个原子操作）。所以并不能随cpu个数增加而获得呈线性scale的性能提升。

```cpp
template<typename T>
class LockFreeStack {
 public:
  LockFreeStack(const LockFreeStack&) = delete;
  LockFreeStack(LockFreeStack&&) = delete;
  LockFreeStack& operator = (const LockFreeStack&) = delete;
  LockFreeStack& operator = (LockFreeStack&&) = delete;

  LockFreeStack() = default;
  ~LockFreeStack() = default;

  void Push(const T& val) {
    auto new_head = std::make_shared<Node>();
    new_head->val = val;
    new_head->next = nullptr;
    while (std::atomic_compare_exchange_weak(&head_, &(new_head->next), new_head) == false) {
      // do nothing
    }
  }

  void Pop(T* val) {
    std::shared_ptr<Node> popped;
    while (!std::atomic_compare_exchange_weak(&head_, &popped, popped ? popped->next : nullptr)
        || popped == nullptr) {
      // do nothing
    }
    *val = popped->val;
  }

 private:
  struct Node {
    T val;
    std::shared_ptr<Node> next;
  };

  std::shared_ptr<Node> head_;

};
```
