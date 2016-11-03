+++
draft = false
comments = true
slug = ""
date = "2016-11-03T16:19:22+08:00"
title = "Golang读写同步细节"
toc = true
tags = ["go"]
categories = ["programming"]

+++
注：这篇文章是翻译（部分）、总结自这篇[官方参考文档][ref]，原文标题为“The Go Memory Model”，不过内容大多是关于同步的。英文好的同学可以直接去看原文，这篇主要用于自己巩固知识。

## Introduction
这篇文章阐明了那些`两个goroutine分别读写同一个变量，读操作在写操作之后`的情况。对于程序中同时被多个goroutine操作的变量，这些操作需要被序列化，而影响序列化操作先后顺序的因素有：

- channel
- sync(Mutex、RWMutex)
- sync/atomic // 本篇暂不涉及

## 初始化
- 如果包a import 包b，那么b的init函数先于包a的init函数
- 所有包的init函数先于main.main函数

## goroutine
这个比较简单，直接举例说明

```
var a string
func f() {
    a = "hello" // 1
    go func() {
        print(a) // 2
        a = "world" // 3
    }()
    print(a) // 4
}
```
上面的代码中，2打印的一定是“hello”，4处打印的却不一定是“world”
## channel
channel可以用来保证读写顺序，根据构造方式，分为有缓存的channel和无缓存的channel两种。注意以下两种情况：

```
// 情况1，打印的一定是“hello world”
var c = make(chan int)
var s string
func main() {
    go func() {
        s = "hello world"
        <-c // 1
    }()
    c <- 0 // 2
    print(s)
}

// 情况2，打印的不能保证是“hello world”
var c = make(chan int， 1)
var s string
func main() {
    go func() {
        s = "hello world"
        <-c // 3
    }()
    c <- 0 // 4
    print(s)
}
```
上述两种情况的区别在于：情况1是没有缓存的channel，1与2是同时发生的，即在执行到1之前，2是被阻塞的；而对于情况2，4的操作向channel输入一个数不会阻塞，此时不能确保s已经被赋值。

在一个channel缓存满了后，才会开始阻塞。对于一个容量为N的channel，第c次输出优先于第c + N次输入。可以利用channel的这一特性，对某些资源做出限制：

```
var limit = make(chan int, 3)

func main() {
    for _, w := range work {
        go func(w func()) {
            limit <- 1
            w()
            <-limit
        }(w)
    }
    select{}
}
```
上面的程序，保证了对于一个工作队列，同一时刻最多只有3项任务在执行
## Locks
sync包实现了sync.Mutex和sync.RWMutex，第n + 1次Lock()会被阻塞，直到第n次Unlock()返回

```
var l sync.Mutex
func f() {
    a = "hello, world"
    l.Unlock()
}

func main() {
    l.Lock()
    go f()
    l.Lock()
    print(a)
}
```
上述程序会确保输出“hello, world”

对于RWMutex，当第n次Unlock()发生后，第n + 1此Lock()会被阻塞，直到有相同数量的RLock()和RUnlock()发生
## Once
Once用于多个goroutine并发时，初始化函数只执行一次，在初始化函数结束前，所有goroutine都会阻塞。

```
var a string
var once sync.Once

func setup() {
    a = "hello, world"
}

func doprint() {
    once.Do(setup)
    print(a)
}

func twoprint() {
    go doprint()
    go doprint()
}
```
上面的代码确保会输出两次“hello world”
## 错误的同步示例
```
// 情况1，有可能出现先打印2，再打印0（与代码中赋值顺序相反）
var a, b int

func f() {
    a = 1
    b = 2
}

func g() {
    print(b)
    print(a)
}

func main() {
    go f()
    g()
}

// 情况2，有可能出现第二次打印为空的现象。因为没有同步机制，不能通过确保done被赋值，推断出a被赋值
var a string
var done bool

func setup() {
    a = "hello, world"
    done = true
}

func doprint() {
    if !done {
        once.Do(setup)
    }
    print(a)
}

func twoprint() {
    go doprint()
    go doprint()
}

// 情况3，有可能出现打印为空的现象。
type T struct {
    msg string
}

var g *T

func setup() {
    t := new(T)
    t.msg = "hello, world"
    g = t
}

func main() {
    go setup()
    for g == nil {
    }
    print(g.msg)
}
```
上述三种情况都需要使用同步机制来保证读写顺序
## 关于编译优化的疑问
作为一名通信工程专业的野生程序员，不懂编译原理（一定会学，暂时优先级不高），我猜测编译器会在程序中根据特定规则标记一些位置，这些位置之间的代码（主要是赋值语句）的顺序是不确定的，而标记过的位置则是有序的。比如，一个函数语句前有a、b两条赋值语句，后有c、d两条赋值语句，那么a与b、c与d的顺序是不确定的，但a、b一定会在c、d之前？

[ref]: [https://golang.org/ref/mem]