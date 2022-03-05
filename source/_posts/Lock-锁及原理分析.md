title: Lock 锁及原理分析
author: Peilin Deng
tags:
  - 并发编程
categories:
  - 摘抄笔记
date: 2022-03-04 16:58:00
---
下图所示，在不同的CPU架构中，为了避免因为指令重排序、或者缓存一致性问题，都提供了不同的内存屏障指令。同时，在不同的操作系统中，也都会实现封装一个内存屏障的实现。

那么，我们写的Java线程，如何能够在不同的硬件、不同操作系统下，仍然能够保证线程安全性呢？这就要引出JMM（Java 内存模型），它就是为了屏蔽操作系统和硬件的差异，让一套代码在不同平台下都能达到线程安全的访问目的。

![](/images/img-150.png)

# 一、什么是JMM

那什么是JMM呢？

首先，我们都知道Java程序是运行在Java虚拟机上的，同时我们也知道，  JVM是一个跨语言跨平台的实 现，也就是Write Once、 Run Anywhere。

那么JVM如何实现在不同平台上都能达到线程安全的目的呢？所以这个时候JMM出来了， **Java内存模型 （Java Memory Model ,JMM）就是一种符合内存模型规范的，屏蔽了各种硬件和操作系统的访问差异的，保证了Java程序在各种平台下对内存的访问都能保证效果一致的机制及规范**

Java内存模型规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存中 保存了这个线程中用到的变量的主内存副本拷贝，线程对变量的所有操作都必须在工作内存中进行，而 不能直接读写主内存。

不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量的传递均需要自己的工作内存和主存之间进行数据同步进行，流程图如下：

![](/images/img-151.png)

再总结一下： JMM定义了共享内存中多线程程序读写操作的行为规范：在虚拟机中把共享变量存储到内存以及从内存中取出共享变量的底层实现细节。

目的是解决由于多线程通过共享内存进行通信时，存在的本地内存数据不一致、编译器会对代码指令重排序、处理器会对代码乱序执行等带来的问题

> 本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。

实际上，如果大家认真听了前面的内容，不难发现JMM的整个模型实际上和CPU高速缓存和内存交互的模型是一致的，因为不管软件怎么设计，最终还是由硬件来执行。而这个抽象模型的意义就在于，它可以针对不同平台来保证并发场景下的可见性问题。

```c++
inline void OrderAccess::loadload()   { acquire(); }
inline void OrderAccess::storestore() { release(); }
inline void OrderAccess::loadstore()  { acquire(); }
inline void OrderAccess::storeload()  { fence(); }
```
**orderAccess_linux_x86.inline**

```c++
inline void OrderAccess::fence() {
	if (os::is_MP()) {
		// always use locked addl since mfence is sometimes expensive
		#ifdef AMD64
		__asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
		# else
		__asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
		#endif
	}
}
```

**orderAccess_linux_sparc.inline**
```c++
inline void OrderAccess::fence() {
	__asm__ volatile ("membar  #StoreLoad" : : :);
}
```

**OrderAccess::storeload();**

**ACC_VOLATILE**

**bool is_volatile () const { return (_flags & JVM_ACC_VOLATILE ) != 0; }**

# 二、总结

## 可见性导致的原因

1. CPU的高速缓存

2. 指令重排序

> 并不是所有的程序指令都会存在可见性或者指令重排序问题。

# 三、Happens-Before模型
前面说了这么多，都是为了讲解清楚，到底是什么原因导致了在多线程环境下的可见性和有序性问题。

并且也了解了volatile解决可见性问题的本质。

那么有没有哪些情况是，不需要通过增加volatile关键字，也能保证在多线程环境下的可见性和有序性的呢？

从JDK1.5开始，引入了一个happens-before的概念来阐述多个线程操作共享变量的可见性问题。所以我们可以认为在JMM中，如果一个操作执行的结果需要对另一个操作课件，那么这两个操作必须要存在happens-before关系。这两个操作可以是同一个线程，也可以是不同的线程。

