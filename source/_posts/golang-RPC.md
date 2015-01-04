title: CMU.Distributed-System.Lab1(RPC)
date: 2014-12-15 21:19:19
tags: distributed-system
categories: golang
---

RPC的一些基本概念和基于golang的简单实现

<!--more-->
# RPC

**RPC**:Remote Procedure Call 远程过程调用
简单解释:在客户端发生的过程调用,却实际操作却在服务器上执行

## 工作原理

1. 客户端应用调用**函数**
2. 该**函数**实际上是桩**(stub)**,它实际上将调用的函数名和参数打包成结构化的数据
3. 底层将这些打包好的数据以消息的形式,通过网络发送给服务器
4. 服务器将收到的消息拆包,解析,决定将得到的数据派发给被客户端**请求**的函数去执行
5. 服务器将执行结果打包发送给客户端
6. 客户端的**桩**拆包,解析服务器的响应,将执行结果返回给调用者

## 同步/异步

同步:客户端必须等到以上所有步骤执行完之后才能,代码才能继续向下执行
异步:**桩**函数在第3步执行之后,客户端就可以往下继续执行了,调用结果在之后的某个时刻再处理

## 和本地过程调用的区别

1. 运行在不同机器上
  * 不共享内存.数据通过网络进行传输
  * 可能有异构(字节序不一样),或者不同编程语言,运行在不同OS上
2. 需要网络通信
  * 可能需要定位服务器(IP)
  * 需要某种程度上的网络**连接**
  * 需要约定好编解码方式
3. 更容易发生错误
  * 丢包,冗余,乱序
  * 节点失效
4. 需要考虑更多的安全问题
  * 需要验证机制
  * 需要加密机制

# 分析

Lab1的组件分为3部分:

1. 请求客户端(Request):提交请求,等待服务器给出运算结果
2. 运算客户端(Worker):初始化,发送**加入**请求给服务器.然后转变角色,成为**server**(概念上的),等待服务器给它分配任务
3. 服务器(Server):组织所有的任务,维持所有的运算客户端.响应请求客户端,切分任务,指派任务,收集运算结果,给请求客户端发送反馈

RPC过程:
Request --> Server --> Request. 典型的同步RPC
Worker --> Server. 同步RPC,但是没有返回值
Server --> Worker. Worker在进行运算的时候显然需要异步RPC,以提高服务器的吞吐率

# 细节

封装(marshaling)约定,我们采用JSON格式
使用Go RPC package

Server端
将操作封装成函数:
```
func (s *servertype) Operate (args *argtype, reply *argtype) Error
```
函数必须对参数进行解码,执行操作,对返回值编码,如果没有错误的话返回nil

然后必须**注册**对应的`servertype`.

---

Client端
同步调用:
封装好方法名,参数,返回值,然后等待调用返回
异步调用:
通过关键字`go`,异步执行调用,通过`channel`获取返回值

Example:

# 课程源码解读
课程提供的两个源文件使用channel和mutex两种方法实现了简单的rpc底层通信过程,实现简单,我们主要学习用golang实现的思想和方法

## channel
本实例使用简单的channel和goroution,用io模拟网络通信过程,实现toyRPC过程

Client关键点:
使用channel缓存请求和响应,使用select语句对io进行多路复用
Call使用channel写入和读取请求
```
func (tc *ToyClient) Call(procNum int32, arg int32) int32 {
  done := make(chan int32) // for tc.Listener()
  tc.requestchan <- RequestChan{Request{int64(0), procNum, arg}, done} //将本次请求写入请求缓冲通道
  reply := <- done // 从响应缓冲通道中读取
  return reply
}
```


```
for {
  var xid int64
  xid = 1 //自增id
  pending := map[int64]chan int32{} //用于缓存已经发出但还没收到响应的请求,channel类型的好处在于将响应的分析过程异步化,不会阻塞底层多路对io的监听
  select {
    case req := <- client.requestchan //请求缓冲通道
      pending[xid] = req.done // 等待响应信息
      wirte(req)
      xid++
    case rep := <- client.replychan //响应信息缓冲通道
      pending[rep.Xid] <- Res  //将响应信息写入通道
      delete(pending, rep.Xid) //本次请求收到正常响应,从缓冲区删除
  }
}
```
![client结构图](/img/golang-client.jpeg)

Server关键点:
使用channel分离request的接收和response的响应
dispatcher
```
for {
  req := server.ReadRequest()
  fn, ok := server.handlers[req.ProcNum] //访问所请求的方法
  go func() { //非阻塞的处理请求
    reply := Reply{req.Xid, 0}
    if ok {
      reply.Res = fn(req.Arg) //调用方法
    }
    server.replychan <- reply //将响应消息写入通道
  }()
}
```
writer:专门负责消息的写出
```
for {
  select {
  case rep := <- server.replychan //从channel中读取待写出的响应
  server.WriteReply($rep) //写出
  }
}
```

