title: ali-intern-java基础整理
date: 2015-05-01 14:49:10
tags:intern
categories:java
---
# java基础

## 1 并发

### 1.1 内存模型
根据jls中的定义
> 内存模型描述了一个程序的执行过程是否合法.
> JMM通过检查一个程序的执行链中的**读操作**和基于该读取操作引起的**写操作**是否满足指定的**规则**.

内存模型只是描述了一个程序的行为.只要程序的执行结果可以被内存模型预测,也就是说满足内存模型中的规则,那么程序的指令级优化可以尽可能的减少冗余代码或者乱序执行.

规则
1. 共享变量:实例字段,静态字段,数组的元素.不包括线程私有的局部变量和方法参数.
2. 动作:
3. 程序和指令序
4. 同步顺序
5. Happens-before:如果一个动作happens-before另一个动作,那么第一个操作的影响一定会被第二个操作观察到.
    * 程序次序
    * constructor->finalizer
    * If an action x synchronizes-with a following action y, then we also have hb(x, y).
    * happens-before具有传递性
    * 同一个监视器锁的unlock->后面对同一个锁的lock操作
    * 对volatile变量的写->对它的读
    * 线程的start操作->所有线程中的操作
    * 线程中的所有操作->join操作
6. 执行
7. 良好的执行
8. Executions and Causality Requirements
9. 可观察的行为和不确定的执行

volatile
1. 对所有线程立即可见,在所有线程中是一致的.但不是线程安全的.(对volatile变量的操作不一定是原子的).通过反汇编发现lock指令在IA32架构中会使其他cpu的Cache无效,同时本cpu的Cache写入内存.
2. 禁止指令重排序优化.
使用volatile变量,会从主存读取最新的值.对volatile变量赋值后会立即刷新到主存.

### 1.2 线程

#### 1.2.1 线程实现
1. 内核线程实现:一对一模型.需要系统调用,在用户态和内核态中来回切换,代价高,并且直接消耗内核资源,支持的线程数量非常有限
2. 用户线程实现:一对多模型.基于用户态的线程库,内核无法感知线程的生命周期,它完全在用户态中进行.实现复杂.
3. 混合实现:多对多模型.用户线程的系统调用通过轻量级线程完成.
4. java线程的实现:取决于平台,windows和Linux提供一对一的线程模型

#### 1.2.2 线程调度
java线程采用抢占式调度,由系统来分配执行时间.可以通过设置优先级控制线程被选择执行的概率.但是不能完全准确的判断每个线程谁一定先执行.因为java中的线程优先级和操作系统的线程优先级并不是一一对应的.
Thread类中定义了5中状态:New Runnable Blocked Waiting Timed_waiting
Blocked:在等待获取一个排他锁
Waiting:无限期等待,只能被显式唤醒.可能由以下方法导致:无参调用 Object.wait() Thread.join()
Timed_Waiting:有限等待,自动被系统唤醒.Thread.sleep()  有参调用Object.wait() Thread.join()

常用方法:
1. yeild方法,提示cpu该线程让出执行权,具体决定取决于cpu,不保证立即发生调度
2. sleep方法,进入有限等待状态,不会释放持有的监视器锁
3. start()方法,在Thread对象创建之后被调用一次,否则会抛出异常
4. run()方法,只有Thread用Runnable对象创建时,调用该方法会执行Runnable对象的run方法(并没有创建新的线程!),否则不做任何操作,直接返回.

#### 1.2.2线程同步
常用的多线程同步机制:
* volatile 变量：轻量级多线程同步机制，不会引起上下文切换和线程调度。仅提供内存可见性保证，不提供原子性。
* CAS 原子指令：轻量级多线程同步机制，不会引起上下文切换和线程调度。它同时提供内存可见性和原子化更新保证。
* 内部锁和显式锁：重量级多线程同步机制，可能会引起上下文切换和线程调度，它同时提供内存可见性和原子性。


1.synchronized(内部锁)
synchronized实现主要依靠**监视器**,每个对象有一个监视器,线程可以lock和unlock.任何时候只能有一个线程持有某个特定对象的监视器,之后任何线程对该监视器的lock尝试都会导致他们陷入阻塞状态,直到那个监视器unlock.监视器锁可重入,unlock次数要等于lock次数,该监视器锁才会被真正释放
synchronized声明 先计算引用的对象->尝试获取监视器锁->执行同步代码块->执行结束后自动释放监视器锁
synchronized方法 实例方法获取实例的锁,静态方法获取相应Class对象的锁,方法执行结束后自动释放监视器锁

2.ReentrantLock(显式锁)
使用类似synchronized,但提供一下高级功能:
1. 等待可中断
2. 公平锁与非公平锁:是否按申请顺序依次获得锁
3. 绑定多个Condition对象

