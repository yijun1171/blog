title: "ali-intern-linux&TCP"
date: 2015-05-09 16:46:10
tags: intern
categories: linux TCP
---

# linux相关

## 磁盘&文件系统

### 文件系统
虚拟文件系统(Virtual File System)为上层提供了一个**统一的接口**,不同的物理设备和文件系统类型对上层来说是透明的.对下层,新的物理设备和文件系统可以直接接入linux,而不必重写或者重新编译程序.
![VFS](http://yijun1171.github.io/img/VFS.gif)
尽管内核是c语言编写的,VFS还是用**面向对象**的思想来实现的.虚拟文件系统中有四个主要对象(模型),他们可以用来描述所有任何一个系统中的特征和行为.它们用c语言的struct实现,struct中包含了**数据**和**函数指针**(内核调用他们来操纵数据).

1. 超级块(super block),表示一个已挂载的文件系统,一般存储在磁盘的特定扇区中.
2. 索引节点(inode),表示一个文件(包括目录,物理设备,普通的文件)的所有元信息.这些信息保存在磁盘中,当访问一个文件时从磁盘读取数据(从元数据区)然后在内存中创建inode(struct).inode的操作**由文件所在的文件系统实现**.
3. 目录项(dentry),表示一个目录项,它是一个路径的组成部分,提高了通过路径进行文件查找的效率(主要是通过**缓存**避免每次查找都要通过系统调用).它并不对应磁盘中的数据,当用字符串表示一个路径的时候它才会被创建(不需要写回).
4. 文件对象(file),表示一个进程打开的文件

#### 访问文件的过程
file->dentry->inode->data
其中dentry和inode的查找都会使用缓存,而inode中保存的文件数据的块号,并且使用**多级索引**.

注意:目录项(dentry)不是目录,目录(directory)是文件(file)的一种形式

### 块设备
块设备是指linux系统中支持**随机访问**固定大小数据块的设备.
扇区(sector)是块设备的最小寻址单位,通常是512字节.而软件层面的最小寻址单位则是块(block),块大小通常是512B,1KB,4KB.
2.6版本以后,每个block IO request用一个bio structure来表示.每个请求会包括不同位置的块(在不同的页中),bio用一个bio_vec链表来记录请求中所有块的位置,链表节点< page,offset,len >,记录了块在物理页中的具体位置.(其中有一个小细节,bi_idx指向了当前节点的位置,有了这个特性可以跟踪IO请求的完成情况,甚至RAID的IO也是通过它来实现的:) 

inode table还有block和其他一些元数据在分区格式化的时候创建和划分.

### 挂载
将文件系统与目录树结合,挂载点是该文件系统的入口.

## 常用命令
用法记录在cheatsheet中,持续保持更新
## shell
参考一本评价较高的Bash手册,写了一些测试代码在这里:[shell-learning-notes](https://github.com/yijun1171/shell-learning-notes)

# TCP
### 状态机
![state-machine](http://yijun1171.github.io/img/tcpfsm.png)
图片来源:[CollShell](http://coolshell.cn/articles/11564.html)
### 建立连接
TCP建立连接的过程一般经过三步
1. client发送SYN
2. server返回SYN+ACK
3. client返回ACK
我们成为**三次握手**

初始序号(ISN)是随时间变化而变化的,这样保证连接重连时,上一个连接在网络中延迟的包不会被错误地解释.

双方同时打开,可能会出现交换了4个报文的情况(不多见)


### 关闭连接
TCP关闭连接一般分为4步
1. A发送FIN
2. B发送ACK
3. B发送FIN
4. A发送ACK

从状态转换的角度来说
1. A接到上层应用的关闭指令,发送FIN后,转换到*FIN_WAIT1*状态
2. B收到FIN,发送ACK后,转换到*CLOSE_WAIT*状态.此时连接处于**半关闭**.B仍然可以发送数据,而A通过发送FIN表示自己不再发送数据.
3. B接到上层应用的关闭指令后,发送FIN,状态转换到*LAST_ACK*.
4. A收到FIN后,发送最后一个ACK,状态转换到*TIME_WAIT*
A在收到最后一个ACK后真正关闭连接
B在等待2MSL的时间(并且没有收到A重发的FIN)后真正关闭连接

#### Time_wait
首先发起关闭的那一方,在发送了最后一个ACK之后,会等待2MSL的时间,这样做的原因:
1. 保证最后那个ACK一定被对端接收到.如果该ACK丢失,对端会在2MSL时间内重发FIN,保证正常关闭.这样一来一回,时间正好2MSL
2. 在等待这2MSL时间的过程中,网络中被缓存或者延迟的数据包也会失效.这样保证下一个连接(socket的四元组完全一样)建立的时候,那些数据包不会产生异常的影响.

### 复位报文段
TCP的标志位中有一个**复位**(RST).它有以下作用:
1. 到不存在的端口的连接请求,对端会返回一个RST置1的报文,表示端口不可达.
![](http://yijun1171.github.io/img/tcp-connRefuse.png)
2. 检测半打开的连接.在没有设置keepalive的情况下,如果server异常断开,client不会察觉,当server恢复时,client向server发送消息,server会返回一个RST,表示之前连接的所有信息已经丢失

### 超时处理
#### 连接建立超时
目的端处于不正常状态,无法响应SYN报文请求(一般是目的端**连接队列**中的连接数超过了**backlog**).请求端间隔一定时间后重发SYN试图重建连接.重传间隔大约是2^n.
这是测试时设置大量tcp请求连接,而服务器端吞吐量很小时,无法即时响应SYN,客户端只能持续重传.
```java
//参数10设置了超时时间
socket.connect(new InetSocketAddress("127.0.0.1",9999), 10);
```
当连接建立过程超过如上设置的超时时间,java会抛出SocketTimeoutException异常
![conn-timeout](http://yijun1171.github.io/img/tcp-connTimeout.png)
TCP有一个默认的最大重试等待间隔,大约是75s,超过这个间隔扔无法建立连接,TCP协议本身会停止尝试,向上层说明connection timeout

#### 读超时
TCP的REVQ没有可读数据时,read操作会阻塞,直到有数据可读或者超时.超时时间通过`setSoTimeout()`方法设置.当抛出超时异常时,**本socket仍然有效**,只是提示本次读操作超时异常.

### Java中socket常用参数
1. SO_KEEPALIVE:保活机制,探测的时间间隔参数在内核中设定.该参数会影响所有socket的行为
2. SO_RCVBUF & SO_SNDBUF:设置读写缓冲区大小,至少为MSS的4倍(用于3duplicate-ACK,保证快速重传)
3. SO_TIMEOUT:设置读超时的时间
4. SO_REUSEADDR:
    * 监听进程重启时,保证重新绑定之前的端口操作成功.
    * 允许在同一个端口上启动同一个服务器的多个实例


参考资料:
1. [linux内核设计和实现](http://book.douban.com/subject/5503292/)
2. [Linux文件系统剖析](https://www.ibm.com/developerworks/cn/linux/l-linux-filesystem/)
3. [Advanced Bash-Scripting Guide](http://tldp.org/LDP/abs/html/)
4. [AWK入门指南](http://awk.readthedocs.org/en/latest/index.html)
5. [TCP/IP详解 卷1](http://book.douban.com/subject/1088054/)
6. [UNIX网络编程 卷1：套接字联网API](http://book.douban.com/subject/4859464/)
7. [CoolShell-TCP那点事儿](http://coolshell.cn/articles/11564.html)