**收获**:利用channel实现的一个简单有效的非阻塞请求处理思想,这也是golang处理并发和异步的精髓

## mutex
使用mutex实现rpc的底层调用
这里主要用到了sync包中的互斥锁Mutex和条件变量Cond
阅读官方文档:
[Mutex](https://golang.org/pkg/sync/#Mutex)
[Cond](https://golang.org/pkg/sync/#Cond)
Mutex的两个导出方法:
*Lock*:加锁,当该锁已经被其他goroutine所持有时,调用者会被挂起
*Unlock*:释放锁
有一句很重要的解释:Mutex的所有权不在加锁的goroutione上,它允许其他goroutine去释放锁

Cond
每个条件变量使用跟一个锁相关联(**Mutex**或者**RWMutex**)
导出方法:
*NewCond*:使用互斥锁初始化
*Broadcast*:唤醒所有挂起的线程,调用时不要求持有锁
*Signal*:唤醒一个挂起的线程,调用时不要求持有锁
*Wait*:调用Wait时会将持有的锁释放,然后等待其他线程调用*Broadcast*或者*Signal*唤醒,被唤醒后,*Wait*返回之前会将锁锁住,但是Wait第一次恢复时不保证锁被锁住,也就不能假设条件变量为true,因此需要用一个循环去测试条件变量,
exampler from doc
```
c.L.Lock()
for !condition {
  c.Wait()
}
....
c.L.Unlock()
```

与使用**channel**的实现不同,这里使用**Cond**条件变量来表示一次请求是否完成
```
type State struct {
  cond *sync.Cond //请求是否完成
  reply *Reply  //封装响应信息
}
```

Client关键点:
Call方法

```
func (tc *ToyClient) Call(procNum int32, arg int32) int32 {
  tc.mu.Lock() //全局共享的互斥锁
  defer tc.mu.Unlock()

  xid := tc.xid // allocate a unique xid
  tc.xid++
  tc.pending[xid] = &State{sync.NewCond(&tc.mu), nil} //每一次调用请求开始,生成一个State表示一次请求的完成状态,使用全局的互斥锁来构造条件变量
  req := &Request{xid,procNum, arg}
  tc.WriteRequest(req) // send to server

  for tc.pending[xid].reply == nil { //循环测试条件,没有得到服务器响应,当得到服务器响应时跳出循环
    tc.pending[xid].cond.Wait() //本线程挂起,解开互斥锁,其他线程有机会向服务器发送调用请求
  }
  
  r := tc.pending[xid].reply
  delete(tc.pending, xid)
  return r.Res
}
```
listener,客户端被创建时,使用go语句调用,专门负责监听响应
```
func (tc *ToyClient) Listener() {
  for {
    reply := tc.ReadReply()
    tc.mu.Lock() //请求锁
    entry, ok := tc.pending[reply.Xid] //获取对应的State.Reply
    if ok {
      entry.reply = reply;
      entry.cond.Signal() //唤醒对应挂起的线程,表示本次调用得到响应,Call方法得以返回
    }
    tc.mu.Unlock() //释放锁
  }
}
```

server关键点
dispatcher,同样是server启动时由go语句执行,专门负责调用请求的派发
```golang
func (ts *ToyServer) Dispatcher() {
  for {
    req := ts.ReadRequest()
    ts.mu.Lock() //全局互斥锁
    fn, ok := ts.handlers[req.ProcNum]
    ts.mu.Unlock()
    go func() { //非阻塞执行调用
      reply := Reply{req.Xid, 0}
      if ok {
        reply.Res = fn(req.Arg)
      }
      ts.mu.Lock()
      ts.WriteReply(&reply)
      ts.mu.Unlock()
    }()
  }
}
```

### 两种方法的比较
channel方式实现更加go style

# rpc package
golang提供了rpc的基础包
我们在这里使用[jsonrpc](https://golang.org/pkg/net/rpc/jsonrpc/)

## 要求
1. 服务器端,声明一个导出类型作为服务,为该类型声明至少一个导出方法.方法要求:
  1. 方法导出
  2. 有两个参数,都是导出的类型
  3. 第二个参数是指针类型
  4. 返回值是error
形如
```
func (t *T)MethodName(argType T1, replyType *T2) error
```
2. 调用`rpc.register(service)`,将声明的服务的指针注册到rpc上
3. 监听指定端口
4. 将到达的请求交给指定方法进行解码,这里使用`server.ServeCodec(jsonrpc.NewServerCodec(conn))`
5. 客户端使用`jsonrpc.Dial(netword, address)`方法向服务端发起请求
6. 方法调用时使用`rpc.Client`的导出方法`Call(serviceMethod, args, reply)`,其中第三个参数必须是指针类型,注意,serviceMethod必须形如**service.method**,service是服务器端注册服务的类型名,method是调用的方法名


