---
layout: post
title: Go实现泛型比较
subtitle: '通过Comparator比较器实现泛型比较，有助于在数据结构中对元素进行排序'
date: 2018-05-03
categories: Golang
tags: Golang 泛型 排序
---

# 空接口实现泛型
空接口是接口类型的特殊形式，空接口没有任何方法，因此任何类型都无须实现空接口。从实现的角度看，任何值都满足这个接口的需求。因此空接口类型可以保存任何值，也可以从空接口中取出原值

> 空接口类型类似于 C# 或 Java 语言中的 Object、C语言中的 void*、C++ 中的 std::any。在泛型和模板出现前，空接口是一种非常灵活的数据抽象保存和使用的方法。

空接口的内部实现保存了对象的类型和指针。使用空接口保存一个数据的过程会比直接用数据对应类型的变量保存稍慢。因此在开发中，应在需要的地方使用空接口，而不是在所有地方使用空接口

## 空接口的赋值

```go
var any interface{}
any = 1
fmt.Println(any)
any = "hello"
fmt.Println(any)
any = false
fmt.Println(any)
```
代码输出如下：

1

hello

false

## 从空接口获取值

保存到空接口的值，如果直接取出指定类型的值时，会发生编译错误，代码如下：

```go
// 声明a变量, 类型int, 初始值为1
var a int = 1
// 声明i变量, 类型为interface{}, 初始值为a, 此时i的值变为1
var i interface{} = a
// 声明b变量, 尝试赋值i
var b int = i
```
第8行代码编译报错：
```go
cannot use i (type interface {}) as type int in assignment: need type assertion
```
编译器告诉我们，不能将i变量视为int类型赋值给b

在代码第 15 行中，将 a 的值赋值给 i 时，虽然 i 在赋值完成后的内部值为 int，但 i 还是一个 interface{} 类型的变量。类似于无论集装箱装的是茶叶还是烟草，集装箱依然是金属做的，不会因为所装物的类型改变而改变

为了让第 8 行的操作能够完成，编译器提示我们得使用 type assertion，意思就是类型断言

使用类型断言修改第 8 行代码如下：
```go
var b int = i.(int)
```
修改后，代码可以编译通过，并且 b 可以获得 i 变量保存的 a 变量的值：1

# 类型断言
一个类型断言表达式的语法为 i.(T)，其中 i 为一个接口值， T 为一个类型名或者类型字面表示。 类型 T 可以为任意一个非接口类型，或者一个任意接口类型。

在一个类型断言表达式 i.(T) 中， i 称为断言值， T 称为断言类型。 一个断言可能成功或者失败。

对于 T 是一个非接口类型的情况，如果断言值 i 的动态类型存在并且此动态类型和 T 为同一类型，则此断言将成功；否则，此断言失败。 当此断言成功时，此类型断言表达式的估值结果为断言值 i 的动态值的一个复制。可以把此种情况看作是一次拆封动态值的尝试。

对于 T 是一个接口类型的情况，当断言值 i 的动态类型存在并且此动态类型实现了接口类型 T，则此断言将成功；否则，此断言失败。 当此断言成功时，此类型断言表达式的估值结果为一个包裹了断言值i的动态值的一个复制的 T 值。

一个失败的类型断言的估值结果为断言类型的零值。

# Golang中的排序
Go语言中提供的 sort 包内置的提供了根据一些排序函数来对任何序列进行排序的功能

sort 中使用了一个接口类型 sort.Interface 来指定通用的排序算法和可能被排序到的序列类型之间的约定。这个接口的实现由序列的具体表示和它希望排序的元素决定，序列的表示经常是一个切片。

一个内置的排序算法需要知道三个东西（sort.Interface 的三个方法）：
1. 序列的长度
2. 表示两个元素比较的结果
3. 一种交换两个元素的方式

