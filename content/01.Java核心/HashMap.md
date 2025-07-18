---
title: HashMap
draft: false
tags:
  - java
  - java核心
date: 2021-08-06
---

# HashMap中的属性
- 默认容量 
	- `static final int *DEFAULT_INITIAL_CAPACITY* = 1 << 4; *// aka 16*`
- 数据存储 
	- `transient Node<K,V>[] table;`
- 负载因子（后续会讲解如何理解） 
	- `static final float *DEFAULT_LOAD_FACTOR* = 0.75f;`
- 最大容量 
	- `static final int *MAXIMUM_CAPACITY* = 1 << 30;`
- 链表转红黑树的阈值（后续会讲解如何理解） 
	- `static final int *TREEIFY_THRESHOLD* = 8;`
- 链表转红黑树的最小数组长度（后续会讲解如何理解） 
	- `static final int *MIN_TREEIFY_CAPACITY* = 64;`
- 红黑树退化为链表的阈值 
	- `static final int *UNTREEIFY_THRESHOLD* = 6;`


# HashMap 的数据结构是怎样的？

- `JKD8` 之前：**数组 + 链表**
- `JDK8` 之后：**数组 + 链表 + 红黑树**
```java

transient Node<K,V>[] table; // 数组

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next; // 链表结构 记录下一个元素
}

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent, left, right, prev; // 红黑树结构
}
```

# HashMap 如何计算键值对的存储位置？

1. **计算键（`key`）的哈希码**
	1. 调用 `key` 的 `hashcode()` 方法，得到一个 `int` 类型的32位的哈希值
2. **扰动函数处理（减少 `hash` 冲突）**
	- 目的：原始哈希码的高位信息可能未被充分利用，导致冲突
	- 操作：将哈希码的高16位与低16位进行异或运算 
		```java
			int perturbedHash = key.hashCode() ^ (key.hashCode() >>> 16);
		```

3. **计算数组索引位置
	```java
		int index = (array.length - 1) & perturbedHash;
	```

4. **解决哈希冲突**
	- 若多个键计算出的索引相同（哈希冲突）
		1. 链表存储：在数组的该位置维护一个链表，新节点插入链表尾部（Java 7）或头部（Java 8）。
		2. 链表转红黑树存储：当链表长度 ≥ 8 且数组长度 ≥ 64 时，链表转为红黑树（提高查询效率）。

# HashMap 何时扩容？扩容过程是怎样的？

扩容的触发条件：当元素数量 >  `容量（默认16） * 负载因子（默认为0.75）`
扩容步骤：
 1. 创建新数组（大小为原数组的 **2 倍**）。
 2. **重新哈希**：遍历旧数组，将节点迁移到新数组。
 
>源码：

```java
	Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab; // 指向新数组
	// 迁移节点
	for (int j = 0; j < oldCap; ++j) {
	    Node<K,V> e;
	    if ((e = oldTab[j]) != null) {
	        oldTab[j] = null;
	        if (e.next == null) // 单个节点
	            newTab[e.hash & (newCap - 1)] = e;
	        else if (e instanceof TreeNode) // 树节点
	            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
	        else { // 链表节点
	            Node<K,V> loHead = null, hiHead = null;
	            do {
	                // 判断是否需要移动（高位是否为1）
	                if ((e.hash & oldCap) == 0) {
	                    // 保留在原下标 j
	                } else {
	                    // 移动到新下标 j + oldCap
	                }
	            } while ((e = e.next) != null);
	        }
	    }
	}
```

# 讲一下HashMap中的树化与退化

- 树化：hashMap 中的链表转为红黑树的过程
- 退化：hashMap 中的红黑树退化成链表的过程

## 树化

### 概念

树化是指当某个桶（数组位置）上的链表长度**超过一定阈值**时，HashMap会将这个链表结构**转换**成一颗**红黑树（Red-Black Tree）** 结构。
### 原因

红黑树是一种**自平衡的二叉查找树**。它在最坏情况下也能保证查找、插入、删除操作的时间复杂度为**O(log n)**。当链表非常长时（n很大），O(log n) 的性能远优于 O(n)。

### 触发条件

必须同时满足以下条件：
1. 链表长度 >= 8
2.  **HashMap的数组长度 >= MIN_TREEIFY_CAPACITY (默认是64)**

> 这样做的原因？

- 链表长度 >= 8。
	- 元素少的时候链表的劣势并没有那么明显，并且他也有自己的优势（节省内存空间）
- **HashMap的数组长度 >= MIN_TREEIFY_CAPACITY (默认是64)**。 
	- 如果数组本身很小（比如只有16），哈希冲突相对集中，此时应该优先考虑的是**扩容（resize）** 来分散节点，而不是立即树化。扩容通常能更有效地解决冲突。只有当数组已经比较大（>=64）且某个桶链表确实过长（>=8）时，树化的收益才更明显。

### 树化的过程

1. 遍历当前通上的链表节点
2. 将每个`Node`节点（链表节点）转换成`TreeNode`节点（树节点）。
3. 根据键（`key`）的`hashCode`和`compareTo`方法（如果键实现了`Comparable`接口）或`System.identityHashCode`等方式构建一颗**红黑树**。

## 退化

 退化是树化的逆过程。它是指当某个桶上的红黑树节点数量**减少到一定阈值以下**时，HashMap会将这颗红黑树**转换回**普通的链表结构。

### 原因

- 当节点数较少时，链表结构比红黑树结构更节省内存。这也是为什么 `hashmap` 初始状态下是 数组+链表而不是直接用 `数组+红黑树`
- 对于非常小的链表（比如只有2-6个节点），链表遍历的O(n)和红黑树操作的O(log n)在实际性能上差别不大，甚至链表可能更简单高效。
-  在扩容（`resize`）时，处理链表比处理树结构通常更简单直接。

### 触发条件

 主要发生在**删除节点（`remove`）** 或**扩容（`resize`）** 过程中。
 - 在删除树节点后，如果当前树的根节点、根节点的左子节点、根节点的右子节点、根节点的左孙子节点（`root.left.left`）中**有一个为`null`**（这通常意味着树的节点数变得非常少，不足以维持一个高效的红黑树结构），则会检查是否满足退化条件。
- 更直接的阈值判断：在`removeTreeNode`方法和`resize`方法的分裂树逻辑中，当树中的节点数 **<= UNTREEIFY_THRESHOLD (默认是6)** 时，就会触发退化。

### 退化过程

1. 遍历红黑树节点
2. 将每个`TreeNode`节点转换回普通的`Node`节点。
3. 按照遍历顺序，将这些`Node`节点重新连接成一个单向链表
4. 将桶的头指针指向新链表的头节点。


# HashMap 在多线程情况下会出现的问题

- 多线程会出现死循环，导致 cpu 100%
- 共享 `map` 线程安全问题
- `put` 与 `get` 操作并发操作时，可能 `get` 的值为 `null`


