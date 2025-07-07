---
title: 伸缩盒模型 - flex
draft: false
tags:
  - 前端
date: 2025-01-31
---
 
# 简介

2009年，W3C提出了一种新的方案—-Flex布局，可以简便、完整、响应式地实现各种页面布局。目前，它已经得到了所有浏览器的支持，这意味着，现在就能很安全地使用这项功能。

# 伸缩容器、伸缩项目

`display: flex` 将父元素设置为伸缩容器，内部的子元素自动变为伸缩项目。

- 一个元素既可以是伸缩容器，也可以是伸缩项目
- 伸缩项目无论之前是哪种元素，一旦成为伸缩项目后，全都会‘块状化‘（可设置宽高）

<aside> 💡

个人理解： 1、通过 display: flex 可以将元素转换为‘伸缩容器’，其内部的子元素（不含孙元素）都变成了伸缩项目。 2、‘伸缩容器’的一些属性可以控制子元素（伸缩项目）的排列方式，比如说：控制

</aside>

# 主轴

## 主轴方向

flex 布局方向分为主轴、侧轴。默认情况下 主轴为 x 轴 横**左向右**排列，侧轴为 y 轴 纵向**上至下**排列。

- 元素排列是按照主轴方向排列的
- flex-direction 属性设置主轴方向，有以下几个值
    - flex-direction: row
    - flex-direction: row-reverse
    - flex-direction: column
    - flex-direction: column-reverse

## 主轴上的换行方式

flex布局主轴方向上默认不换行，元素宽度超过容器宽度后会挤压自身宽度。

通过 `flex-wrap` 属性设置换行方式，`flex-wrap` 默认值为 `no-wrap`

- flex-wrap: no-wrap
- flex-wrap: wrap
- flex-wrap: wrap-reverse

## 主轴上的对齐方式

对齐方式，控制容器内元素的左右对齐方式。通过 `justify-content` 控制，排列的方向是由 flex-wrap 决定，所以对齐方式只能是主轴的开始、主轴的结束

- `justify-content: flex-start`
- `justify-content: flex-end`
- `justify-content: center`
- `justify-content: space-around`。项目均匀分布在一行中，项目之间的距离是项目与项目距边缘的2倍。
- `justify-content: space-between` （常用）。项目均匀分布在一行，两边项目紧贴边缘。
- `justify-content: space-evenly`。 项目均匀分布在一行中。

![[伸缩盒模型 - flex-1751872373287.png|704x636]]
![image.png|1147x24](https://prod-files-secure.s3.us-west-2.amazonaws.com/04789627-9321-4adf-aac5-34bcb0c91ad5/94f471c0-8cb0-4074-9819-cdc54451cf8e/image.png)

# 侧轴对齐

## 侧轴对齐-单行情况

align-items: flex-start（最常用）

align-items: flex-end

align-items: center

align-items: baseline

align-items: stretch。拉伸到整个父容器，前提伸缩容器不能给高度。（默认值）

![[伸缩盒模型 - flex-1751872352081.png|652x593]]
## 侧轴对齐-多行情况

`align-content: flex-start` 与交叉轴的起点对齐。

`align-content: flex-end` 与交叉轴的终点对齐。

`align-content: center` 与交叉轴的中点对齐。

`align-content: space-around` 。每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍。

`align-content: space-between` 与交叉轴两端对齐，轴线之间的间隔平均分布。

`align-content: space-evenly;` 均分剩余空间

`align-content: stretch` 默认值。轴线占满整个交叉轴。

![[伸缩盒模型 - flex-1751872342487.png|772x686]]

# 伸缩盒模型

## 拉伸

伸缩项目添加 `flex-grow` 属性，属性值为数值，代表该项目在容器中所占的份数。默认值为0.

举个例子：

1. 若所有伸缩项目的 flex-grow 都为 1， 则他们瓜分剩余空间
2. 若三个伸缩项目的 flex-grow 分别为1、2、3，则他们分别占据剩余空间的1/6、2/6、 3/6

## 压缩

前提容器 flex-wrap 的值需要是 no-wrap。

flex-shrink 定义了伸缩项目的压缩比例，默认为1。伸缩项目的计算举个例子：

```css
3个伸缩项目，200px、300px、200px， flex-shrink 分别为 1、2、3
则，父容器宽度为 700px 时可以完美放下3个伸缩项目，但是 当父容器小于700时，伸缩项目开始缩小。

计算步骤如下：
第一步，计算分母，每一份的长度 * flex-shrink 值
	200*1 + 300*2 + 200*3 = 1400px
	
第二步，分别计算伸缩项目占分母的比例
	项目1： 200*1/1400 = 比例1
	项目2： 300*2/1400 = 比例2
	项目3： 200*3/1400 = 比例3
	
第三步，计算最终的收缩大小，比例值*缩小的空间
	项目1： 比例1 * 缩小空间
```

# flex 复合属性

flex 属性 复合了 `flex-grow`、`flex-wrap`、`flex-basis（基准长度）`

- flex: 1 1 auto; 可拉伸，值为1、可压缩 值为1、基准长度auto
- flex: 1 1 0; 可简写为 flex: 1
- flex: 0 0 auto; 可简写为 flex: none
- flex: 0 1 auto; 可简写为 flex: 0 auto;

# 项目排序与单独对齐

`order` 属性可以控制伸缩元素原有排序，值越小越靠前。

伸缩项目通过 `align-self` 属性调整自身在主轴方向的对齐方式