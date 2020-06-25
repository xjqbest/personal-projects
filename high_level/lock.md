



### 读写锁

原子指令能为我们的服务赋予两个重要属性：wait-free和lock-free。前者指不管OS如何调度线程，每个线程都始终在做有用的事；后者比前者弱一些，指不管OS如何调度线程，至少有一个线程在做有用的事。如果我们的服务中使用了锁，那么OS可能把一个刚获得锁的线程切换出去，这时候所有依赖这个锁的线程都在等待，而没有做有用的事，所以用了锁就不是lock-free，更不会是wait-free。为了确保一件事情总在确定时间内完成，实时操作系统(RTOS)的关键代码至少是lock-free的。在我们广泛又多样的在线服务中，对时效性也有着严苛的要求，如果RPC中最关键的部分满足wait-free或lock-free，就可以提供更稳定的服务质量。

值得提醒的是，常见想法是lock-free或wait-free的算法会更快，但事实可能相反，因为：
 - lock-free和wait-free必须处理复杂的race condition和ABA problem，完成相同目的的代码比用锁更复杂。
 - 使用mutex的算法变相带“后退”效果。后退(backoff)指出现竞争时尝试另一个途径以避免激烈的竞争，mutex出现竞争时会使调用者睡眠，在高度竞争时规避了激烈的cacheline同步，使拿到锁的那个线程可以很快地完成一系列流程，总体吞吐可能反而高了。

mutex导致低性能往往是因为临界区过大（限制了并发度），或临界区过小（上下文切换开销变得突出，应考虑用adaptive mutex）。lock-free和wait-free算法的价值在于其避免了deadlock/livelock，在各种情况下的稳定表现，而不是绝对的高性能。但在一种情况下lock-free和wait-free算法的性能多半更高：就是算法本身可以用少量原子指令实现。实现锁也是要用原子指令的，当算法本身用一两条指令就能完成的时候，相比额外用锁肯定是更快了。

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
