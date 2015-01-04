title: golang-concurrency-学习笔记
date: 2014-12-27 14:24:35
tags: concurrency
categories: golang
---
# golang并发编程学习笔记
主要涉及channel goroutine 同步
参照阅读<< Go并发编程实战 >>

<!--more-->
# Goroutine

## 使用方法
**go**语句 + 表达式
```
go println("Go! Goroutine!")
go func() {
  println("Go! Goroutine!")
}() //匿名函数
```
runtime.Gosched():使其他Goroutine有机会运行

一些详细与goroutine交互的函数见官方文档[runtime](https://golang.org/pkg/runtime)

# channel

## 声明

```
chan T
type IntChan chan int
```

## 操作

```
chan<- v //写入

val := <-chan //读取
val, ok := <-chan //如果通道已经关闭,ok返回false,val返回对应类型的零值
for e := range chan { //对一个双向通道进行读取迭代,通道关闭后并且没有剩余的元素,for语句执行结束
  printf("Element: %v\n",e)
}
```
个人感觉channel的操作和一个有限缓冲区实现的生产者消费者模型类似,读取操作会在channel空时阻塞,写入操作会在channel满时阻塞.阻塞的Goroutine的唤醒顺序与其阻塞顺序相同,可以理解为一个用于存放阻塞Goroutine的队列

写入时的值拷贝
向通道写入时,写入的是值的一个完全拷贝

## 初始化

```
intChan := make(chan int, 10) //通道类型 缓冲区大小
```

## 关闭

```
close(intChan) 
```
通道关闭后不能再向其中写入数据,但接收方可以继续读取数据

## select语句

```
var ch1 = make(chan int, 10)
var ch2 = make(chan string, 10)

select {
  case e1 := <-ch1:
    ftm.Printf("1th case is selected. e1=%v\n", e1)
  case e2 := <-ch2:
    ftm.Printf("2th case is selected. e2=%v\n", e2)
  default:
    ftm.Printf("default") 
}
```
通道类型不受约束
分支选择规则:
1. 自上而下对每个case求值
2. 选择可以通道操作可以执行的case,多个case满足的话随机选择一个,都不满足的话执行default

惯用法
```
go func() {
  var e int
  ko := true //标记
  for {
    select {
    case e, ok = <-ch11:
      if !ok {
        fmt.Println("END")
        break
      } else {
        fmt.Println("%d\n", e)
      }      
    }
    if !ok {
      break
    }
  }
}
```

单个操作超时触发器
```
go func() {
  var e int
  ok := true
  for {
    select {
      case e, ok = <-ch11:
      ...
      case ok = <-func() chan bool { //有case被选择之前此语句已经求值为 ok = <-timeout
                                     //,当1毫秒之内没有case执行的话,本case会执行,然后超时退出循环
        timeout := make(chan int, 1)
        go func() {
          time.Sleep(time.Millisecond)
          timeout <- false
        }()
        return timeout
      }():
        fmt.Println("Timeout")
        break       
    }
    if !ok {
      break
    }
  }
}
```
## 非缓冲通道
重要特性:
1. 只有一对发送和接受操作"配对"之后,数据才会开始传输,在此之前的操作都是阻塞的
2. 发送操作晚于接受操作完成,以确保至少有一个接收方正确接受到数据
判断一个通道是否是缓冲类型,使用**cap**方法,非缓冲通道返回0

