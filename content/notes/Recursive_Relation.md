---
title: "递归基础及题目（Recursive Relation）" # <--- 修改这一行
date: 2025-09-15T18:25:38+08:00
draft: false
tags: ["Algorithm", "Recursion", "C++"]
location: ""
featured: true
---

<a href="https://www.bilibili.com/video/BV1UD4y1Y769/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=de9d0b5f46420ead2e7be875b9558767" target="_blank" rel="noopener noreferrer">B 站“灵茶山艾府“的解释</a>

## 递归的核心思想：不是人肉压栈，而是“甩锅”

初学者的困境：人肉模拟调用栈

当递归层数很少时，比如三层，你还能模拟：`main` 调用 `A`，`A` 调用 `B`，`B` 调用 `C`。`C` 返回给 `B`，`B` 返回给 `A`，`A` 返回给 `main`。但当层数一多，你的大脑就会像计算机内存溢出一样，难以追踪。

**正确的递归思维方式是“信任”或者叫“信念之跃”（Leap of Faith）。**

你只需要关注两件事：

- 1.终止条件（Base Case）：什么时候问题小到可以直接解决，不需要再“甩锅”给下一层了？

- 2.递归关系（Recursive Relation）：如何把当前问题，分解成一个或多个规模更小的同类子问题，并假设（信任）下一层调用能完美解决这些子问题。然后，你只需要思考如何利用子问题的解来组合成当前问题的解。

**你不需要去想下一层是怎么实现的，你只要相信它能给你正确的结果就行了。**就像调用一个库函数`sort()`，你不会去关心它内部是快排还是归并.

**把递归函数本身，也当成一个黑盒的、可信赖的库函数来调用。**

---

## 以二叉树为例

```cpp
#include <iostream>
#include <algorithm>

// 定义二叉树节点
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

/*
  我们来定义函数 maxDepth(root)
  它的功能是：接收一个树的根节点，返回这棵树的最大深度。
*/
int maxDepth(TreeNode* root) {
    // 1. 终止条件 (Base Case)
    // 如果这个节点是空的（比如一个叶子节点的子节点），
    // 那么它构成的树深度就是 0。这是最简单、可以直接回答的问题。
    if (root == nullptr) {
        return 0;
    }

    // 2. 递归关系 (Recursive Relation)
    // 对于任何一个非空的节点 root，它的最大深度是多少？
    // 我们可以把它分解成两个子问题：
    //   - 左子树的最大深度是多少？
    //   - 右子树的最大深度是多少？

    // 这两个子问题的规模都比当前问题（整棵树的深度）要小。
    // 而且它们是同类问题：都是“计算一棵树的最大深度”。

    // 所以，我们可以“甩锅”了！
    // 我假设 maxDepth() 这个函数已经能完美工作了（信念之跃）。
    // 我把它当成一个已知的、可靠的工具来调用。

    // 我“相信”下面这行代码能正确计算出左子树的深度。
    // 我不去想 aleftDepth 是怎么一步步算出来的。
    int leftDepth = maxDepth(root->left);

    // 我同样“相信”这行代码能正确计算出右子树的深度。
    int rightDepth = maxDepth(root->right);

    // 3. 组合子问题的解
    // 现在，我已经拿到了两个子问题的答案：leftDepth 和 rightDepth。
    // 如何用这两个答案来解决我当前的问题（以 root 为根的树的深度）？
    // 很显然，整棵树的深度 = 左右子树深度的较大者 + 1 (当前这一层)
    return std::max(leftDepth, rightDepth) + 1;
}

int main() {
    /*
        构造一棵树:
            3
           / \
          9  20
            /  \
           15   7
    */
    TreeNode* root = new TreeNode(3);
    root->left = new TreeNode(9);
    root->right = new TreeNode(20);
    root->right->left = new TreeNode(15);
    root->right->right = new TreeNode(7);

    std::cout << "这棵树的最大深度是: " << maxDepth(root) << std::endl; // 应该输出 3

    delete root->left;
    delete root->right->left;
    delete root->right->right;
    delete root->right;
    delete root;

    return 0;
}
```