## 程序顺序规则

一个线程中的每个操作，happens-before这个线程中的任意后续操作，可以简单认为是as-if-serial。

as-if-serial的意思是，不管怎么重排序，单线程的程序的执行结果不能改变。
- 处理器不能对存在依赖关系的操作进行重排序，因为重排序会改变程序的执行结果。
- 对于没有依赖关系的指令，即便是重排序，也不会改变在单线程环境下的执行结果。

具体来看下面这段代码，A和B允许重排序，但是C是不允许重排，因为存在依赖关系。根据as-if-serial语义，在单线程环境下， 不管怎么重排序，最终执行的结果都不会发生变化

```java
int a=2; //A 
int b=2; //B 
int c=a*b; //C
```

## 传递性规则

仍然看下面这段代码，根据程序顺序规则可以知道，这三者之间存在一个happens-before关系。

```java
int a=2; //A 
int b=2; //B 
int c=a*b; //C
```

- A happens-before B。
- B happens-before C。
- A happens-before C。

这三个happens-before关系，就是根据happens-before的传递性推导出来的。很多同学这个时候又有疑惑了，不是说，A和B之间允许重排序吗？那是不是A happens-before B不一定存在，也可能是B可以重排序在A之前执行呢？

没错，确实是这样，JMM不要求A一定要在B之前执行，但是他要求的是前一个操作的执行结果对后一个操作可见。这里操作A的执行结果不需要对操作B可见，并且重排序操作A和操作B后的执行结果与A happens-before B顺序执行的结果一直，这种情况下，是允许重排序的。

## volatile变量规则

对于volatile修饰的变量的写操作，一定happens-before后续对于volatile变量的读操作，这个是因为volatile底层通过内存屏障机制防止了指令重排，这个规则前面已经分析得很透彻了，所以没什么问题，我们再来观察如下代码，基于前面两种规则再结合volatile规则来分析下面这个代码的执行顺序，

假设两个线程A和B，分别访问writer方法和reader方法，那么它将会出现以下可见性规则。

```java
public class VolatileExample {
    int a=0;
    volatile Boolean flag=false;
    public void writer(){
        a=1;		//1
        flag=true;	//2
    }
    public void reader(){
        if(flag){ 		//3
            int i=a;	//4
        }
    }
}
```

- 1 happens before 2、 3 happens before 4， 这个是程序顺序规则
- 2 happens before 3、 是由volatile规则产生的，对一个volatile变量的读，总能看到任意线程对这个volatile变量的写入。
- 1 happens before 4， 基于传递性规则以及volatile的内存屏障策略共同保证

那么最终结论是，如果在线程B执行reader方法时，如果flag为true，那么意味着 i=1成立。

![](/images/img-152.png)

> 这里有同学可能会有疑问说，你前面讲的程序顺序规则中，在单线程中，如果两个指令之间不存在依赖关系，是允许重排序的，也就是1 和 2的顺序可以重排，那么是不是意味着最终4输出的结果是0呢?

这里也是因为volatile修饰的重排序规则的存在，导致1和2是不允许重排序的，在volatile重排序规则表中，如果第一操作是普通变量的读/写，第二个操作是volatile的写，那么这两个操作之间不允许重排序

### volatile 重排序规则表
![](/images/img-153.png)

## 监视器锁规则

一个线程对于一个锁的释放锁操作，一定happens-before与后续线程对这个锁的加锁操作。

```java
int x=10;
synchronized (this) {
    // 此处自动加锁
    // x 是共享变量 , 初始值 =10
    if (this.x < 12) {
        this.x = 12;
    }
}
// 此处自动解锁
```

假设x的初始值是10，线程A执行完代码块后，x的值会变成12，执行完成之后会释放锁。线程B进入代 码块时，能够看到线程A对x的写操作，也就是B线程能够看到x=12。


