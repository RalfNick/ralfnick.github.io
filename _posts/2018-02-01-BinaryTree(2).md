---
layout: post
title: "BinaryTree(2)"
date: 2018-02-01
description: "二叉树"
tag: 数据结构
---

继上一篇[二叉树](https://ralfnick.github.io/2018/01/BinaryTree(1)/)之后，这次加深难度了，对前序、后序、中序遍历的迭代算法进行练习了一下，还有一些其他算法，那就来快看一下吧！

### 1. 前序遍历（迭代）

```java
/**
 * 前序遍历的迭代算法
 * 要点：使用栈来进行装载节点
 * @param root
 * @return
 */
public List<T> preOrderWithIteration(TreeNode<T> root){

	if (root == null) {
		return null;
	}

	List<T> list = new ArrayList<T>();
	Stack<TreeNode<T>> stack = new Stack<TreeNode<T>>();
	stack.push(root);
	while(!stack.isEmpty()){
		TreeNode<T> currentNode = stack.pop();
		list.add(currentNode.getData());

		if (currentNode.right != null) {
			stack.push(currentNode.right);
		}

		if (currentNode.left != null) {
			stack.push(currentNode.left);
		}
	}
	return list;
}
```

### 2. 中序遍历（迭代）

```java
/**
	 * 中序遍历的迭代算法
	 * @param root
	 * @return
	 */
	public List<T> inOrderWithIteration(TreeNode<T> root){

		if(root == null){
			return null;
		}
		List<T> list = new ArrayList<T>();
		Stack<TreeNode<T>> stack = new Stack<TreeNode<T>>();

		while(root != null || !stack.isEmpty()){

			while(root != null){
				stack.push(root);
				root = root.left;
			}
			if (!stack.isEmpty()) {

				TreeNode<T> currentNode = stack.pop();
				list.add(currentNode.getData());
				root = currentNode.right;
			}
		}
		return list;
 }
```

### 3. 后序遍历（迭代）

>思路：要保证根结点在左孩子和右孩子访问之后才能访问，因此对于任一结点P，先将其入栈。如果P不存在左孩子和右孩子，则可以直接访问它；或者P存 在左孩子或者右孩子， 但是其左孩子和右孩子都已被访问过了，则同样可以直接访问该结点。若非上述两种情况，则将P的右孩子和左孩子依次入栈，这样就保证了 每次取栈顶元素的时候，左孩子在右孩子前面被访问，左孩子和右孩子都在根结>点前面被访问。

```java
/**
	 * 后序遍历，非递归算法
	 * @param root
	 * @return
	 */
	public List<T> postOrderWithIteration(TreeNode<T> root){

		if(root == null){
			return null;
		}

		List<T> list = new ArrayList<T>();
		Stack<TreeNode<T>> stack = new Stack<TreeNode<T>>();
		TreeNode<T> curNode;
		TreeNode<T> preNode = null;
		stack.push(root);

		while(!stack.isEmpty()){

			curNode = stack.peek();
			if ((curNode.left == null && curNode.right == null) || (preNode != null
					&& (preNode == curNode.left || preNode == curNode.right))) {

				list.add(curNode.getData());
				preNode = curNode;
				stack.pop();
			}
			else {
				if (curNode.right != null) {
					stack.push(curNode.right);
				}
				if(curNode.left != null){
					stack.push(curNode.left);
				}
			}
		}
		return list;
	}

```
### 4. 判断两个二叉树是否镜像

```java
/**
	 * 判断两个二叉树是否镜像
	 * @param root1
	 * @param root2
	 * @return
	 */
	public boolean isMirror(TreeNode<T> root1,TreeNode<T> root2){

		if (root1 == null && root2 == null) {
			return true;
		}
		else if (root1 != null || root2 != null) {
			return false;
		}
		else if(root1.getData() != root2.getData()){
			return false;
		}

		return isMirror(root1.left, root2.right) && isMirror(root1.right, root2.left);

	}

```

### 5. 求镜像二叉树，或者是翻转二叉树

```java
/**
	 * 求镜像二叉树，或者是翻转二叉树
	 * @param root
	 * @return
	 */
	public TreeNode<T> mirrorTreeNode(TreeNode<T> root){
		if (root == null) {
			return null;
		}
		TreeNode<T> left = mirrorTreeNode(root.left);
		TreeNode<T> right = mirrorTreeNode(root.right);

		root.left = right;
		root.right = left;

		return root;
	}
```

### 6. 寻找连个节点的最小祖先节点

```java
/**
	 * 寻找连个节点的最小祖先节点
	 * @param root
	 * @param nodeA
	 * @param nodeB
	 * @return
	 */
	public TreeNode<T> getLowestAncestor(TreeNode<T> root,TreeNode<T> nodeA,TreeNode<T> nodeB){
		if (findNode(root.left,nodeA)) {
			if (findNode(root.right, nodeB)) {
				return root;
			}
			else {
				return getLowestAncestor(root.left, nodeA, nodeB);
			}

		}
		else {
			if (findNode(root.left, nodeB)) {
				return root;
			}
			else {
				return getLowestAncestor(root.right, nodeA, nodeB);
			}
		}

	}

	private boolean findNode(TreeNode<T> root, TreeNode<T> node) {

		if (root == null || node == null) {
			return false;
		}
		if (root == node) {
			return true;
		}
		boolean isFound = findNode(root.left, node);
		if (!isFound) {
			isFound = findNode(root.right, node);
		}
		return isFound;
	}
```

### 7. 获取二叉树两个节点之间最大距离

> 思路：二叉树中两个节点的最长距离可能有三种情况：
>
>1.左子树的最大深度+右子树的最大深度为二叉树的最长距离
>
> 2.左子树中的最长距离即为二叉树的最长距离
>
> 3.右子树种的最长距离即为二叉树的最长距离



```java
	/**
	 * 获取二叉树两个节点之间的最大距离
	 * @param root
	 * @return
	 */
	public int getMaxDistance(TreeNode<T> root){

		return getMaxDistanceHolder(root).maxDistance;
	}

	private DistanceHolder getMaxDistanceHolder(TreeNode<T> root){

		if (root == null) {
			return new DistanceHolder(-1,0);
		}
		DistanceHolder leftHolder = getMaxDistanceHolder(root.left);
		DistanceHolder rightHolder = getMaxDistanceHolder(root.right);

		DistanceHolder result = new DistanceHolder();
		result.maxDepth = Math.max(leftHolder.maxDepth, rightHolder.maxDepth) + 1;
		result.maxDistance = Math.max(leftHolder.maxDepth + rightHolder.maxDepth,
				Math.max(leftHolder.maxDistance, rightHolder.maxDistance));

		return result;
	}

	/**
	 * 用来持有距离的最大深度的类
	 * @author Administrator
	 *
	 */
	private static class DistanceHolder{

		public int maxDistance;
		public int maxDepth;

		public DistanceHolder(){}
		public DistanceHolder(int maxDistance, int maxDepth) {
			this.maxDistance = maxDistance;
			this.maxDepth = maxDepth;
		}

	}
```

### 8. 输入一个二叉树和一个整数，打印出二叉树中节点值的和等于输入整数所有的路径

>注意：这里的路径是指从根节点到叶子节点

```java
/**
 *
 * @param root
 * @param k
 * @return
 */
public List<Integer> findAllPath(TreeNode<Integer> root,int k){
	List<Integer> list = new ArrayList<Integer>();
	if (root == null) {
		return list;
	}
	int sum = 0;
	Stack<TreeNode<Integer>> stack = new Stack<TreeNode<Integer>>();
	findAllPath(root, k,list,stack,sum);
	return list;
}

private void findAllPath(TreeNode<Integer> root, int k, List<Integer> list,
		Stack<TreeNode<Integer>> stack, int sum) {

	sum += root.getData();
	stack.push(root);
	if (root.left == null && root.right == null && sum == k) {

		for(TreeNode<Integer> node : stack){
			list.add(node.getData());
		}
	}

	if (root.left != null) {
		findAllPath(root.left, k, list, stack, sum);
	}
	if (root.right != null) {
		findAllPath(root.right, k, list, stack, sum);
	}

	stack.pop();//这里每次要弹出栈，因为到这里已经不满足条件了，需要将本次检索的元素清除掉
}

```

### 总结

以上就是本次练习的二叉树几道算法题，当然，还有其他实现的方式，如后序遍历的迭代算法还有其他思路，这里我就给出我认为比较容易理解的思路，仅做参考，如果你有比较好的思路和其他写法，可以分享出来！

下一次练习，就开始对二叉树查找树进行练习，敬请期待！

### 参考
[一篇文章搞定面试中的二叉树题目(java实现)](https://www.jianshu.com/p/0190985635eb)
