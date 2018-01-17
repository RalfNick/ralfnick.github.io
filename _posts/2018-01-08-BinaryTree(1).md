---
layout: post
title: "BinaryTree(1)"
date: 2018-01-08
description: "学习数据结构，提升内功"
tag: 数据结构与算法
---
由于基础比较差，什么设计模式，数据结构，操作系统。。。均没有学过，着实让人着急。没办法，只能耐着性子慢慢来。把遇到的东西一点一点搞懂。本篇就来一点提升内功的，数据结构中的二叉树，注意，不是二叉查找树，后面我还会练习二叉查找树。

### 1. 二叉树？

看一下wiki上的说明
>In computer science, a binary tree is a tree data structure in which each node has at most two children, which are referred to as the left child and the right child.
>

意思是说二叉树是一种数据结构，每个节点最多有两个子节点，也就是左右两个子节点。当然，也有三叉树，四叉树。。。n叉树，只是二叉树我们最常用。
```Java

特点：
1、每个结点最多有两颗子树，结点的度最大为2，所以树的度也是2。
2、左子树和右子树是有顺序的，次序不能颠倒。
3、即使某结点只有一个子树，也要区分左右子树。
```


```Java

种类：
1、满二叉树
2、完全二叉树
3、非完全二叉树

```
这里不给出具体的定义：满二叉树很好理解，完全二叉树主要要学会跟满二叉树对比，就可以判断出是否是完全二叉树，除了完全二叉树之外的数，都是非完全二叉树。

搜了下，二叉树的各种性质，看的有点头大。我就以简单的用法入手，看代码中怎么实现简单的定义与使用，下面就来看代码。
对于二叉树的定义相对比较简单，设置左右节点即可。

```java
public class TreeNode<T>{

    private T data;
    private TreeNode left;
    private TreeNode right;

}

```
### 2. 二叉树基本算法

(1) 查找最大深度

```java
/**
	 * 获取最大深度
	 * @param root
	 * @return
	 */
	public int getMaxDepth(TreeNode<T> root){

		if (root == null) {
			return 0;
		}
		int left = getMaxDepth(root.left);
		int right = getMaxDepth(root.right);

		return 1 + Math.max(left, right);
	}

```
(2) 查找最小深度

```Java
/**
	 * 获取最小深度
	 * @param root
	 * @return
	 */
	public int getMinDepth(TreeNode<T> root) {

		if (root == null) {
			return 0;
		}

		if (root.left == null && root.right == null) {
			return 1;
		}
		return Math.min(getMinDepth(root.left), getMinDepth(root.right)) + 1;
	}
```

(3) 获取二叉树节点总数

```Java
/**
	 * 获取节点总数
	 * @param root
	 * @return
	 */
	public int totalNodes(TreeNode<T> root){
		if (root == null) {
			return 0;
		}
		int left = totalNodes(root.left);
		int right = totalNodes(root.right);

		return left + right + 1;
	}

```
(4) 获取叶子节点总数

```Java
/**
	 * 获取所有叶子节点
	 * @param root
	 * @return
	 */
	public int totalChildNodes(TreeNode<T> root){

		if (root == null) {
			return 0;
		}
		if (root.left == null && root.right == null) {
			return 1;
		}
		return totalChildNodes(root.left) + totalChildNodes(root.right);
	}
```
(5) 获取K层节点数

```Java
/**
	 * 获取K层节点数
	 * @param root
	 * @param k
	 * @return
	 */
	public int getLevelKNodes(TreeNode<T> root,int k){

		if (root == null || k < 1) {
			return 0;
		}
		if (k == 1) {
			return 1;
		}
		int left = getLevelKNodes(root.left, k-1);
		int right = getLevelKNodes(root.right, k-1);
		return left + right;
	}

```
(6) 层次遍历，每一层从左向右遍历

要点：

1.利用队列添加每一层节点，进行遍历

2.如果当前节点存在子节点，加到队列尾部，下次循环遍历取出