## start规则

如果线程A执行操作ThreadB.start(),那么线程A的ThreadB.start()之前的操作happens-before线程B中的任意操作。

```java
public StartDemo{
    int x=0;
    Thread t1 = new Thread(()->{
        // 主线程调用 t1.start() 之前
        // 所有对共享变量的修改，此处皆可见
        // 此例中，x==10
    }
    );
    // 此处对共享变量 x修改
    x = 10;
    // 主线程启动子线程
    t1.start();
}
```

## join规则

join规则，如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于 线程A从ThreadB.join()操作成功的返回。

```java
Thread t1 = new Thread(()->{
    // 此处对共享变量 x 修改
    x= 100;
});
// 例如此处对共享变量修改，
// 则这个修改结果对线程 t1 可见
// 主线程启动子线程
t1.start();
t1.join()
// 子线程所有对共享变量的修改
// 在主线程调用 t1.join() 之后皆可见
// 此例中，x==100
```

# 四、DCL问题

//instance=new DCLExample();
- 为对象分配内存
- 初始化对象
- 把内存空间的地址复制给对象的引用

指令重排序后
- 为对象分配内存
- 把内存空间的地址复制给对象的引用
- 初始化对象(还没有执行的时候。

造成不完整对象


# 五、J.U.C

Java.util.Concurrent


# 六、Lock ->synchronized

锁是用来解决线程安全问题的


# 七、AbstractQueuedSynchronizer(AQS)

## 什么是AQS
- 共享锁
- 互斥锁

AQS 是一种抽象的队列同步器, 它是java除了synchroized之外自带的一种锁机制, 底层使用了大量的CAS进行互斥.

## AQS原理
以ReentrantLock为例, 在我们执行lock()方法进行上锁的时候, 如果有多个线程同时访问上锁的资源, 其底层会对AQS中的共享变量state进行一个CAS操作, 该操作是具有原子性, 如果线程A通过CAS将state状态修改成功就会获取到锁, 并记录一个共享变量设为A线程独享,则线程B尝试获取锁时由于锁状态state已经发生改变不再为0, 则就会把线程B存入一个FIFO的双向链表中, 该链表主要存储的是没有获取到锁的线程, 然后将里面的线程阻塞, 等到线程A释放锁完毕之后, 再唤醒队列中等待的线程. 
    
由于AQS是自旋锁, 在等待唤醒的时候, 会不停的使用while(cas())的方式, 不停的尝试获取锁. 

## AQS中为什么采用双向链表，它和单向链表相比，有什么优势？

ASQ中使用双向链表更容易访问相邻的节点. 


# 八、ReentrantLock

**重入锁** -> 互斥锁


## ReentrantLock的实现原理

满足线程的互斥特性

意味着同一个时刻，只允许一个线程进入到加锁的代码中。 -> 多线程环境下，线程的顺序访问。

### 锁的设计猜想（如果我们自己去实现）

- 一定会设计到锁的抢占 ， 需要有一个标记来实现互斥。 全局变量（0，1）
- 抢占到了锁，怎么处理（不需要处理.）
- 没抢占到锁，怎么处理
	* 需要等待（让处于排队中的线程，如果没有抢占到锁，则直接先阻塞->释放CPU资源）。
		- 如何让线程等待？
		- wait/notify(线程通信的机制，无法指定唤醒某个线程)
		- LockSupport.park/unpark（阻塞一个指定的线程，唤醒一个指定的线程）
		- Condition
	* 需要排队（允许有N个线程被阻塞，此时线程处于活跃状态）。
		- 通过一个数据结构，把这N个排队的线程存储起来。
- 抢占到锁的释放过程，如何处理
	* LockSupport.unpark() -> 唤醒处于队列中的指定线程.\
- 锁抢占的公平性（是否允许插队）
	* 公平
	* 非公平

### ReentrantLock的实现原理分析

![](/images/img-154.png)