**如何正确思考 maxDepth(root) 的过程？**

- 1.我的目标：实现 maxDepth(root) 函数。

- 2.第一步：找终止条件。 什么情况下最简单？树是空的！`if (root == nullptr)`，深度就是 0。搞定。

- 3.第二步：找递归关系。 如果树不空，`root` 指向一个节点。这棵树的深度和它的左右孩子有什么关系？

> > 左子树的深度。

> > 右子树的深度。

> > 整棵树的深度就是 `max(左深度, 右深度) + 1`。

- 4.第三步：写代码（信念之跃）。

> > 如何获取左子树的深度？直接调用`maxDepth(root->left)`。相信它！ 不要去想它内部是怎么对 `root->left` 的子节点进行递归的。就把它当成 `int leftDepth = getLeftDepth()` 这么一个普通的函数调用。

> > 如何获取右子树的深度？同理，调用 `maxDepth(root->right)`。相信它！

> > 拿到 `leftDepth` 和 `rightDepth` 后，根据第二步的分析，`return std::max(leftDepth, rightDepth) + 1`;。

**整个思考过程，我们只关注“当前层”的逻辑。** 我们把对子树的处理完全委托给了递归调用，我们是“管理者”，不是“微观执行者”。

---

## 再举一例：二叉树的前序遍历

前序遍历的顺序是：根 -> 左 -> 右。这个定义本身就充满了递归的味道

```cpp
void preOrderTraversal(TreeNode* root) {
    // 1. 终止条件
    if (root == nullptr) {
        return; // 空树就直接返回，什么也不做
    }

    // 2. 处理当前节点 (根)
    std::cout << root->val << " "; // 访问根节点

    // 3. 递归处理左子树 (左)
    // “信念之跃”：我相信这个调用会完美地以前序遍历的方式打印完整个左子树
    preOrderTraversal(root->left);

    // 4. 递归处理右子树 (右)
    // “信念之跃”：我相信这个调用会完美地以前序遍历的方式打印完整个右子树
    preOrderTraversal(root->right);
}
```

正确思考 `preOrderTraversal`：

- 1.我的目标：按“根-左-右”的顺序打印树。

- 2.终止条件：`root` 是 `nullptr`，没什么可打印的，直接 `return`。

- 3.当前层的逻辑：如果 `root` 不是 `nullptr`，根据“根-左-右”的定义，我应该：

> > 先处理“根”：`cout << root->val`;

> > 然后处理“左”：怎么处理？这是一个规模更小的同类问题，直接调用 `preOrderTraversal(root->left)`; 来搞定整个左子树。相信它能行！

> > 最后处理“右”：同理，调用 `preOrderTraversal(root->right)`; 来搞定整个右子树。相信它也能行！

代码和逻辑完全一一对应，清晰明了。 你根本不需要去想，调用 `preOrderTraversal(root->left)`之后，它又会怎么打印，怎么调用它的左孩子和右孩子。你只要相信这个函数的功能定义是正确的，你调用它，它就能完成任务。

---

## 总结与建议

- 明确函数定义：在写递归函数之前，用一句话清晰地描述这个函数的功能。例如：“`maxDepth(node)`的功能是返回以 `node` 为根的子树的最大深度”。在后续的递归调用中，始终坚信你调用的就是这个已经实现的功能。

- 只考虑当前层：你的所有逻辑都应该只围绕“当前节点”展开。你需要做什么？你需要从子问题的解中得到什么信息？你如何利用这些信息？

- 找到终止条件：思考什么情况下问题规模最小，可以被直接解决，不再需要递归。这是递归的出口，没有它就会无限循环，导致“栈溢出”。

- 信任递归调用：这是最关键的一步。当你对 `function(sub_problem)` 进行调用时，就把它当成一个已知的、正确的黑盒。你的任务不是去追踪它的执行，而是去使用它的返回结果。

