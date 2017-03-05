---
title: 114 Flatten Binary Tree to Linked List
date: 2017-03-05 08:52:23
tags: [binary_tree, list]
categories: [leetcode]
---
# 将二叉树展开成为链表
<!--more-->
[description](https://leetcode.com/problems/flatten-binary-tree-to-linked-list/?tab=Description)

	public static void flattenRecursive(TreeNode root) {
        if(null == root)
            return;
        flat(root, null);
    }
    /*the next need to be apped to the tail of subtree who's root
    is "root"*/
    public static void flat(TreeNode root, TreeNode next) {
        if(null != root.right)
        /*flatten the right, the near ancestor's right must be
        appended to it*/
            flat(root.right, next);
        if(null != root.left) {
            TreeNode right = root.right;
            root.right = root.left;
            /*it it has no right, child, the nearest ancestor's right is to be
            appended*/
            flat(root.left, right == null ? next : right);
            root.left = null;
        }
        else/*it has no child, just do the append, our passing direction make
        it the left tree's right bound element*/
            if(null == root.right)
                root.right = next;
    }
	
	public static void flattenIterative(TreeNode root) {
        if(null == root)
            return;
        LinkedList<TreeNode> stack = new LinkedList<>();
        stack.push(root);
        /*Think about it in recursion, for the node root, if it has left child,
        its right pointer must point to it left child, or if it has right child, it
        must be appended, otherwise, its right must point to the nearest ancestor's right child.*/
        while(!stack.isEmpty()) {
            root = stack.pop();
        /*according to the description above, we have the following push order*/
            if(null != root.right)
                stack.push(root.right);
            if(null != root.left)
                stack.push(root.left);
            root.left = null;
            if(!stack.isEmpty())
                root.right = stack.peek();
        }
    }
    
  本质上是一个搜索树展开为有序链表，本质上，对于一个根节点，需要将其右子树，或者最近邻祖先右子树连接到左子树的最右节点的右指针上，我们只需在往左遍历时，将右侧待追加根节点传入即可。
  递归实现中，可以恰好让待追加子树根位于栈顶，使用peek()读取，虽然这与递归的处理顺序不同，但右子树的展开不会造成其根节点的变化，因此调换处理顺序不会影响最终结果。