3.非阻塞同步
CAS 存在ABA问题,但一般不会影响并发正确性

4.ThreadLocal:将变量的使用范围限制在本线程中
ThreadLocal实例操控Thread实例中的LocalMap,将变量限制在线程中.每个Thread实例都有一个LocalMap,ThreadLocal为key,持有的变量为value.

5.等待集合
每个对象会关联一个等待集合(wait set),集合元素是在该对象的监视器上等待的线程.
通过Object.wait Object.notify Object.notifyAll 来操纵等待集合,同时它还受当前线程的中断状态和Thread类的中断方法影响.

6.Wait
线程t调用对象m的wait(),并且未unlock的次数为n,调用之后,一下过程按序执行
1. t加入到m的等待集合里,并且执行n次unlock(释放掉吃有的m的锁)
2. 直到t被移出等待集合之前,t中的指令不再执行,移出等待集合的触发动作是:
     a.在m上调用notify(),并且t被选中
    b.在m上调用notiyfAll()
    c.如果是timed wait,等待时间到了之后,t会被内部动作移出等待队列
    d.中断t
3. 执行n次lock(重新获得m的监视器锁)
4. 如果在步骤2中移出等待集合的原因是中断,t的中断状态置为false,并且wait方法抛出InterruptionException

7.Notification
通知动作因notify和notifyAll方法的调用而发生
线程t在对象m上执行以上两种方法之一时,并且未unlock的次数为n,以下动作依次发生:
1. 如果n是零,抛IllegalMonitorStateException(必须持有监视器锁)
2. 如果n大于零,并且执行的是notify,从m的等待集合中选择一个线程.同时不能保证哪一个线程被唤醒,被唤醒的线程在t执行n次unlock之后,重新获得m的锁
3. 如果n大于零,并且执行的是notifyAll,所有线程被移出等待集合,但是只有一个线程能够竞争获取到m的监视器锁.

#### 1.2.3 中断
调用Thread.interrupt会发生中断,会将被调线程的中断状态置为true
> 如果该线程在object的wait方法上阻塞,或者在Thread的join,sleep方法上阻塞,本线程的中断状态将被清除,并抛出InterruptionException
> 如果该线程阻塞在可中断的channel之上的IO操作,那么这个channel会关闭,该线程的中断状态会被设置,该线程会收到一个 ClosedByInterruptException
> 如果该线程正阻塞在selector上,那么中断状态会被设置,并且从select方法中返回,返回值是非零值,类似selector的wakeup方法被调用
> 如果不是以上情况,那么只有线程的中断状态被设置

**总而言之,java的中断机制本质上是一种线程间的协作机制,它并不保证一定能终止线程的执行.个人理解,中断的最合适的应用场景是用来处理取消请求.**
interrupt方法会将线程的中断状态设置为true.某些方法在调用过程中会检查当前线程的中断状态,并抛出异常,通知线程它被中断了.这样的方法叫可中断方法.线程的执行过程中检测到这样的中断引起的异常后,根据需要进行对应的处理.在执行不可中断方法的时候,如果外界调用了该线程的interrupt,该线程也不会有感知,只是中断状态被设置为true而已,所以并不会停止方法的执行

isInterrupted方法只检查中断不状态,没有任何影响
interrupted静态方法会将中断状态清零

处理中断:
既然是线程自己处理状态,那么就要在合适的时间去检查中断状态(检查中断需要轮训中断状态).合适的时间也取决于业务需求.在检测到中断请求后,有以下两个处理原则
如果遇到的是可中断的阻塞方法抛出InterruptedException，可以继续向方法调用栈的上层抛出该异常，如果是检测到中断，则可清除中断状态并抛出InterruptedException，使当前方法也成为一个可中断的方法。
若有时候不太方便在方法上抛出InterruptedException，比如要实现的某个接口中的方法签名上没有throws InterruptedException，这时就可以捕获可中断方法的InterruptedException并通过Thread.currentThread.interrupt()来重新设置中断状态。如果是检测并清除了中断状态，亦是如此。
中断响应:
视情况而定, 可能是终止线程,回滚执行过的任务,执行下一个任务等等

sleep和yield没有同步的语义,也就是说,执行前后不会更新缓存中的共享变量.
```
while(!this.done) // done is volatile boolean
    Thread.sleep(1000)
```
以上代码,即使其他线程更新了done的状态,仍然有可能死循环


#### 1.2.4 线程池
Executor接口
线程池初始化时,构造函数中的参数代表了任务的执行策略.在什么线程中执行,按什么顺序执行,有多少任务能并发执行,队列中有多少个任务在等待,过载的话选择哪个任务拒绝,怎么通知应用程序有任务被拒绝.