当你下次再遇到一个递归问题时，请抑制住你的大脑去模拟整个调用栈的冲动。强迫自己用上面的思维模式去思考，多练习几次，你会发现递归问题会变得异常清晰和简单。它是一种将复杂问题分解为简单、重复单元的强大思维工具。

## 1. 基础遍历类 (Basic Traversal)

这类问题的核心是“访问”到每一个节点并执行简单操作，递归函数本身通常没有返回值（void）或者返回一个包含所有节点值的列表。

**核心思路：**
定义一个 traverse(node) 函数，在函数内部先处理当前节点，然后递归调用 `traverse(node->left) `和 `traverse(node->right)`。根据处理当前节点的时机不同，分为前、中、后序遍历。

**经典题目:**

### <a href="https://leetcode.cn/problems/binary-tree-preorder-traversal/" target="_blank" rel="noopener noreferrer">144. 二叉树的前序遍历</a>

```cpp
    class Solution {
public:
    void perorder(TreeNode* root, vector<int>& res) {
        if (root == nullptr)
            return;
        res.push_back(root->val);
        perorder(root->left, res);
        perorder(root->right, res);
    }
    vector<int> preorderTraversal(TreeNode* root) {
        vector<int> res;
        perorder(root, res);
        return res;
    }
};
```

### <a href=" https://leetcode.cn/problems/binary-tree-inorder-traversal/" target="_blank" rel="noopener noreferrer">94. 二叉树的中序遍历</a>

```cpp
class Solution
{
public:
       void inorder(TreeNode *node, vector<int> &res)
    {
        if (node == nullptr)
            return;
        inorder(node->left, res);
        res.push_back(node->val);
        inorder(node->right, res);
    }
    vector<int> inorderTraversal(TreeNode *root)
    {
        vector<int> res;
        inorder(root, res);
        return res;
    }
};
```

### <a href=" https://leetcode.cn/problems/binary-tree-postorder-traversal/" target="_blank" rel="noopener noreferrer">145. 二叉树的后序遍历</a>

```cpp
class Solution {
public:
    void postorder(TreeNode* root, vector<int>& res) {
        if (root == nullptr)
            return;

        postorder(root->left, res);

        postorder(root->right, res);
        res.push_back(root->val);
    }
    vector<int> postorderTraversal(TreeNode* root) {
        vector<int> res;
        postorder(root, res);
        return res;
    }
};
```

## 2. 分治 / 自下而上信息汇总 (Divide and Conquer / Bottom-up)

这是最最常见的一类递归问题。你相信递归函数能帮你解决子问题，然后你只需要思考如何利用子问题的解来解决当前问题。

**核心思路** ：
“我不知道怎么解决整棵树的问题，但我假设 `solve(root->left) `和 `solve(root->right) `已经帮我解决了左右子树的问题并返回了正确的结果。现在，我只需要在当前 root 节点，利用这两个结果，计算出当前树的结果，然后 return 回去。”

**经典题目**:

### <a href="  https://leetcode.cn/problems/maximum-depth-of-binary-tree/" target="_blank" rel="noopener noreferrer">104. 二叉树的最大深度</a> (我们之前讨论过的)

左子树深度 = `solve(root->left)`, 右子树深度 = `solve(root->right)`

当前树深度 = max(左子树深度, 右子树深度) + 1

### <a href=" https://leetcode.cn/problems/minimum-depth-of-binary-tree/" target="_blank" rel="noopener noreferrer">111. 二叉树的最小深度 </a> (最大深度的变体，注意处理只有单边子树的情况)

```cpp
    int minDepth(TreeNode* root) {
        if (root == nullptr)
            return 0;
        int ldepth = minDepth(root->left);
        int rdepth = minDepth(root->right);
        if (root->right == nullptr) {
            return ldepth+1;
        }
        if (root->left == nullptr) {
            return rdepth+1;
        }
        return min(ldepth, rdepth) + 1;
    }
```

