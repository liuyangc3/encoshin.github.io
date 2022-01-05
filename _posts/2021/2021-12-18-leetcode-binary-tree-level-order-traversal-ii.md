---
layout: post
title:  "LeetCode 107. 二叉树层序遍历 II（中等）"
date:   2021-12-18 20:30:10 +0800
author: Coshin
comments: true
category: 力扣题解
tags: LeetCode Medium Tree Breadth-First-Search Binary-Tree
---
> 给定一个二叉树的 `root`，返回*其节点值的自下而上的层序遍历*。
> （即，从左到右，从叶子到根一层一层）。
> 
> <img alt="" src="https://assets.leetcode.com/uploads/2021/02/19/tree1.jpg" style="width: 277px; height: 302px;">
> 
> **限制条件：**
> 
> * 树中节点数在范围 `[0, 2000]` 里。
> * `-1000 <= Node.val <= 1000`

## 解决方案

### 方法一：广度优先搜索

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> levelOrderBottom(TreeNode* root) {
        vector<vector<int>> ret;
        if (root == nullptr) return ret;
        queue<TreeNode*> que;
        que.push(root);
        while (!que.empty()) {
            int curLvSize = que.size();
            ret.push_back(vector<int>());
            for (int i = 1; i <= curLvSize; i++) {
                auto node = que.front();
                que.pop();
                ret.back().push_back(node->val);
                if (node->left) que.push(node->left);
                if (node->right) que.push(node->right);
            }
        }
        reverse(ret.begin(), ret.end());
        return ret;
    }
};
```

复杂度分析：
* 时间复杂度：*O*(n)。
  n 为树中节点的数量。
* 空间复杂度：*O*(n)。

## 参考链接

* [Binary Tree Level Order Traversal II - LeetCode](https://leetcode.com/problems/binary-tree-level-order-traversal-ii/){:target="_blank"}