Executors提供的配置好的线程池.
1. newFixedThreadPool:core和max线程数一样,所以当线程数增长到固定数之后就不再新增线程
2. newCachedThreadPool:core=0,max=Integer.MAX_VALUE.使用吞吐率很高的SynchronousQueue,每提交一个任务就新增一个线程去处理,线程执行完任务后如果还在生存期之内的话可以重用
3. newSingleThreadPool:core=max=1,使用无界队列,单线程执行任务,执行顺序和提交顺序相同
4. newScheduledThreadPool:固定长度线程池,以延迟或定时的方式执行任务
任务队列保存在BlockingQueue中,有三种可选:无界队列,有界队列,同步移交.

新任务的执行流程:


合理配置线程池
1. cpu密集型任务使用尽可能少的线程:N+1
2. IO密集型任务使用尽可能多的线程:2*N
3. 优先级不同的任务使用PriorityBlockingQueue

线程饥饿死锁
只要线程池中的任务需要无限期的等待一些必须由池中其他任务才能提供的资源或者条件,那么除非线程池足够大,否则将发生线程饥饿死锁.
```java
public class Task implements Callable<String>{

    public String call() throws Callable<String>{
        Future<String> header, footer;
        header = exec.submit(new LoadTask("head")); //exce是单线程的线程池
        footer = exec.submit(new LoadTask("foot"));
        String page = body();
        return header.get() + page + footer.get(); //等待任务队列中的任务,将会发生死锁
     }
}
```
使用无界队列固定大小的线程池,在任务到达速率超过线程池处理速率的情况下,仍然可能会耗尽资源.
更稳妥的策略是使用有界队列
1. 线程池较小而队列较大,可以降低CPU的使用率和上下文切换频率,但是限制了吞吐量
2. 对于大的线程池,配合**SynchronousQueue**可以避免任务排队.基于该队列的性质,任务可以直接交给工作线程.如果线程池大小已经到达最大值,而且没有空闲线程的话,根据**饱和策略**,这个任务将被拒绝.
3. 针对任务之间存在依赖关系,可能发生线程饥饿死锁的情况,应该使用无界的线程池,例如newCachedThreadPool

饱和策略
1. Abort:默认策略,抛出RejectedException,调用者自己处理拒绝任务
2. Discard:抛弃提交被拒绝的任务
3. Discard-Oldest:抛弃队列中下一个将被执行的任务,不要和优先级队列一起使用
4. Caller-Runs:不抛弃任务也不抛异常,将任务回退给调用者.在调用execute的线程中执行该任务,因此主线程在一段时间内不能提交任何任务

```java
public class CallerRunsTest {

    static class Task implements Runnable{

    public void run() {
        for (int i = 0; i < Integer.MAX_VALUE; i++);//execute
            System.out.println(Thread.currentThread());
        }
    }
    public static void main(String[] args){
        int CAPACITY = 8;
        ThreadPoolExecutor executor = new ThreadPoolExecutor(5,5,0L, TimeUnit.SECONDS,
                                                 new LinkedBlockingDeque<Runnable>(CAPACITY));
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        Thread.currentThread().setName("main");
        for(int i = 0; i < 100; i++){
            executor.execute(new Thread(new Task(),"Task-" + i));
             //当任务队列满之后,实际的任务执行线程变成main
            System.out.println("submit task "+i);
        }
    }
}
```

生命周期管理
ExecutorService
shutdown执行平缓关闭,不再接受新的任务,同时等待已经提交的任务执行完毕.
shutdownNow尝试取消所有运行中的任务,不再启动队列中没开始的任务.
被拒绝的任务交给指定的
Callable代表任务
Future表示任务的生命周期,可以通过它判断是否已经完成或者取消,以及获取任务的结果和取消任务
Future可以设置任务执行的超时时间,get(time,timeUnit),超时会抛出异常,捕获异常后取消任务即可
```java
try{
    result = future.get(timeLeft, NANOSECOND);
} catch(ExecutionException e){
    
} catch(TimeoutException e){
    future.cancle(true);
}
```
ExecutorService的submit提交任务方法返回该任务的Future,用来监控任务的状态
invokeAll(List<Callable>, long, time) 集中提交一组任务,返回一组结果,每个任务要么执行完毕要么被取消了