### <a href=" https://leetcode.cn/problems/diameter-of-binary-tree/" target="_blank" rel="noopener noreferrer">543. 二叉树的直径</a>

子问题返回子树的深度。

当前节点计算的“穿过我的直径”是 左深度 + 右深度，同时更新全局最大值。

```cpp
int landrDepth(TreeNode* root,int& maxdepth) {
        if (root == nullptr)
            return 0;
        int ldepth = landrDepth(root->left,maxdepth);
        int rdepth = landrDepth(root->right,maxdepth);
        int currentdepth = ldepth + rdepth;
        maxdepth = max(maxdepth,currentdepth);
        return max(ldepth,rdepth)+1;
    }
    int diameterOfBinaryTree(TreeNode* root) {
        int maxdepth=0;
        landrDepth(root,maxdepth);

        return maxdepth;
    }
```

### <a href=" https://leetcode.cn/problems/balanced-binary-tree/" target="_blank" rel="noopener noreferrer">110. 平衡二叉树</a>

引入-1，蛮有意思......

子问题需要返回两个信息：子树是否平衡，以及子树的高度。

```cpp
class Solution {
public:
    bool isBalanced(TreeNode* root) {

        return dep(root)!=-1;

    }
    int dep(TreeNode* root) {
        if(root==nullptr){
            return 0;
        }
        int ldepth = dep(root->left);
        if(ldepth==-1){
            return -1;
        }
        int rdepth = dep(root->right);
        if(rdepth==-1){
            return -1;
        }
        // chazhi = max(chazhi,abs(ldepth - rdepth));
        if(abs(ldepth - rdepth)>1){
            return -1;
        }
        return max(ldepth,rdepth)+1;
    }
};
```

### <a href="https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/" target="_blank" rel="noopener noreferrer">236.二叉树的最近公共祖先</a>

- 子问题返回在子树中是否找到了 p 或 q。

- 当前节点根据左右子树的返回情况做出判断。

把一个复杂的“寻找 LCA”问题，降维成了一个简单的“寻找 p 或 q”的问题 emmm 怪有意思......

```cpp
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        //如果你确定树中所有节点的值唯一（比如二叉搜索树 BST 中常常是这样），那用 root->val == p->val 也可以。
    if (root == nullptr || root == p || root == q) {
            return root;
        }
        // 函数不是在“寻找祖先”，而是在“汇报发现”。这个算法巧妙地改变了子问题的定义
        // 把一个复杂的“寻找LCA”问题，降维成了一个简单的“寻找p或q”的问题
        TreeNode *lAncestor = lowestCommonAncestor(root->left, p, q);

        TreeNode *rAncestor = lowestCommonAncestor(root->right, p, q);

        if (lAncestor ==nullptr&&rAncestor==nullptr)
            return nullptr;
        if (lAncestor != nullptr && rAncestor != nullptr)
            return root;
        if (lAncestor == nullptr && rAncestor != nullptr)
            return rAncestor;
        return lAncestor;
    }
```

## 3. 路径问题 / 自上而下信息传递 (Path Problems / Top-down)

这类问题与上一类相反，子问题的解决需要依赖其父节点的信息。因此，你需要通过递归函数的参数将信息自上而下地传递下去。

**核心思路**：
“我需要定义一个 `solve(node, state) `函数，其中 `state` 是从根节点到我父节点为止积累的状态。在函数内部，我根据 `state `和当前`node` 的值计算出新的状态 new_state，然后把它传递给我的子节点`solve(node->left, new_state)`和 `solve(node->right, new_state)`。”

**经典题目**:

### <a href="https://leetcode.cn/problems/path-sum/description/" target="_blank" rel="noopener noreferrer">112. 路径总和</a>

向下传递 `targetSum - node->val`。

