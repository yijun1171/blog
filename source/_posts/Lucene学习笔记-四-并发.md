title: Lucene学习笔记(四)并发
date: 2014-12-08 12:49:35
tags: search
categories: java
---
# 并发,线程安全及锁
索引文件的并发访问,Reader和Writer的线程安全性,以及Lucene实现前两项内容的锁机制
<!--more-->

## 线程安全

### Lucene的并发处理规则
1.只读属性的IndexReader可以同时打开一个索引,最好的办法是用多线程共享单个IndexReader实例,多个线程并行搜索同一个索引.
  获取一个IndexReader的方法:
```
IndexReader dr = new DirectoryReader.open(fsDirectory);
```

2.一个索引一次只能打开一个Writer(使用文件锁).

### 索引锁机制
  Lucene采用基于文件的锁,一般只允许一个witer打开同一个索引,可以修改锁实现方法:
```
directory.setLockFactory(new NativFSLockFactory);
```
Lucene提供的锁实现:**NativeFSLockFactory**,**SimpleFSFactory**,**SingleInstanceLockFactory**,**NoLockFactory**

Lucene不提供申请锁的队列机制,需要自己实现

## 调试索引
1. 可以用一下方法将信息输出到控制台
```
IndexWriterConfig.setInfoStream(System.out);
```

2. 使用**Luke**调试工具
