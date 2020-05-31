#### [二叉树中所有距离为 K 的结点](https://leetcode-cn.com/problems/all-nodes-distance-k-in-binary-tree/)

`给定一个二叉树（具有根结点 root）， 一个目标结点 target ，和一个整数值 K 。返回到目标结点 target 距离为 K 的所有结点的值的列表。 
答案可以以任何顺序返回。`

**思路** ：如果这是个无向图／有向图，就直接bfs了，现在缺的只是子节点指向父节点的指针。两遍遍历，先用一个map记录子节点指向父节点，再bfs遍历一次即可。


#### [二叉树的最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)

`给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。`

`百度百科中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，
满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”`

**思路** ：我们可以用哈希表存储所有节点的父节点，然后我们就可以利用节点的父节点信息从 p 结点开始不断往上跳，并记录已经访问过的节点，
再从 q 节点开始不断往上跳，如果碰到已经访问过的节点，那么这个节点就是我们要找的最近公共祖先。

#### [二叉树的中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/)

**思路** ：背下来。
```cpp
    vector<int> inorderTraversal(TreeNode* root) {
        vector<int> ans;
        if (!root) {
            return ans;
        }
        stack<TreeNode*> st;
        TreeNode* p = root;
        while(!st.empty() || p) {
            while(p) {
                st.push(p);
                p = p->left;
            }
            TreeNode* r = st.top();
            st.pop();
            ans.push_back(r->val);
            p = r->right;
        }
        return ans;
    }
```

#### [二叉树的后序遍历](https://leetcode-cn.com/problems/binary-tree-postorder-traversal/)

**思路** ：后序是左-右-根，可以迭代的时候先按根-右-左，最后把输出数组反转一下。

#### [不同的二叉搜索树](https://leetcode-cn.com/problems/unique-binary-search-trees/)

`给定一个整数 n，求以 1 ... n 为节点组成的二叉搜索树有多少种？`

**思路** ： 动态规划题，

```cpp
// f[n] = f[0] * f[n-1] + f[1] * f[n-2] + ...+ f[n-1] * f[0];
// f[0] = 1  f[1] = 1
```

#### [不同的二叉搜索树 II](https://leetcode-cn.com/problems/unique-binary-search-trees-ii/)

`给定一个整数 n，生成所有由 1 ... n 为节点所组成的二叉搜索树。`

**思路** ：递归建树 vector<TreeNode*> recur(int start, int end) 分别以每个节点作为根节点。

#### [所有可能的满二叉树](https://leetcode-cn.com/problems/all-possible-full-binary-trees/)

`满二叉树是一类二叉树，其中每个结点恰好有 0 或 2 个子结点。返回包含 N 个结点的所有可能满二叉树的列表。 答案的每个元素都是一个可能树的根结点。答案中每个树的每个结点都必须有 node.val=0。你可以按任何顺序返回树的最终列表。`

**思路** ：跟上面的题类似。代码也几乎一样，区别是vector<TreeNode*> recur(int start, int end)返回的列表里没有nullptr。特殊情况是叶子节点，建立叶子节点后直接返回。