```cpp
  bool hasPathSum(TreeNode *root, int targetSum) {
    if (root == nullptr)
      return 0;
    targetSum -= root->val;
    if (root->left == nullptr && root->right == nullptr)
      return targetSum == 0;

    bool res1 = hasPathSum(root->left, targetSum);
    bool res2 = hasPathSum(root->right, targetSum);
    return res1 || res2;
  }
```

### <a href="https://leetcode.cn/problems/sum-root-to-leaf-numbers/description/" target="_blank" rel="noopener noreferrer">129. 求根节点到叶节点数字之和</a>

向下传递 `currentSum \* 10 + node->val`。

辅助函数并没有返回值，而是把值存在了常量里面。因为返回的不是结果，所以不用只是子问题的答案，可以是最终结果 totalsum。

```cpp
class Solution {
public:
    int totalsum = 0;
    void traverse(TreeNode* node, int currentnum) {
        if (node == nullptr)
            return;
        int parentnum = currentnum * 10 + node->val;
        if (node->left == nullptr && node->right == nullptr) {
            totalsum = totalsum + parentnum;
            return;
        }
        traverse(node->left, parentnum);
        traverse(node->right, parentnum);
    }
    int sumNumbers(TreeNode* root) {

        traverse(root, 0);
        return totalsum;
    }
};
```

### <a href="https://leetcode.cn/problems/binary-tree-paths/description/" target="_blank" rel="noopener noreferrer">257. 二叉树的所有路径 </a>

向下传递当前路径的字符串 `currentPath + "->" + node->val`。

```cpp
class Solution {
public:
    vector<string> ans;
    void getpath(TreeNode* root, string path) {

        if (root == nullptr) {
            return;
        }
        path = path+to_string(root->val);
        if (root->right == nullptr && root->left == nullptr) {
            ans.push_back(path);
            return;
        }
        path = path+"->";
        getpath(root->right, path);
        getpath(root->left, path);
    }
    vector<string> binaryTreePaths(TreeNode* root) {
        getpath(root, "");
        return ans;
    }
};
```

优化方案：**使用回溯（Backtracking）避免浪费** 还不太会，放放吧&#x23f3;  
路径用 string& path 传引用，递归中添加/移除（pop_back），结果直接 push 到共享 res

- 无临时向量创建/合并;
- 字符串修改就地（O(1) 添加/移除）;
- 空间：O(log n) 递归栈 + O(n \_ L) 最终 res;
- 时间：O(n \_ L) 但常数小，无拷贝。

```cpp
class Solution {
public:
    void getpath(TreeNode* node, string& path, vector<string>& res) {
        if (node == nullptr) return;

        // 添加当前值（记录长度以便回溯）
        int prev_len = path.size();
        path += to_string(node->val);

        if (node->left == nullptr && node->right == nullptr) {
            res.push_back(path);  // 直接 push 当前路径
        } else {
            path += "->";  // 添加箭头
            getpath(node->left, path, res);
            getpath(node->right, path, res);
            path.resize(path.size() - 2);  // 移除 "->"
        }

        path.resize(prev_len);  // 移除当前值（回溯）
    }

    vector<string> binaryTreePaths(TreeNode* root) {
        vector<string> res;
        string path = "";
        getpath(root, path, res);
        return res;
    }
};

```

### <a href=" https://leetcode.cn/problems/path-sum-ii/description/" target="_blank" rel="noopener noreferrer">113. 路径总和 II (路径总和的升级版) </a>

向下传递当前路径的节点列表` vector<int> currentPath`。

```cpp
class Solution {
public:
    vector<int> path;
    vector<vector<int>> ans;
    void cacula(TreeNode* root, int currentnum) {
        if(root==nullptr){
            return;
        }
        path.push_back(root->val);
        currentnum = currentnum - root->val;

        if (root->right == nullptr && root->left == nullptr) {
            if (currentnum == 0) {
                ans.push_back(path);
                path.pop_back();
                return;
            }
            path.pop_back();
            return;
        }
        cacula(root->left, currentnum);
        cacula(root->right, currentnum);

        path.pop_back();
    }
    vector<vector<int>> pathSum(TreeNode* root, int targetSum) {
        if(root==nullptr)
        {
            return ans;
        }
        cacula(root, targetSum);
        return ans;
    }
};
```

