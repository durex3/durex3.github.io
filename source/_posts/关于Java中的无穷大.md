---
title: 关于Java中的无穷大
date: 2019-09-18 13:11:50
categories:
- Java
tags:
- Java
- Infinity
---
最近笔试遇到一道题，没有做出来
<!-- more -->

### 题目内容如下：
```java
	double a = 60.00;
	double b = 0.00;
	System.out.println(a / b);
```
### 这段代码运行结果如下：
```java
	Infinity
```
### 原因：
&emsp;浮点型是不精确的，所以0.00并不等于0，是无限趋近于0。而根据高数中无穷大的理解，分母无限趋近于0，分子为常数，会得到无穷大。  
&emsp;Java中有一个特殊值：Infinity，它的意思就是正无穷大，同理-Infinity表示负无穷小。