```
## 2 容器
常用容器类图

### 2.1 常用容器

主要分为两大类
1. Collection的子接口:List,Set,Queue
    * List的实现类包括不同步的:ArrayList和LinkedList,分别基于数组和链表实现.同步的Vector和Stack
    * Set的实现类包括HashSet,LinkedHashSet,TreeSet.HashSet基于HashMap实现的,LinkedHashSet在哈希的存储结构上提供了控制迭代顺序的功能,利用辅助的双向链表来记录顺序.TreeSet基于TreeMap实现的
    * Queue的实现类包括LinkedList,PriorityQueue和大量的阻塞队列
2. Map的实现类:HashMap, TreeMap,HashTable(同步的)
    * HashMap,拉链法实现的哈希表,使用HashMap最重要的是作为键的类正确实现了hashCode方法和equals方法,并且不可变.
    * TreeMap,红黑树实现的K-V映射,保证了键有序

在使用有序容器时要正确实现Comparable接口
Collections和Arrays提供了常见的容器操作
阻塞队列的实现基本采用了RentrentLock和Condition,实现了生产者-消费者模型,同时还可以选择性保持线程公平

### 2.2 并发容器(非阻塞)
**并发容器大多采用了非阻塞算法进行同步,算法实现是基于处理器指令集提供的CAS指令,这是一种乐观锁形式的同步方式,并且锁的粒度非常小,这也就意味着实现的复杂性.**

基于非阻塞算法的容器:
* ConcurrentLinkedQueue:基于链接节点的无界线程安全队列
* SynchronousQueue:没有容量的阻塞队列，它使用双重数据结构 来实现非阻塞算法
* Exchanger:是一个能对元素进行配对和交换的交换器
* ConcurrentSkipListMap:是一个可以根据 Key 进行排序的可伸缩的并发 Map

## 3 IO&NIO

## 4 内存管理和GC
### 4.1 运行时的数据区
> 根据虚拟机规范划分,只有根据实际应用需求,实现方式选择最优的收集方式才能获取最高的性能

1. 方法区:线程共享,存储加载的类信息,常量池(编译期确定的字面量和符号引用),静态变量,即时编译器编译后的代码
2. 堆:线程共享,存放绝大多数的对象实例
3. 虚拟机栈:保存栈帧
4. 本地方法栈:保存本地方法的栈帧
5. 程序计数器:指向下一条字节码指令

### 4.2 HotSpot中的对象
1. 对象创建过程:类加载检查->分配内存(指针碰撞/空闲列表,由GC之后的内存布局决定)->内存初始化为零值->设置对象头->执行<init>
2. 对象的内存布局:对象头(运行时数据和类型指针,实例数据,对齐填充
3. 对象的访问定位:句柄和直接指针(hotspot采用,引用指向对象的内存地址)
GC

### 4.3 判断对象存活
1. 引用计数:不能解决循环引用问题
2. 可达性分析:从GC ROOT开始,不可达的对象判定为可回收

### 4.4 GC算法
虚拟机实现一般使用分代收集,将堆划分成不同的代,根据每代的对象特点使用不同的GC算法
1. 标记-清除:完成所有对象的标记,进行回收.效率不高,内存碎片多
2. 复制:牺牲内存.用来回收新生代,按一定比例划分内存区域,需要分配**担保**
3. 标记-整理:多用于对象存活率高的内存区域

### 4.4 GC
讨论HotSpot中的垃圾收集器,没有最好的GC,每个GC都有自己最适合应用的场景
新生代:
1. Serial:复制算法.单线程,GC时必须停止所有用户线程,简单而高效
2. ParNew:Serial的多线程版本
3. Parallel Scaveng:使用复制算法,并行多项收集.

老年代
1. Serial Old
2. Parallel Old
3. **CMS**:标记-清除算法.CPU敏感,无法处理浮动垃圾,需要预留一部分空间供并发时的用户线程使用.碎片多.

G1收集器
将java堆划分成不同的region,在后台维护一个有限列表,有限回收价值最大的region
用Remember Set来避免处理**Region之间**的对象引用时进行全堆扫描.

### 4.5内存分配与回收策略
TLAB(本地线程分配缓冲)>Eden>老年代
具体的分配规则取决于GC组合和虚拟机参数

1. 优先在Eden中分配,Eden内存不足时触发Minor GC
2. 大对象直接进入老年代(长字符串或大数组之类的).通过设置参数直接在老年代分配内存
3. 长期存活的对象进入老年代.每个对象有一个计数器,每经过一个MinorGC,计数器加1,晋升到老年代的阈值通过虚拟机参数设置
4. 动态年龄判断

### 4.6空间分配担保

参考资料:
1. 深入理解java虚拟机
2. java并发编程实战
3. Java Language Specifications
4. Java SE7 API Specification
5. 并发编程网
6. IBM developerWorkers
7. [java.util.concurrent并发包诸类概览](http://www.raychase.net/1912)




