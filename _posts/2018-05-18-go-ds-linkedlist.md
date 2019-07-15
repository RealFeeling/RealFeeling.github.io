---
layout: post
title: Go源码解析之list
subtitle: '详解Golang中提供的Container包第一篇'
date: 2018-05-18
categories: Golang
tags: Golang 源码解析 Container 数据结构 list
---

# 链表
链表是一种非连续存储的容器，由多个节点组成，节点通过一些变量记录彼此之间的关系。列表有多种实现方法，如单链表、双链表等

列表的原理可以这样理解：假设 A、B、C 三个人都有电话号码，如果 A 把号码告诉给 B，B 把号码告诉给 C，这个过程就建立了一个单链表结构，如下图所示。

[img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31I54a30.jpg)

图：三人单向通知电话号码形成单链表结构

如果在这个基础上，再从 C 开始将自己的号码给自己知道号码的人，这样就形成了双链表结构，如下图所示

[img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31IJRI.jpg)

那么如果需要获得所有人的号码，只需要从 A 或者 C 开始，要求他们将自己的号码发出来，然后再通知下一个人如此循环。这个过程就是列表遍历

如果 B 换号码了，他需要通知 A 和 C，将自己的号码移除。这个过程就是列表元素的删除操作，如下图所示

[img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31J0524T.jpg)