### <a href=" https://leetcode.cn/problems/binary-tree-maximum-path-sum/?envType=study-plan-v2&envId=top-100-liked" target="_blank" rel="noopener noreferrer">124. 二叉树中的最大路径和</a>

辅助函数的主要功能是求解单边最大，这样计算左右两边最大就可以得到想要的最大值。
不要拘泥于辅助函数必须返回目标值，这种直接得到目标值行不通的，可以考虑分解目标，但是将目标值存储在变量中。

```cpp
    int globalmax = numeric_limits<int>::min();
    int cacule2(TreeNode \*root) {
        if (root == nullptr)
            return 0;

        int lmax = max(0,cacule2(root->left));
        int rmax = max(0,cacule2(root->right));
        int currentnum = root->val + lmax + rmax;
        globalmax = max(currentnum, globalmax);
        return root->val+max(lmax,rmax);

}

    int maxPathSum(TreeNode \*root) {
        globalmax = numeric_limits<int>::min();
        cacule2(root);
        return globalmax;
}
```

## 4. 结构比较与修改 (Structural Comparison & Modification)

这类问题通常涉及两棵树，或者需要翻转、修改树的结构。

**核心思路：**

递归函数通常同时作用于两棵树的对应节点，或者在修改完当前节点的结构后，再对子节点进行递归调用。

**经典题目:**

### <a href=" https://leetcode.cn/problems/same-tree/" target="_blank" rel="noopener noreferrer">100. 相同的树 </a>

`isSameTree(p, q)` 依赖于 `isSameTree(p->left, q->left) `和 `isSameTree(p->right, q->right) `的结果。

```cpp
class Solution {
public:
    bool isSameTree(TreeNode* p, TreeNode* q) {
        if(p==nullptr&&q==nullptr){
            return true;
        }
        if((p==nullptr&&q!=nullptr)||(p!=nullptr&&q==nullptr)||p->val!=q->val)
        {
            return false;
        }

        bool isLeftSame = isSameTree(p->left,q->left);
        bool isRightSame = isSameTree(p->right,q->right);
        return isLeftSame&&isRightSame;
    }
};
```

### <a href=" https://leetcode.cn/problems/symmetric-tree/" target="_blank" rel="noopener noreferrer">101. 对称二叉树 </a>

这是判断两棵树是否镜像对称的变体。递归函数 `isMirror(p, q) `依赖于 `isMirror(p->left, q->right) `和` isMirror(p->right, q->left)`。

```cpp
class Solution {
public:
    bool isMirror(TreeNode* p, TreeNode* q) {
        if (p == nullptr && q == nullptr) {
            return true;
        }
        if ((p == nullptr && q != nullptr) || (p != nullptr && q == nullptr)
            || (p->val != q->val)) { return false; }
        bool isSameOuter = isMirror(p->left, q->right);
        bool isSameInner = isMirror(p->right, q->left);
        return isSameOuter && isSameInner;
    }
    bool isSymmetric(TreeNode* root) {
        if (root == nullptr) {
            return true;
        }
        return isMirror(root->left, root->right

        );
    }
```

### <a href=" https://leetcode.cn/problems/invert-binary-tree/" target="_blank" rel="noopener noreferrer">226. 翻转二叉树 </a>

在当前节点交换左右子节点，然后递归调用` invertTree(root->left)` 和 `invertTree(root->right)`。

```cpp
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (root == nullptr)
            return nullptr;
        TreeNode* left = invertTree(root->left);
        TreeNode* right = invertTree(root->right);
        root->right = left;
        root->left = right;
        return root;
    }
};

```

## 总结与练习建议

