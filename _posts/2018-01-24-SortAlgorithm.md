---
layout: post
title: "SortAlgorithm"
date: 2018-01-24
description: "排序算法"
tag: 算法
---

之前一篇练习数据结构中的二叉树-[BinaryTree](https://ralfnick.github.io/2018/01/BinaryTree(1)/)，本篇来点——排序算法，调调味，都是基本的排序算法中。

### 1. 冒泡排序

```java
/**
	 * 冒泡排序
	 * @param arr
	 */
public static void bubbleSort(int[] arr){

	for (int i = 0; i < arr.length-1; i++) {
		for (int j = 0; j < arr.length-i-1; j++) {
			if (arr[j] > arr[j+1]) {
				swap(arr, j, j+1);
			}
		}
	}
}
```

### 2. 插入排序

```java
/**
	 * 插入排序
	 * 要点：像扑克牌一样，小牌放在左边，插入的牌比左边的某张牌大（左边的牌已经排好大小），
	 * 该张牌后面的牌依次向右移动一个位置
	 * @param arr
	 */
public static void insertSort(int[] arr){

	for (int i = 1; i < arr.length; i++) {

		int temp = arr[i];
		int j = i - 1;//要插入的牌和前面的所有牌比较
		while(j >= 0 && arr[j] > temp){
			arr[j+1] = arr[j];//向右移动一位
			j--;
		}
		arr[j+1] = temp;//注意：j+1，因为循环中j--最后又执行一次了
	}
}
```

### 3. 选择排序

```java
/**
	 * 选择排序
	 * 每次选择一个最小（或最大）的数，拿走，再从剩下的数中重复选择最小（或最大）的数
	 * @param arr
	 */
public static void selectSort(int[] arr){

	for (int i = 0; i < arr.length - 1; i++) {
		int min = i;//每次假定第一个数最小
		for (int j = i+1; j < arr.length; j++) {
			if (arr[j] < arr[min]) {
				min = j;
			}
		}
		swap(arr, i, min);
	}
}

```
### 4. 快速排序

原理部分可以看这边文章，讲得很好，当时还真是坐在马桶上看的这篇文章，哈哈。

[坐在马桶上看算法：快速排序](http://developer.51cto.com/art/201403/430986.htm)

注意点：

* 如果基轴选的是左边的点，那么就应该从右边先开始遍历

* 左边是要找到一个比基轴大的数，对应程序中遍历左边时，注意要加上“=”，因为第一个数与基轴相同

```java
/**
	 * 快速排序算法
	 * @param arr
	 * @param left
	 * @param right
	 */
	public static void quickSort(int[] arr, int left, int right) {

		if (left > right) {
			return;
		}
		int index = potrit(arr,left,right);
		quickSort(arr, left, index - 1);
		quickSort(arr, index + 1, right);
	}

	private static int potrit(int[] arr, int left, int right) {

		int i = left;
		int j = right;
		int pivot = arr[left];
		while(i != j){

			while(arr[j] >= pivot && i < j){
				j--;
			}

			//注意有 =，因为从第一个开始，所以要加等号，第一个与左边的pivot相等
			while(arr[i] <= pivot && i < j){
				i++;
			}

			if (i < j){
				swap(arr, i, j);
			}

		}
		arr[left] = arr[i];
		arr[i] = pivot;

		return i;
	}

```
思考：为啥选择左边第一个数为基轴，就要从右边先开始遍历？

> 也就是两个while的顺序是不能改变的，假设对如下进行排序：
>
> 6--1--2--7--9
>
>6在左，9在右  我们将6作为基轴。
>假设从左边开始（与正确程序正好相反）于是 i 就会移动到现在的数字 7 那个位置停下来，而 j 原来在数字 9 那个位置 ，因为有 i<j 条件,于是，j 也会停留在数字 7 那个位置，于是问题来了。当你最后交换基数 6 与 7 时，就错了。问题在于当我们先从在边开始时，那么 i 所停留的那个位置肯定是大于基数6的，而在上述例子中，为了满足 i<j 于是 j也停留在 7 的位置,但最后交换回去的时候，7就到了左边，因为我们原本交换后数字 6 左边应该是全部小于6，右边全部大于 6，结果出现大于 6 的数字 7 就不行了。

**所以，如果你选定左边的数为基轴，一定要从右边先开始，只有从右边才是要找到比基轴小的数，而左边是要找到比基轴大的数。**

### 5. 交换

```java
/**
 * 交换数组中两个位置的数值
 * @param arr
 * @param i
 * @param j
 */

public static void swap(int[] arr,int i,int j){
  int temp = arr[i];
  arr[i] = arr[j];
  arr[j] = temp;
}
```

### 总结

（1）这几种基本的排序算法，在实际中快速排序算法应用最多，效率最高，当然也相对更复杂，前面的三种也要理解，虽然可能用不到，但是作为练习，提高逻辑思维能力还是很有必要的。

（2）现在我有时还容易写错快速排序算法，所以多练习，多练习，多练习！

### 快速排序变形
当需要排序的数据很多时，可以将基轴选取为中间值，这样也能够提高效率。

```java
/**
	 * 快速排序算法
	 * @param arr
	 * @param left
	 * @param right
	 */
	public static void quickSort(int[] arr, int left, int right) {

		if (left > right) {
			return;
		}
		int index = potrit(arr,left,right);
		quickSort(arr, left, index - 1);
		quickSort(arr, index + 1, right);
	}

	private static int potrit(int[] arr, int left, int right) {

		int mid = left + (right-left)/2;
		if (arr[mid] > arr[right]) {
			swap(arr,mid,right);
		}
		if (arr[left] > arr[right]) {
			swap(arr,left,right);
		}
		if (arr[mid] > arr[left]) {
			swap(arr,mid,left);
		}

		int i = left;
		int j = right;
		int pivot = arr[left];
		while(i != j){

			while(arr[j] >= pivot && i < j){
				j--;
			}

			//注意有 =，因为从第一个开始，所以要加等号，第一个与左边的pivot相等
			while(arr[i] <= pivot && i < j){
				i++;
			}

			if (i < j){
				swap(arr, i, j);
			}

		}
		arr[left] = arr[i];
		arr[i] = pivot;

		return i;
	}

```