```go
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

## 使用sort.Interface接口进行排序
对一系列字符串进行排序时，使用字符串切片（[]string）承载多个字符串。使用 type 关键字，将字符串切片（[]string）定义为自定义类型 MyStringList。为了让 sort 包能识别 MyStringList，能够对 MyStringList 进行排序，就必须让 MyStringList 实现 sort.Interface 接口。
```go
package main
import (
    "fmt"
    "sort"
)
// 将[]string定义为MyStringList类型
type MyStringList []string
// 实现sort.Interface接口的获取元素数量方法
func (m MyStringList) Len() int {
    return len(m)
}
// 实现sort.Interface接口的比较元素方法
func (m MyStringList) Less(i, j int) bool {
    return m[i] < m[j]
}
// 实现sort.Interface接口的交换元素方法
func (m MyStringList) Swap(i, j int) {
    m[i], m[j] = m[j], m[i]
}
func main() {
    // 准备一个内容被打乱顺序的字符串切片
    names := MyStringList{
        "3. Triple Kill",
        "5. Penta Kill",
        "2. Double Kill",
        "4. Quadra Kill",
        "1. First Blood",
    }
    // 使用sort包进行排序
    sort.Sort(names)
    // 遍历打印结果
    for _, v := range names {
            fmt.Printf("%s\n", v)
    }
}
```

代码输出结果：
1. First Blood

2. Double Kill

3. Triple Kill

4. Quadra Kill

5. Penta Kill

## 常见类型的便捷排序
通过实现 sort.Interface 接口的排序过程具有很强的可定制性，可以根据被排序对象比较复杂的特性进行定制。例如，需要多种排序逻辑的需求就适合使用 sort.Interface 接口进行排序。但大部分情况中，只需要对字符串、整型等进行快速排序。Go语言中提供了一些固定模式的封装以方便开发者迅速对内容进行排序。
### 1.字符串切片的便捷排序
sort 包中有一个 StringSlice 类型，定义如下：
```go
type StringSlice []string
func (p StringSlice) Len() int           { return len(p) }
func (p StringSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p StringSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
// Sort is a convenience method.
func (p StringSlice) Sort() { Sort(p) }
```
sort 包中的 StringSlice 的代码与 MyStringList 的实现代码几乎一样。因此，只需要使用 sort 包的 StringSlice 就可以更简单快速地进行字符串排序。将代码1中的排序代码简化后如下所示：
```go
names := sort.StringSlice{
    "3. Triple Kill",
    "5. Penta Kill",
    "2. Double Kill",
    "4. Quadra Kill",
    "1. First Blood",
}
sort.Sort(names)
```
简化后，只要两句代码就实现了字符串排序的功能。
### 2.对整型切片进行排序
除了字符串可以使用 sort 包进行便捷排序外，还可以使用 sort.IntSlice 进行整型切片的排序。sort.IntSlice 的定义如下：
```go
type IntSlice []int
func (p IntSlice) Len() int           { return len(p) }
func (p IntSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p IntSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
// Sort is a convenience method.
func (p IntSlice) Sort() { Sort(p) }
```
sort 包在 sort.Interface 对各类型的封装上还有更进一步的简化，下面使用 sort.Strings 继续对代码1进行简化，代码如下：
```go
names := []string{
    "3. Triple Kill",
    "5. Penta Kill",
    "2. Double Kill",
    "4. Quadra Kill",
    "1. First Blood",
}
sort.Strings(names)
// 遍历打印结果
for _, v := range names {
    fmt.Printf("%s\n", v)
}
```

### 3.sort包内建的类型排序接口一览
类  型 |	实现 sort.lnterface 的类型 | 直接排序方法 | 说  明
-|-|-|-
字符串（String）|	StringSlice	| sort.Strings(a [] string) |	字符 ASCII 值升序
整型（int）|	IntSlice |	sort.Ints(a []int) |	数值升序
双精度浮点（float64）|	Float64Slice |	sort.Float64s(a []float64) |	数值升序

编程中经常用到的 int32、int64、float32、bool 类型并没有由 sort 包实现，使用时依然需要开发者自己编写

## 对结构体数据进行排序
除了基本类型的排序，也可以对结构体进行排序。结构体比基本类型更为复杂，排序时不能像数值和字符串一样拥有一些固定的单一原则。结构体的多个字段在排序中可能会存在多种排序的规则，例如，结构体中的名字按字母升序排列，数值按从小到大的顺序排序。一般在多种规则同时存在时，需要确定规则的优先度，如先按名字排序，再按年龄排序等。

### 1.完整实现sort.Interface进行结构体排序
将一批英雄名单使用结构体定义，英雄名单的结构体中定义了英雄的名字和分类。排序时要求按照英雄的分类进行排序，相同分类的情况下按名字进行排序，详细代码实现过程如下:

结构体排序代码（代码2）：
```go
package main
import (
    "fmt"
    "sort"
)
// 声明英雄的分类
type HeroKind int
// 定义HeroKind常量, 类似于枚举
const (
    None HeroKind = iota
    Tank
    Assassin
    Mage
)
// 定义英雄名单的结构
type Hero struct {
    Name string  // 英雄的名字
    Kind HeroKind  // 英雄的种类
}
// 将英雄指针的切片定义为Heros类型
type Heros []*Hero
// 实现sort.Interface接口取元素数量方法
func (s Heros) Len() int {
    return len(s)
}
// 实现sort.Interface接口比较元素方法
func (s Heros) Less(i, j int) bool {
    // 如果英雄的分类不一致时, 优先对分类进行排序
    if s[i].Kind != s[j].Kind {
        return s[i].Kind < s[j].Kind
    }
    // 默认按英雄名字字符升序排列
    return s[i].Name < s[j].Name
}
// 实现sort.Interface接口交换元素方法
func (s Heros) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}
func main() {
    // 准备英雄列表
    heros := Heros{
        &Hero{"吕布", Tank},
        &Hero{"李白", Assassin},
        &Hero{"妲己", Mage},
        &Hero{"貂蝉", Assassin},
        &Hero{"关羽", Tank},
        &Hero{"诸葛亮", Mage},
    }
    // 使用sort包进行排序
    sort.Sort(heros)
    // 遍历英雄列表打印排序结果
    for _, v := range heros {
        fmt.Printf("%+v\n", v)
    }
}
```
代码输出如下：

&{Name:关羽 Kind:1}

&{Name:吕布 Kind:1}

&{Name:李白 Kind:2}

&{Name:貂蝉 Kind:2}

&{Name:妲己 Kind:3}

&{Name:诸葛亮 Kind:3}

### 2.使用sort.Slice进行切片元素排序
从 Go 1.8 开始，Go语言在 sort 包中提供了 sort.Slice() 函数进行更为简便的排序方法。sort.Slice() 函数只要求传入需要排序的数据，以及一个排序时对元素的回调函数，类型为 ```func(i,j int)bool```，sort.Slice() 函数的定义如下：
```go
func Slice(slice interface{}, less func(i, j int) bool)
```
使用 sort.Slice() 函数，对代码2重新优化的完整代码如下：
```go
package main
import (
    "fmt"
    "sort"
)
type HeroKind int
const (
    None = iota
    Tank
    Assassin
    Mage
)
type Hero struct {
    Name string
    Kind HeroKind
}
func main() {
    heros := []*Hero{
        {"吕布", Tank},
        {"李白", Assassin},
        {"妲己", Mage},
        {"貂蝉", Assassin},
        {"关羽", Tank},
        {"诸葛亮", Mage},
    }
    sort.Slice(heros, func(i, j int) bool {
        if heros[i].Kind != heros[j].Kind {
            return heros[i].Kind < heros[j].Kind
        }
        return heros[i].Name < heros[j].Name
    })
    for _, v := range heros {
        fmt.Printf("%+v\n", v)
    }
}
```
使用 sort.Slice() 不仅可以完成结构体切片排序，还可以对各种切片类型进行自定义排序。

## 利用辅助结构实现通用排序方法实现
第一步：首先我们先定义一个名为Comparator的方法别名，分别传入两个泛型，并返回一个int值
```go
type Comparator func(a, b interface{}) int
```

第二步：下面是一段关键代码：
```go
func Sort(values []interface{}, comparator Comparator){
	sort.Sort(sortable{values, comparator})
}

type sortable struct {
	values []interface{}
	comparator Comparator
}

func (s sortable) Len() int {
	return len(s.values)
}

func (s sortable) Less(i, j int) bool {
	return s.comparator(s.values[i], s.values[j]) < 0
}

func (s sortable) Swap(i, j int) {
	s.values[i], s.values[j] = s.values[j], s.values[i]
}
```

上面的代码主要内容：
1. 定义一个Sort方法，该方法主要是调用了sort.Sort方法，给该方法中传入了我们自定义的sortable结构体

2. 定义一个sortable结构体，第一个变量是需要排序的slice切片，另一个则是我们在第一步中自定义的Comparator方法

3. 给sortable结构体实现sort中接口的三个方法

第三步：自定义排序方法，调用排序方法：
```go
func IntComparator(a, b interface{}) int {
	aAsserted := a.(int)
	bAsserted := b.(int)
	switch {
	case aAsserted > bAsserted:
		return 1
	case aAsserted < bAsserted:
		return -1
	default:
		return 0
	}
}

func main() {
	list := make([]int, 10)

  for i:=0; i != 10; i++{
    list[i] = i
  }

	Sort(list, IntComparator)

}
```
从上面的代码可以看出，不管是任何类型的slice，只要自定义一个排序器Comparator即可实现泛型排序

至此，在Golang中如何实现泛型比较的学习就到此为止了，如果有任何探讨的问题可以在下方留言评论一起探讨。