| 题号         | 题目                 | 核心思路                                   | 类 别           | 状态     |
| ------------ | -------------------- | ------------------------------------------ | --------------- | -------- |
| 94, 144, 145 | 二叉树遍历           | 访问所有节点                               | 基础遍历        | &#x2705; |
| 104, 111     | 树的深度             | 从子树获取深度，组合成当前深度             | 分治 / 自下而上 | &#x2705; |
| 543          | 二叉树的直径         | 从子树获取深度，计算穿过当前节点的直径     | 分治 / 自下而上 | &#x2705; |
| 110          | 平衡二叉树           | 从子树获取“是否平衡”和“深度”两个信息       | 分治 / 自下而上 | &#x2705; |
| 236          | 最近公共祖先         | 从子树获取是否包含目标节点的信息           | 分治 / 自下而上 | &#x2705; |
| 112, 113     | 路径总和             | 将目标和减去当前值，向下传递               | 路径 / 自上而下 | &#x2705; |
| 129          | 求根到叶节点数字之和 | 将当前路径和乘以 10 加上当前值，向下传递   | 路径 / 自上而下 | &#x2705; |
| 257          | 二叉树的所有路径     | 将当前路径字符串拼接上当前值，向下传递     | 路径 / 自上而下 | &#x2705; |
| 100          | 相同的树             | 同时递归比较两棵树的对应子树               | 结构比较        | &#x2705; |
| 101          | 对称二叉树           | 同时递归比较一棵树的内外侧子树             | 结构比较        | &#x2705; |
| 226          | 翻转二叉树           | 交换当前节点的左右子节点，然后递归翻转子树 | 结构修改        | &#x2705; |

**如何练习：**

从基础遍历开始，确保你理解前、中、后序的区别。

主攻“分治 / 自下而上”，这是最重要的递归模式。以“最大深度”为模板，彻底理解“信任”递归调用的思想。

然后练习“路径 / 自上而下”，理解通过参数传递状态的方法。

最后解决结构类问题，它们通常是前面几种思想的结合或变体。

几乎所有二叉树问题都能被归入这几类。当你遇到一个新问题时，先思考一下：“解决这个问题，我需要从子树获得什么信息，还是需要向子树传递什么信息？” 这能帮助你快速确定递归的结构。

## 是否需要返回值问题

如果叶节点或中间节点需要“向上报告”信息（如子树结果），用返回值
如果只是“遍历 + 收集到外部容器”，用 void + 共享状态

## 更多练习题目：\*\*

### <a href=" https://leetcode.cn/problems/convert-sorted-array-to-binary-search-tree/" target="_blank" rel="noopener noreferrer">108. 将有序数组转换为二叉搜索树 </a>

```cpp
class Solution {
public:
    TreeNode* sortedArrayToBST(vector<int>& nums) { return help(nums,0,nums.size()-1); }
    TreeNode* help(vector<int>& nums, int left, int right) {
        if (left > right) {  //左右相同的下一步就是左边大于右边，左右相同得放进树里面
            return nullptr;
        }
        int mid = (left + right + 1) / 2;
        TreeNode* root = new TreeNode(nums[mid]);
        root->left = help(nums, left, mid - 1);
        root->right = help(nums, mid + 1, right);
        return root;
    }
};
```

### <a href=" https://leetcode.cn/problems/validate-binary-search-tree/" target="_blank" rel="noopener noreferrer">98. 验证二叉搜索树 </a>

方案 1：中序遍历（推荐的简洁方法）
**核心原理：**
对一个二叉搜索树（BST）进行中序遍历（左 → 根 → 右），得到的节点值序列一定是严格递增的。

```cpp
class Solution {
private:
    // 必须用成员变量或引用参数来维护这个状态
    long long prevVal = LLONG_MIN;

public:
    bool isValidBST(TreeNode* root) {
        if (root == nullptr) {
            return true;
        }

        // 1. 递归检查左子树
        if (!isValidBST(root->left)) {
            return false;
        }

        // 2. 检查当前节点（中序遍历的核心逻辑）
        // 核心：当前值必须严格大于上一个值
        if (root->val <= prevVal) {
            return false;
        }

        // 3. 更新上一个值
        prevVal = root->val;

        // 4. 递归检查右子树
        return isValidBST(root->right);
    }
};
```

