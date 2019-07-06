---
layout: post
title: Go编程中的冷门小知识
subtitle: '码农的人生中总会遇到无数玄学的Error，在此记录下一些比较冷门的内容，希望对Gopher有所帮助'
date: 2018-05-03
categories: Golang
tags: Golang 汇总
---
## 1.使用简式声明重复声明变量
>不能在一个单独的声明中重复声明一个变量，但在多变量声明中这是允许的，其中至少要有一个新的声明变量。重复变量需要在相同的代码块内，否则你将得到一个隐藏变量。

错误代码：
```go
package main

func main() {
    one := 0
    one := 1
}
```
编译错误：
```go
  no new variables on left side of :=
```
正确代码：
```go
package main

func main() {
    one := 0
    one, two := 1, 2
    one, two = two, one
}
```

## 2.不使用显式类型，无法使用“nil”来初始化变量
>nil标志符用于表示interface、函数、maps、slices和channels的“零值”。如果你不指定变量的类型，编译器将无法编译你的代码，因为它猜不出具体的类型。

错误代码：
```go
package main

func main() {
    var x = nil
    _ = x
}
```
编译错误：
```go
  use of untyped nil
```
正确代码：
```go
package main

func main() {
    var x interface{} = nil
    _ = x
}
```

## 3.使用“nil” Slices and Maps
>在一个nil的slice中添加元素是没问题的，但对一个map做同样的事将会生成一个运行时的panic。

错误代码：
```go
package main

func main() {
    var m map[string]int
    m["one"] = 1
}
```
编译错误：
```go
  panic: assignment to entry in nil map
```
正确代码：
```go
package main

func main() {
    var s []int
    s = append(s, 1)
}
```

## 4.Map的容量
>你可以在map创建时指定它的容量，但你无法在map上使用cap()函数。

错误代码：
```go
package main

func main() {
    m := make(map[string]int, 99)
    cap(m)
}
```
编译错误：
```go
  invalid argument m (type map[string]int) for cap
```

## 5.Strings无法修改
>尝试使用索引操作来更新字符串变量中的单个字符将会失败。string是只读的byte slice（和一些额外的属性）。如果你确实需要更新一个字符串，那么使用byte slice，并在需要时把它转换为string类型。

错误代码：
```go
package main

import "fmt"

func main() {
    x := "text"
    x[0] = 'T'
    fmt.Println(x)
}
```
编译错误：
```go
  cannot assign to x[0]
```
正确代码：
```go
package main

import "fmt"

func main() {
    x := "text"
    xbytes := []byte(x)
    xbytes[0] = 'T'
    fmt.Println(string(xbytes)) //prints Text
}
```

## 6.在多行的Slice、Array和Map语句中遗漏逗号
>当你把声明折叠到单行时，如果你没加末尾的逗号，你将不会得到编译错误。

错误代码：
```go
package main

func main() {
    x := []int{
        1,
        2
    }
    _ = x
}
```
编译错误：
```go
  syntax error: unexpected newline, expecting comma or }
```
正确代码：
```go
package main

func main() {
    x := []int{
        1,
        2,
    }
    x = x
    y := []int{3, 4}
    y = y
}
```

## 7.未导出的结构体不会被编码
>以小写字母开头的结构体将不会被（json、xml、gob等）编码，因此当你编码这些未导出的结构体时，你将会得到零值。

```go
package main

import (
    "encoding/json"
    "fmt"
)

type MyData struct {
    One int
    two string
}

func main() {
    in := MyData{1, "two"}
    fmt.Printf("%#v\n", in) //prints main.MyData{One:1, two:"two"}
    encoded, _ := json.Marshal(in)
    fmt.Println(string(encoded)) //prints {"One":1}
    var out MyData
    json.Unmarshal(encoded, &out)
    fmt.Printf("%#v\n", out) //prints main.MyData{One:1, two:""}
}
```
运行结果：
```go
    main.MyData{One:1, two:"two"}
    {"One":1}
    main.MyData{One:1, two:""}
```

## 8.使用"nil" Channels
>在一个nil的channel上发送和接收操作会被永久阻塞。这个行为有详细的文档解释，但它对于新的Go开发者而言是个惊喜。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    var ch chan int
    for i := 0; i < 3; i++ {
        go func(idx int) {
            ch <- (idx + 1) * 2
        }(i)
    }
    //get first result
    fmt.Println("result:", <-ch)
    //do other work
    time.Sleep(2 * time.Second)
}
```
运行结果：
```go
    fatal error: all goroutines are asleep - deadlock!
```
这个行为可以在select声明中用于动态开启和关闭case代码块的方法。
```go
package main

import "fmt"
import "time"

func main() {
    inch := make(chan int)
    outch := make(chan int)
    go func() {
        var in <-chan int = inch
        var out chan<- int
        var val int
        for {
            select {
            case out <- val:
                out = nil
                in = inch
            case val = <-in:
                out = outch
                in = nil
            }
        }
    }()
    go func() {
        for r := range outch {
            fmt.Println("result:", r)
        }
    }()
    time.Sleep(0)
    inch <- 1
    inch <- 2
    time.Sleep(3 * time.Second)
}
```
运行结果：
```go
    result: 1
    result: 2
```

## 9.更新Map的值
>如果你有一个struct值的map，你无法更新单个的struct值。

```go
package main

type data struct {
    name string
}

func main() {
    m := map[string]data{"x": {"one"}}
    m["x"].name = "two" //error
}
```
编译错误：
```go
    cannot assign to struct field m["x"].name in map
```
这个操作无效是因为map元素是无法取址的。

而让Go新手更加困惑的是slice元素是可以取址的。
```go
package main

import "fmt"

type data struct {
    name string
}

func main() {
    one := data{"one"}
    s := []data{one}
    s[0].name = "two" //ok
    fmt.Println(s)    //prints: [{two}]
}
```
运行结果：
```go
    [{two}]
```
