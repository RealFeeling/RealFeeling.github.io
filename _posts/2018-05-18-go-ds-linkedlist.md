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

![img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31I54a30.jpg)

<center>图：三人单向通知电话号码形成单链表结构</center>

如果在这个基础上，再从 C 开始将自己的号码给自己知道号码的人，这样就形成了双链表结构，如下图所示

![img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31IJRI.jpg)

<center>图：三人相互通知电话号码形成双链表结构</center>

那么如果需要获得所有人的号码，只需要从 A 或者 C 开始，要求他们将自己的号码发出来，然后再通知下一个人如此循环。这个过程就是列表遍历

如果 B 换号码了，他需要通知 A 和 C，将自己的号码移除。这个过程就是列表元素的删除操作，如下图所示

![img](http://c.biancheng.net/uploads/allimg/180813/1-1PQ31J0524T.jpg)

<center>图：从双链表中删除一人的电话号码</center>

在 Go 语言中，链表使用 container/list 包来实现，内部的实现原理是双链表。列表能够高效地进行任意位置的元素插入和删除操作

# list 使用
## 1. 初始化
list 的初始化有两种方法：New 和声明。两种方法的初始化效果都是一致的。

1) 通过 container/list 包的 New 方法初始化 list
```go
变量名 := list.New()
```

2) 通过声明初始化list
```go
var 变量名 list.List
```

list 与 slice 和 map 不同的是，列表并没有具体元素类型的限制。因此，列表的元素可以是任意类型。这既带来便利，也会引来一些问题。

给一个列表放入了非期望类型的值，在取出值后，将 interface{} 转换为期望类型时将会发生panic。

## 2. 插入元素

双链表支持从队列前方或后方插入元素，分别对应的方法是 PushFront 和 PushBack

> 这两个方法都会返回一个```*list.Element``` 结构。如果在以后的使用中需要删除插入的元素，则只能通过```*list.Element``` 配合``` Remove()``` 方法进行删除，这种方法可以让删除更加效率化，也是双链表特性之一

下面代码展示如何给 list 添加元素：

```go
l := list.New()
l.PushBack("fist")
l.PushFront(67)
```

插入元素的方法如下表所示：


方  法 |	功  能
-|-
```InsertAfter(v interface {}, mark * Element) * Element```	| 在 mark 点之后插入元素，mark 点由其他插入函数提供
```InsertBefore(v interface {}, mark * Element) *Element``` |	在 mark 点之前插入元素，mark 点由其他插入函数提供
```PushBackList(other *List)``` |	添加 other 列表元素到尾部
```PushFrontList(other *List)``` |	添加 other 列表元素到头部

## 3. 删除元素
链表的插入函数的返回值会提供一个 *list.Element 结构，这个结构记录着列表元素的值及和其他节点之间的关系等信息。从列表中删除元素时，需要用到这个结构进行快速删除

```go
package main
import "container/list"
func main() {
    l := list.New()
    // 尾部添加
    l.PushBack("canon")
    // 头部添加
    l.PushFront(67)
    // 尾部添加后保存元素句柄
    element := l.PushBack("fist")
    // 在fist之后添加high
    l.InsertAfter("high", element)
    // 在fist之前添加noon
    l.InsertBefore("noon", element)
    // 使用
    l.Remove(element)
}
```

## 4. 遍历链表
遍历双链表需要配合 Front() 函数获取头元素，遍历时只要元素不为空就可以继续进行。每一次遍历调用元素的 Next
```go
l := list.New()
// 尾部添加
l.PushBack("canon")
// 头部添加
l.PushFront(67)
for i := l.Front(); i != nil; i = i.Next() {
    fmt.Println(i.Value)
}
```

# 源码解析
## 实现原理
```go
type Element struct {
	// Next and previous pointers in the doubly-linked list of elements.
	// To simplify the implementation, internally a list l is implemented
	// as a ring, such that &l.root is both the next element of the last
	// list element (l.Back()) and the previous element of the first list
	// element (l.Front()).
	next, prev *Element

	// The list to which this element belongs.
	list *List

	// The value stored with this element.
	Value interface{}
}
```

上面是链表中单个节点的定义Element，其中包括：
1. 指向上一个节点的指针prev

2. 指向下一个节点的指针next

3. 该节点存储的数据value

4. 所在的链表list

