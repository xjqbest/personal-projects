
#### 面试问题

- 机器学习／深度学习框架的演进
- 相关竞品调研（xdl、byteps）
- elf/hippo/paddle fleet整体介绍

#### 编程题

 - [二叉树中的最大路径和](https://leetcode-cn.com/problems/binary-tree-maximum-path-sum/)
 - [判断一棵满二叉树是否为二叉搜索树](https://www.nowcoder.com/questionTerminal/76fb9757332c467d933418f4adf5c73d)
 
 ```cpp
#include <iostream>
#include <vector>
#include <stack>
using namespace std;

bool is_bst(const vector<int>& array) {
    if(array.size() <= 1) {
        return true;
    }
    stack<int> q;
    int cur_ptr = 0;
    int pre_ptr = -1;
    while(!q.empty() || (cur_ptr >= 0 && cur_ptr < array.size())) {
        while(cur_ptr < array.size()) {
            q.push(cur_ptr);
            cur_ptr = cur_ptr * 2 + 1;
        }
        int top = q.top();
        q.pop();
        if(pre_ptr != -1 && array[pre_ptr] >= array[top]) {
            return false;
        }
        pre_ptr = top;
        cur_ptr = 2 * top + 2;
    }
    return true;
}

int main() {
    cout << "case 1 " << is_bst(vector<int>{5, 2, 7, 1, 3, 6, 8}) << endl;
    cout << "case 2 " << is_bst(vector<int>{5, 2, 7, 1, 6, 3, 8}) << endl;
    cout << "case 3 " << is_bst(vector<int>{5}) << endl;
    cout << "case 4 " << is_bst(vector<int>{}) << endl;
}
 ```

- 哈希表

```cpp
// HashDict，将 uint64 的 key 映射到 [0, max_element_num) 的 id
// 这里使用链表法处理冲突
class HashDict {
 public:
  // max_element_num: 最大元素个数
  explicit HashDict(int max_element_num)
      : hash_table_(max_element_num, -1), next_(max_element_num, -1), keys_(max_element_num, 0), total_(0) {}
  ~HashDict() {}

  int GetElementNum() const { return total_; }
  int Capacity() const { return keys_.size(); }

  // 如果 key 已经存在于词典中, 给出它的 id
  // 如果 key 不在词典中, 则把它加入词典, 并返回给它分配的 id
  int InsertKey(uint64 key) {
    // NOTE(填写程序): 完成插入操作
    int* id_ptr = &(hash_table_[key % Capacity()]);
    while(*id_ptr >= 0) {
        if (keys_[*id_ptr] == key) return *id_ptr;
        id_ptr = &(next_[*id_ptr]);
    }
    keys_[total] = key;
    *id_ptr = total;
    return total++;
  }
  // 如果 key 已经存在于词典中, 返回它的 id, 否则返回 -1
  int LookupKey(uint64 key) const {
        int *id_ptr = &(hash_table_[key % Capacity()]);
    while (*id_ptr >= 0) {
      if (keys_[*id_ptr] == key) return *id_ptr;
      id_ptr = &(next_[*id_ptr]);
        }
        return -1;
  }

 private:
  std::vector<int> hash_table_;
  std::vector<int> next_;
  std::vector<uint64> keys_;
  int total_;
};
````

 - 表达式
 
 ```cpp
 class Expression {
 public:
  static bool GetBoolValue(const std::string &expression, bool *value) {
    // 边界检查
    if (expression.size() == 0) return false;
    // 简单情况
    if (expression == "0") {
      *value = false;
      return true;
    }
    if (expression == "1") {
      // NOTE(填写程序): 仿照 0 填写 1 的情况
      *value = true;
      return true;
    }
    // 去除首部空格
    if (expression[0] == ' ') {
      return GetBoolValue(expression.substr(1, expression.size() - 1), value);
    }
    if (expression[expression.size() - 1] == ' ') {
      // NOTE(填写程序): 去除尾部空格
      return GetBoolValue(expression.substr(0, expression.size() - 1), value);
    }
    // OR 运算
    int bracket_count = 0;
    for (int i = expression.size() - 1; i >= 0; --i) {
      if (expression[i] == ')') bracket_count++;
      if (expression[i] == '(') bracket_count--;
      if (bracket_count < 0) return false;
      if (bracket_count == 0 && expression.compare(i, 2, "OR") == 0) {
        // NOTE(填写程序): 完成 OR 运算
        bool left_value = false;
        bool left_valid = GetBoolValue(expression.substr(0, i), &left_value);
        if (!left_valid) {
            return false;
        } 

        bool right_value = false;
        bool right_valid = GetBoolValue(expression.substr(i + 2, expression.size() - i - 1), &right_value);
        if(!right_valid) {
            return false;
        }
        *value = left_value || right_value;
        return true;
      }
    }
    if (bracket_count != 0) return false;
    // AND 运算
    
    int bracket_count = 0;
    for (int i = expression.size() - 1; i >= 0; --i) {
      // NOTE(填写程序): 完成 AND 运算
      if (expression[i] == ')') bracket_count++;
      if (expression[i] == '(') bracket_count--;
      if (bracket_count < 0) return false;
      if (bracket_count == 0 && expression.compare(i, 3, "AND") == 0) {
        bool left_value = false;
        bool left_valid = GetBoolValue(expression.substr(0, i), &left_value);
        if (!left_valid) {
            return false;
        } 

        bool right_value = false;
        bool right_valid = GetBoolValue(expression.substr(i + 3, expression.size() - i - 2), &right_value);
        if(!right_valid) {
            return false;
        }
        *value = left_value && right_value;
        return true;
      }
    }
    // 去括号
    if (expression[0] == '(' && expression[expression.size() -1] == ')') {
      // NOTE(填写程序): 完成 去括号运算
      return GetBoolValue(expresssion.substr(1, expression.size() - 2), value);
    }
    return false;
  }
};

 ```
