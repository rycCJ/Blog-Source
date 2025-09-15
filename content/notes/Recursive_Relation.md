---
title: 'Recursive Relation' # <--- 修改这一行
date: 2025-09-15T18:25:38+08:00
draft: false
tags: ["Algorithm","Recursion", "C++"]
location: ""
featured: true
---

<a href="https://www.bilibili.com/video/BV1UD4y1Y769/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=de9d0b5f46420ead2e7be875b9558767" target="_blank" rel="noopener noreferrer">B站“灵茶山艾府“的解释</a>
## 递归的核心思想：不是人肉压栈，而是“甩锅”

{{< alert icon="fire" cardColor="#e63946" iconColor="#1d3557" textColor="#f1faee" >}}
初学者的困境：人肉模拟调用栈
{{< /alert >}}


当递归层数很少时，比如三层，你还能模拟：`main` 调用 `A`，`A` 调用 `B`，`B` 调用 `C`。`C` 返回给 `B`，`B` 返回给 `A`，`A` 返回给 `main`。但当层数一多，你的大脑就会像计算机内存溢出一样，难以追踪。

{{< alert >}}
**正确的递归思维方式是“信任”或者叫“信念之跃”（Leap of Faith）。**
{{< /alert >}}
你只需要关注两件事：

-   1.终止条件（Base Case）：什么时候问题小到可以直接解决，不需要再“甩锅”给下一层了？

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

我需要知道左子树的深度。

我需要知道右子树的深度。

整棵树的深度就是 `max(左深度, 右深度) + 1`。

- 4.第三步：写代码（信念之跃）。

如何获取左子树的深度？直接调用`maxDepth(root->left)`。相信它！ 不要去想它内部是怎么对 `root->left` 的子节点进行递归的。就把它当成 `int leftDepth = getLeftDepth()` 这么一个普通的函数调用。

如何获取右子树的深度？同理，调用 `maxDepth(root->right)`。相信它！

拿到 `leftDepth` 和 `rightDepth` 后，根据第二步的分析，`return std::max(leftDepth, rightDepth) + 1`;。

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

>> 先处理“根”：`cout << root->val`;

>>然后处理“左”：怎么处理？这是一个规模更小的同类问题，直接调用 `preOrderTraversal(root->left)`; 来搞定整个左子树。相信它能行！

>>最后处理“右”：同理，调用 `preOrderTraversal(root->right)`; 来搞定整个右子树。相信它也能行！

代码和逻辑完全一一对应，清晰明了。 你根本不需要去想，调用 `preOrderTraversal(root->left)`之后，它又会怎么打印，怎么调用它的左孩子和右孩子。你只要相信这个函数的功能定义是正确的，你调用它，它就能完成任务。

---
## 总结与建议
- 明确函数定义：在写递归函数之前，用一句话清晰地描述这个函数的功能。例如：“`maxDepth(node)`的功能是返回以 `node` 为根的子树的最大深度”。在后续的递归调用中，始终坚信你调用的就是这个已经实现的功能。

- 只考虑当前层：你的所有逻辑都应该只围绕“当前节点”展开。你需要做什么？你需要从子问题的解中得到什么信息？你如何利用这些信息？

- 找到终止条件：思考什么情况下问题规模最小，可以被直接解决，不再需要递归。这是递归的出口，没有它就会无限循环，导致“栈溢出”。

- 信任递归调用：这是最关键的一步。当你对 `function(sub_problem)` 进行调用时，就把它当成一个已知的、正确的黑盒。你的任务不是去追踪它的执行，而是去使用它的返回结果。

当你下次再遇到一个递归问题时，请抑制住你的大脑去模拟整个调用栈的冲动。强迫自己用上面的思维模式去思考，多练习几次，你会发现递归问题会变得异常清晰和简单。它是一种将复杂问题分解为简单、重复单元的强大思维工具。