方案 2：上下限递归（更基础的方法）

```cpp
class Solution {
public:
    bool isValidBST(TreeNode* root) {
        return helper(root,LLONG_MIN,LLONG_MAX);
    }
    bool helper(TreeNode* root, long long lower, long long upper) {
        if (root == nullptr)
            return true;
        if (root->val <= lower || root->val >= upper)
            return false;
        bool left = helper(root->left, lower, root->val);
        bool right = helper(root->right, root->val, upper);
        return left && right;
    }
};
```

### <a href=" https://leetcode.cn/problems/kth-smallest-element-in-a-bst/" target="_blank" rel="noopener noreferrer">230. 二叉搜索树中第 K 小的元素 </a>

```cpp
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        vector<int> result;
        helper(root,result);
        return result[k-1];
    }
    void helper(TreeNode* root, vector<int>& res) {
        if (root == nullptr)
            return;
        helper(root->left,res);
        res.push_back(root->val);
        helper(root->right,res);
    }
};
```

使用布尔信号提前终止

```cpp
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        vector<int> result;
        helper(root, result, k);
        return result[k - 1];
    }
    bool helper(TreeNode* root, vector<int>& res, int n) {
        // 递归终止条件 1: 节点为空，返回 false (继续遍历)
        if (root == nullptr)
            return false;
        // 递归终止条件 2: 已经找到 K 个元素，直接返回 true (停止)
        if (res.size() == n) {
            return true;
        }
        if (helper(root->left, res, n))
            return true;
        res.push_back(root->val);
        if (res.size() == n) {
            return true;
        }
        return helper(root->right, res, n);

    }
};
```

### <a href=" https://leetcode.cn/problems/flatten-binary-tree-to-linked-list/" target="_blank" rel="noopener noreferrer">114. 二叉树展开为链表 </a>

方法一：普通方法（递归）

```cpp
class Solution {
public:
    void flatten(TreeNode* root) {
        vector<TreeNode*> res;
        help(root, res);
        int n = res.size();

        for (int i = 0; i < n-1; i++) {
            TreeNode* node = res[i];
            node->left =nullptr;
            node->right = res[i+1];
        }
    }
    void help(TreeNode* root, vector<TreeNode*>& res) {
        if (root != nullptr) {
            res.push_back(root);
            help(root->left, res);
            help(root->right, res);
        }
    }
};
```

方法二：递归/逆向先序遍历 (Right → Left → Root)

```cpp
class Solution {
public:
    void flatten(TreeNode* root) {
        help(root);
    }
    TreeNode* pre = nullptr;
    void help(TreeNode* root) {
        if (root == nullptr) {
            return;
        }
        help(root->right);
        help(root->left);


        root->right = pre;
        root->left = nullptr;
        pre = root;
        return;
    }
};

```

方法三：迭代 + 寻找前驱 (Morris Traversal 思想) //看不懂思密达

```cpp
class Solution {
public:
    void flatten(TreeNode* root) {
            TreeNode* curr = root;

    while (curr != nullptr) {
        // 1. 如果有左子树
        if (curr->left != nullptr) {

            // 2. 找到左子树的最右节点 (前驱 predecessor)
            TreeNode* predecessor = curr->left;
            while (predecessor->right != nullptr) {
                predecessor = predecessor->right;
            }

            // 3. 关键连接：将右子树接在左子树的末尾
            predecessor->right = curr->right;

            // 4. 重排：将左子树整体搬到右侧
            curr->right = curr->left;
            curr->left = nullptr; // 清空左指针
        }

        // 5. 前进到下一个节点（即原先的右子树或新的右子树的根）
        curr = curr->right;
    }
    }

};
```
