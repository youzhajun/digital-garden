---
title: ArryaList
draft: false
tags:
  - "#java"
  - "#java核心"
date: 2021-08-02
---
[[00.java核心]]
# ArrayList 是如何添加元素的

通过 `add()` 方法添加元素，具体的执行步骤如下：
```md
add(value)
└── if (size == elementData.length) // 判断是否需要扩容 
├── grow(minCapacity) // 扩容 
│    └── newCapacity = oldCapacity + (oldCapacity >> 1) // 计算新的数组容量 
│    └── Arrays.copyOf(elementData, newCapacity) // 创建新的数组 
├── elementData[size++] = element; // 添加新元素 
└── return true; // 添加成功
```


# ArrayList 的扩容机制

机制概述：ArrayList 使用动态数组实现，当添加元素时容量不足会自动扩容。**默认扩容规则为增加原容量的 50%**（即扩容至 1.5 倍），特殊场景（如初始空数组、超大容量）有额外处理。

首先看一下构造方法：

```java
/**
 * 默认初始容量大小
 */
private static final int DEFAULT_CAPACITY = 10;

// 存储内容的数组对象
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 默认构造函数，使用初始容量10构造一个空列表(无参数构造)
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * 带初始容量参数的构造函数。（用户自己指定容量）
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {//初始容量大于0
        //创建initialCapacity大小的数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {//初始容量等于0
        //创建空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
	    //初始容量小于0，抛出异常
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}


/**
 *构造包含指定collection元素的列表，这些元素利用该集合的迭代器按顺序返回
 *如果指定的集合为null，throws NullPointerException。
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

**以无参数构造方法创建 `ArrayList` 时，实际上初始化赋值的是一个空数组。当真正对数组进行添加元素操作时，才真正分配容量。即向数组中添加第一个元素时，数组容量扩为 10**

1. 添加元素触发扩容方法
	
	```java
		public boolean add(E e) {
		    modCount++;
		    add(e, elementData, size);  // 核心添加方法
		    return true;
		}
		
		private void add(E e, Object[] elementData, int s) {
		    if (s == elementData.length)  // 当元素数量达到数组容量
		        elementData = grow();     // 触发扩容
		    elementData[s] = e;
		    size = s + 1;
		}
	```

2.   扩容核心方法 `grow()`
		```java
			public boolean add(E e) {
			    modCount++;
			    add(e, elementData, size);  // 核心添加方法
			    return true;
			}
			
			private void add(E e, Object[] elementData, int s) {
			    if (s == elementData.length)  // 当元素数量达到数组容量
			        elementData = grow();     // 触发扩容
			    elementData[s] = e;
			    size = s + 1;
			}
		```

3. 容量计算规则

	```java
		public static int newLength(int oldLength, int minGrowth, int prefGrowth) {
		    int newLength = oldLength + Math.max(minGrowth, prefGrowth); // 基础计算
		    if (newLength <= MAX_ARRAY_LENGTH) return newLength;
		    return hugeLength(oldLength, minGrowth); // 处理超大容量
		}
	```