[![Z7RF41.png](https://s2.ax1x.com/2019/07/16/Z7RF41.png)](https://imgchr.com/i/Z7RF41)

根据节点的定义可以得出以上示意图，各个节点之间有两个引用分别指向上一个节点和下一个节点，由此可看到该链表可以在左边第一个元素或向右进行遍历，
也可以在右边第一个元素向左开始遍历

这时便有一个问题，怎么记录该链表的的头节点和尾节点？这就需要用到我们上面提过的list了

先来看下List的定义：
```go
// List represents a doubly linked list.
// The zero value for List is an empty list ready to use.
type List struct {
	root Element // sentinel list element, only &root, root.prev, and root.next are used
	len  int     // current list length excluding (this) sentinel element
}
```

List 是一个相当简单的结构体，其中只包含了一个root的节点和一个整型的len。一个List代表一条链表，而len就是该链表的元素个数

root是一个虚拟节点，即root这个节点不包含在链表元素内，它仅仅是用来记录链表的头和尾的一个辅助节点，加上了虚拟节点的链表如下图所示

[![Z7WcWt.png](https://s2.ax1x.com/2019/07/16/Z7WcWt.png)](https://imgchr.com/i/Z7WcWt)

由上图可以看到，通过一个root节点便将链表的头尾连接起来，实现了一个循环链表。通过获取root的prev和next就能快速地获取链表的头、尾两个节点，这两个操作都是O(1)复杂度

```go
// Init initializes or clears list l.
func (l *List) Init() *List {
	l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}

// New returns an initialized list.
func New() *List { return new(List).Init() }
```

使用New函数去初始化一个List链表，该函数会new一个List，返回的指针调用了Init方法。从该方法我们可以看到，初始化是root节点的prev和next都是指向自身

## 插入节点

[![Z7fj3t.png](https://s2.ax1x.com/2019/07/16/Z7fj3t.png)](https://imgchr.com/i/Z7fj3t)

假如我们现在要插入3这个节点到2这个节点的后面，则只需改变2的next和5的prev都指向新添加的元素，新元素的prev指向2，next指向5即可。最后再维护一下list的len

下面是和插入相关的方法， 主要围绕insert这个方法扩展：
```go
// insert inserts e after at, increments l.len, and returns e.
func (l *List) insert(e, at *Element) *Element {
	n := at.next
	at.next = e
	e.prev = at
	e.next = n
	n.prev = e
	e.list = l
	l.len++
	return e
}

// insertValue is a convenience wrapper for insert(&Element{Value: v}, at).
func (l *List) insertValue(v interface{}, at *Element) *Element {
	return l.insert(&Element{Value: v}, at)
}

// PushFront inserts a new element e with value v at the front of list l and returns e.
func (l *List) PushFront(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, &l.root)
}

// PushBack inserts a new element e with value v at the back of list l and returns e.
func (l *List) PushBack(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, l.root.prev)
}

// InsertBefore inserts a new element e with value v immediately before mark and returns e.
// If mark is not an element of l, the list is not modified.
// The mark must not be nil.
func (l *List) InsertBefore(v interface{}, mark *Element) *Element {
	if mark.list != l {
		return nil
	}
	// see comment in List.Remove about initialization of l
	return l.insertValue(v, mark.prev)
}

// InsertAfter inserts a new element e with value v immediately after mark and returns e.
// If mark is not an element of l, the list is not modified.
// The mark must not be nil.
func (l *List) InsertAfter(v interface{}, mark *Element) *Element {
	if mark.list != l {
		return nil
	}
	// see comment in List.Remove about initialization of l
	return l.insertValue(v, mark)
}
```

## 删除元素
了解了添加元素的原理后，删除元素就相对来说比较简单了，只需要将要删除元素的上一个元素的next指向下一个元素，同时将要删除元素的下一个元素的prev指向上一个元素即可。同时别忘了将删除元素的指向都设置为nil，并维护一下list的len
```go
// remove removes e from its list, decrements l.len, and returns e.
func (l *List) remove(e *Element) *Element {
	e.prev.next = e.next
	e.next.prev = e.prev
	e.next = nil // avoid memory leaks
	e.prev = nil // avoid memory leaks
	e.list = nil
	l.len--
	return e
}
```

## 移动元素
[![Z7WcWt.png](https://s2.ax1x.com/2019/07/16/Z7WcWt.png)](https://imgchr.com/i/Z7WcWt)

如果我们需要将上图中的元素5移动到链表头，则需要进两个步骤

第一步：将元素5从链表中移出

[![Z7IVgI.png](https://s2.ax1x.com/2019/07/16/Z7IVgI.png)](https://imgchr.com/i/Z7IVgI)

第二步：将元素5插入到链表头

[![Z7IG2n.png](https://s2.ax1x.com/2019/07/16/Z7IG2n.png)](https://imgchr.com/i/Z7IG2n)

移动元素的源码如下：

```go
// move moves e to next to at and returns e.
func (l *List) move(e, at *Element) *Element {
	if e == at {
		return e
	}
	e.prev.next = e.next
	e.next.prev = e.prev

	n := at.next
	at.next = e
	e.prev = at
	e.next = n
	n.prev = e

	return e
}
```

扩展的方法：
```go
// MoveToFront moves element e to the front of list l.
// If e is not an element of l, the list is not modified.
// The element must not be nil.
func (l *List) MoveToFront(e *Element) {
	if e.list != l || l.root.next == e {
		return
	}
	// see comment in List.Remove about initialization of l
	l.move(e, &l.root)
}

// MoveToBack moves element e to the back of list l.
// If e is not an element of l, the list is not modified.
// The element must not be nil.
func (l *List) MoveToBack(e *Element) {
	if e.list != l || l.root.prev == e {
		return
	}
	// see comment in List.Remove about initialization of l
	l.move(e, l.root.prev)
}

// MoveBefore moves element e to its new position before mark.
// If e or mark is not an element of l, or e == mark, the list is not modified.
// The element and mark must not be nil.
func (l *List) MoveBefore(e, mark *Element) {
	if e.list != l || e == mark || mark.list != l {
		return
	}
	l.move(e, mark.prev)
}

// MoveAfter moves element e to its new position after mark.
// If e or mark is not an element of l, or e == mark, the list is not modified.
// The element and mark must not be nil.
func (l *List) MoveAfter(e, mark *Element) {
	if e.list != l || e == mark || mark.list != l {
		return
	}
	l.move(e, mark)
}

```

至此，list的源码就基本分析完了，本文相对简单，比较适合初学者学习