```Java
/**
	 * 层次遍历，每一层从左向右遍历
	 * @param root
	 * @return
	 */
	public List<List<T>> printTreeNodes(TreeNode<T> root) {

		List<List<T>> result = new ArrayList<List<T>>();
		if (root == null) {
			return result;
		}
		LinkedList<TreeNode<T>> linkedList = new LinkedList<TreeNode<T>>();
		linkedList.add(root);

		while (!linkedList.isEmpty()) {

			//用于装每一层的节点值
			ArrayList<T> row = new ArrayList<T>();

			//遍历每一层
			int size = linkedList.size();
			for (int i = 0; i < size; i++) {
				TreeNode<T> node = linkedList.removeFirst();
				row.add(node.getData());

				if (node.left != null) {
					linkedList.add(node.left);
				}
				if (node.right != null) {
					linkedList.add(node.right);
				}
			}
			result.add(row);
		}

		return result;
	}
```

(7) 计算二叉树最大宽度

要点：

过程同层次遍历类似，遍历每一层时，比较每一层的最大宽度，作为二叉树的最大宽度

```Java
/**
	 * 计算二叉树的最大宽度，通过使用队列计算，每一层的节点数即为每一层的宽度取出最大的一层的宽度
	 * @param root
	 * @return
	 */
	public int getBinaryTreeWidth(TreeNode<T> root){
		if (root == null) {
			return 0;
		}
		LinkedList<TreeNode<T>> queue = new LinkedList<TreeNode<T>>();
		queue.add(root);
		int width = 1;

		while (!queue.isEmpty()) {
			int size = queue.size();
			if (width < size) {
				width = size;
			}
			for (int i = 0; i < size; i++) {

				TreeNode<T> currentNode = queue.removeFirst();
				if (currentNode.left != null) {
					queue.add(currentNode.left);
				}
				if (currentNode.right != null) {
					queue.add(currentNode.right);
				}
			}
		}
		return width;
}
```

(8) 判断两个二叉树是否相同

```Java
/**
	 * 判断两个二叉树是否相同
	 * @param root1
	 * @param root2
	 * @return
	 */
	@SuppressWarnings("null")
	public boolean isSameTreeNode(TreeNode<T> root1,TreeNode<T> root2){

		if (root1 == null && root2 == null) {
			return true;
		}
		else if(root1 != null || root2 != null){
			return false;
		}
		else if (root1.data != root2.data) {
			return false;
		}

		boolean left = isSameTreeNode(root1.left, root2.left);
		boolean right = isSameTreeNode(root1.right, root2.right);
		return left && right;
	}
```

(9) 前序遍历（递归）

```Java
/**
	 * 前序遍历
	 * @param root
	 * @return
	 */
	public List<T> preOrder(TreeNode<T> root){

		List<T> list = new ArrayList<T>();
		if (root == null) {
			return list;
		}
		preOder(list, root);
		return list;
	}

	private void preOder(List<T> list,TreeNode<T> root){

		if (root == null) {
			return;
		}
		list.add(root.getData());
		preOder(list, root.left);
		preOder(list, root.right);

	}

```
(10) 后序遍历（递归）

```Java
/**
 * 后序遍历
 * @param root
 * @return
 */
public List<T> postOrder(TreeNode<T> root){

  List<T> list = new ArrayList<T>();
  if (root == null) {
    return list;
  }
  postOder(list, root);
  return list;
}

private void postOder(List<T> list,TreeNode<T> root){

  if (root == null) {
    return;
  }

  postOder(list, root.left);
  postOder(list, root.right);
  list.add(root.getData());

}
```

(11) 中序遍历（递归）

```Java
/**
 * 中序遍历
 * @param root
 * @return
 */
public List<T> InOrder(TreeNode<T> root){

  List<T> list = new ArrayList<T>();
  if (root == null) {
    return list;
  }
  InOder(list, root);
  return list;
}

private void InOder(List<T> list,TreeNode<T> root){

  if (root == null) {
    return;
  }

  InOder(list, root.left);
  list.add(root.getData());
  InOder(list, root.right);

}
```

### 总结

上面的算法使用基本上采用递归的方式，递归的思想在编程中还是很重要的，要注意慢慢培养，另外，要想很好的理解递归的过程，需要学习一下栈帧的概念，可以阅读《深入理解计算机系统》这本书，下篇将对二叉树其他算法进行练习，敬请期待！

### 参考

[二叉树常见面试题目(java实现)](https://www.jianshu.com/p/0190985635eb)
