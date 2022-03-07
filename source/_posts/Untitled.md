title: AQS
author: Peilin Deng
tags:
  - 并发编程
  - JUC
  - 锁
  - 线程安全
  - 数据结构
categories:
  - 摘抄笔记
date: 2022-03-05 22:55:00
---
# 一、AbstractQueuedSynchronizer(AQS)

## 什么是AQS
- 共享锁
- 互斥锁

AQS 是一种抽象的队列同步器, 它是java除了synchroized之外自带的一种锁机制, 底层使用了大量的CAS进行互斥.

## AQS原理
以ReentrantLock为例, 在我们执行lock()方法进行上锁的时候, 如果有多个线程同时访问上锁的资源, 其底层会对AQS中的共享变量state进行一个CAS操作, 该操作是具有原子性, 如果线程A通过CAS将state状态修改成功就会获取到锁, 并记录一个共享变量设为A线程独享,则线程B尝试获取锁时由于锁状态state已经发生改变不再为0, 则就会把线程B存入一个FIFO的双向链表中, 该链表主要存储的是没有获取到锁的线程, 然后将里面的线程阻塞, 等到线程A释放锁完毕之后, 再唤醒队列中等待的线程. 
    
由于AQS是自旋锁, 在等待唤醒的时候, 会不停的使用while(cas())的方式, 不停的尝试获取锁. 

## AQS中为什么采用双向链表，它和单向链表相比，有什么优势？

ASQ中使用双向链表更容易访问相邻的节点. 


# 二、Lock

**重入锁** -> 互斥锁

## lock()
- 公平锁和非公平锁

	- 公平锁
 	```java
  final void lock() {
      acquire(1);
      //抢占1把锁 .
  }
  public final void acquire(int arg) {
      -> AQS里面的方法
      if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
  }
  protected final Boolean tryAcquire(int acquires) {
      final Thread current = Thread.currentThread();
      int c = getState();
      if (c == 0) {
          //表示无锁状态
          if (!hasQueuedPredecessors() &&
          compareAndSetState(0, acquires)) {
              //CAS（#Lock） -> 原子操作| 实现互斥 的判断
              setExclusiveOwnerThread(current);
              //把获得锁的线程保存到 exclusiveOwnerThread中
              return true;
          }
      }
      //如果当前获得锁的线程和当前抢占锁的线程是同一个，表示重入 else if (current == getExclusiveOwnerThread()) {
          int nextc = c + acquires;
          //增加重入次数 .      if (nextc < 0)
          throw new Error("Maximum lock count exceeded");
          setState(nextc);
          //保存state
          return true;
      }
      return false;
  }
```

  - 非公平锁
  ```java
  final void lock() {
      //不管当前AQS队列中是否有排队的情况，先去插队
      if (compareAndSetState(0, 1)) //返回false表示抢占锁失败 setExclusiveOwnerThread(Thread.currentThread()); else
      acquire(1);
  }
  public final void acquire(int arg) {
      --AQS
      if (!tryAcquire(arg) &&                             acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
  }
  protected final Boolean tryAcquire(int acquires) {
      return nonfairTryAcquire(acquires);
  }
  final Boolean nonfairTryAcquire(int acquires) {
      final Thread current = Thread.currentThread();
      int c = getState();
      if (c == 0) {
          //hasQueuedPredecessors
          if (compareAndSetState(0, acquires)) {
              setExclusiveOwnerThread(current);
              return true;
          }
      } else if (current == getExclusiveOwnerThread()) {
          int nextc = c + acquires;
          if (nextc < 0) // overflow
          throw new Error("Maximum lock count exceeded");
          setState(nextc);
          return true;
      }
      return false;
  }
  ```

## 加入队列并进行自旋等待
  - acquire(int arg)
  ```java
  public final void acquire(int arg) {
      if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
  }
  ```

  - acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
    - addWaiter(Node.EXCLUSIVE) -> **添加一个互斥锁的节点**
    ```java
      private Node addWaiter(Node mode) {
          //把当前线程封装成一个Node节点。
          Node node = new Node(Thread.currentThread(), mode);
          //后续唤醒线程的时候，需要 得到被唤醒的线程 .
          // Try the fast path of enq; backup to full enq on failure
          Node pred = tail;
          //假设不存在竞争的情况
          if (pred != null) {
              node.prev = pred;
              if (compareAndSetTail(pred, node)) {
                  pred.next = node;
                  return node;
              }
          }
          enq(node);
          return node;
      }
      private Node enq(final Node node) {
          for (;;) {
              //自旋
              Node t = tail;
              if (t == null) {
                  // Must initialize //初始化一个head节点
                  if (compareAndSetHead(new Node())) tail = head;
              } else {
                  node.prev = t;
                  if (compareAndSetTail(t, node)) {
                      = node;
                      t;
                  }
              }
          }
      }
      ```

    - acquireQueued() -> **自旋锁和阻塞的操作**
    ```java
    //node表示当前来抢占锁的线程，有可能是ThreadB、 ThreadC。。
    final Boolean acquireQueued(final Node node, int arg) {
        Boolean failed = true;
        try {
            Boolean interrupted = false;
            for (;;) {
                //自旋
                //begin  ->尝试去获得锁(如果是非公平锁)
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果返回true，则不需要等待，直接返 回。
                    setHead(node);
                    p.next = null;
                    // help GC
                    failed = false;
                    return interrupted;
                }
                //end
                //否则，让线程去阻塞(park)
                if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) //LockSupport.park
                interrupted = true;
            }
        }
        finally {
            if (failed)
            cancelAcquire(node);
        }
    }
    ```
    ```java
    //ThreadB、 ThreadC、ThreadD、ThreadE  -> 都会阻塞在下面这个代码的位置 .
    private final Boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //被唤醒 . (interrupt()->)
        return Thread.interrupted(); //中断状态（是否因为中断被唤醒的 .）
    }
    ```

## unlock
  - release(int arg)
  ```java
  public final Boolean release(int arg) {
      if (tryRelease(arg)) {
          Node h = head;
          //得到当前AQS队列中的head节点。
          if (h != null && h.waitStatus != 0) //head节点不为空
          unparkSuccessor(h);
          //
          return true;
      }
      return false;
  }
  ```

  - unparkSuccessor(Node node)
  ```java
  private void unparkSuccessor(Node node) {
      int ws = node.waitStatus;
      if (ws < 0) //表示可以唤醒状态
      compareAndSetWaitStatus(node, ws, 0);
      //恢复成0
      Node s = node.next;
      if (s == null | | s.waitStatus > 0) {
          //说明ThreadB这个线程可能已经被销毁，或 者出现异常 ...
          s = null;
          //从tail -> head进行遍历 .
          for (Node t = tail; t != null && t != node; t = t.prev)
          if (t.waitStatus <= 0) //查找到小于等于0的节点
          s = t;
      }
      if (s != null)
      LockSupport.unpark(s.thread);
      //封装在Node中的被阻塞的线程。ThreadB、
      ThreadC。
  }
  ```

## 1. ReentrantLock

### ReentrantLock的实现原理

满足线程的互斥特性

意味着同一个时刻，只允许一个线程进入到加锁的代码中。 -> 多线程环境下，线程的顺序访问。

#### 锁的设计猜想（如果我们自己去实现）

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

#### ReentrantLock的实现原理分析

![](/images/img-154.png)

## 2. Wait/Notify
线程的通信 -> 共享内存

Wait/Notify 不是 J.U.C 包下的 -> 基于某个条件来等待或者唤醒

![](/images/img-155.png)

### 基于Wait/Notify 实现生产者/消费者
```java
public class Consumer implements Runnable{
    private Queue<String> bags;
    private int maxSize;
    public Consumer(Queue<String> bags, int maxSize) {
        this.bags = bags;
        this.maxSize = maxSize;
    }
    @Override
    public void run() {
        while(true){
            synchronized (bags){
                if(bags.isEmpty()){
                    System.out.println("bags为空");
                    try {
                        bags.wait();
                    }
                    catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                try {
                    Thread.sleep(1000);
                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String bag=bags.remove();
                System.out.println("消费者消费： "+bag);
                bags.notify(); //这里只是唤醒Producer线程，但是Producer线程并不能马上执行。
            } //同步代码块执行结束， monitorexit指令执行完成
        }
    }
}
```
```java
public class Producer implements Runnable {
    private Queue<String> bags;
    private int maxSize;
    public Producer(Queue<String> bags, int maxSize) {
        this.bags = bags;
        this.maxSize = maxSize;
    }
    @Override
    public void run() {
        int i=0;
        while(true){
            i++;
            synchronized (bags){
                //抢占锁
                if(bags.size()==maxSize){
                    System.out.println("bags 满了");
                    try {
                        //park(); ->JVM ->Native
                        bags.wait();
                        //满了，阻塞当前线程并且释放Producer抢到的锁 } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                try {
                    Thread.sleep(1000);
                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("生产者生产：ag"+i);
                bags.add("bag"+i);
                //生产bag
                bags.notify();
                //表示当前已经生产了数据，提示消费者可以消费了 } //同步代码快执行结束
            }
        }
    }
}
```
```java
public static void main(String[] args){
    Queue<String> message = new LinkedList<>();
    Producer producer = new Producer(message, 10);
    Consumer consumer = new Consumer(message, 10);
    producer.start();
    try {
        Thread.sleep(100);
    }
    catch (InterruptedException e) {
        e.printStackTrace();
    }
    consumer.start();
}
```

## 3. join
join 也是基于  wait/notify来实现，notify是在线程销毁之后调用的，代码如下。
```java
static void ensure_join(JavaThread* thread) {
    // We do not need to grap the Threads_lock, since we are operating on ourself.
    Handle threadObj(thread, thread->threadObj());
    assert(threadObj.not_null(), "java thread object must exist");
    ObjectLocker lock(threadObj, thread);
    // Ignore pending exception (ThreadDeath), since we are exiting anyway
    thread->clear_pending_exception();
    // Thread is exiting. So set thread_status field in  java.lang.Thread class to TERMINATED.
    java_lang_Thread::set_thread_status(threadObj(),
    java_lang_Thread::TERMINATED);
    // Clear the native thread instance - this makes isAlive return false and allows the join()
    // to complete once we've done the notify_all below
    java_lang_Thread::set_thread(threadObj(), NULL);
    lock.notify_all(thread);
    // Ignore pending exception (ThreadDeath), since we are exiting anyway
    thread->clear_pending_exception();
}
```

## 4. Condition

Condition 实际上就是 J.U.C 版本的 wait/notify 。可以让线程基于某个条件去等待和唤醒。

```java
public class ConditionDemoWait implements Runnable{
    private Lock lock;
    private Condition condition;
    public ConditionDemoWait(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }
    @Override
    public void run() {
        System.out.println("begin - ConditionDemoWait");
        lock.lock();
        try{
            condition.await();
            //让当前线程阻塞，Object.wait();
            System.out.println("end - ConditionDemoWait");
        }
        catch (Exception e){
            e.printStackTrace();
        }
        finally {
            lock.unlock();
        }
    }
}
```
```java
public class ConditionDemeNotify implements Runnable{
    private Lock lock;
    private Condition condition;
    public ConditionDemeNotify(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }
    @Override
    public void run() {
        System.out.println("begin - ConditionDemeNotify");
        lock.lock();
        //synchronized(lock)
        try{
            condition.signal();
            //让当前线程唤醒  Object.notify(); //因为任何对象都会
            有monitor
            System.out.println("end - ConditionDemeNotify");
        }
        catch (Exception e){
            e.printStackTrace();
        }
        finally {
            lock.unlock();
        }
    }
}
```
### Condition设计猜想
- 作用：   实现线程的阻塞和唤醒
- 前提条件：   必须先要获得锁
- await/signal;signalAll
   - await -> 让线程阻塞，   并且释放锁
   - signal -> 唤醒阻塞的线程
- 加锁的操作，必然会涉及到AQS的阻塞队列
- await 释放锁的时候，  -> AQS队列中不存在已经释放锁的线程 -> 这个被释放的线程去了哪里？ 
- signal 唤醒被阻塞的线程 -> 从哪里唤醒？

> 通过await方法释放的线程，必须要有一个地方来存储，并且还需要被阻塞；   -> 会存在一个等待队列，  LockSupport.park阻塞 

>signal -> 上面 猜想到的 等待队列中，唤醒一个线程，放哪里去？是不是应该再放到AQS队列？


## 5. CountDownLatch （倒计时器）
① 某一线程在开始运行前等待n个线程执行完毕。将 CountDownLatch 的计数器初始化为n ：new CountDownLatch(n)，每当一个任务线程执行完毕，就将计数器减1 countdownlatch.countDown()，当计数器的值变为0时，在CountDownLatch上 await() 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

②实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 CountDownLatch 对象，将其计数器初始化为 1 ：new CountDownLatch(1)，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

### CountDownLatch 的三种典型用法

- ① 某一线程在开始运行前等待n个线程执行完毕。将 CountDownLatch 的计数器初始化为n ：new CountDownLatch(n)，每当一个任务线程执行完毕，就将计数器减1 countdownlatch.countDown()，当计数器的值变为0时，在CountDownLatch上 await() 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

- ② 实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 CountDownLatch 对象，将其计数器初始化为 1 ：new CountDownLatch(1)，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

- ③ 死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。

### CountDownLatch 的使用示例
```java
public class CountDownLatchExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          test(threadnum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } finally {
          countDownLatch.countDown();// 表示一个请求已经被完成
        }

      });
    }
    countDownLatch.await();
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```
上面的代码中，我们定义了请求的数量为550，当这550个请求被处理完成之后，才会执行System.out.println("finish");。

与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

其他N个线程必须引用闭锁对象，因为他们需要通知CountDownLatch对象，他们已经完成了各自的任务。这种通知机制是通过 CountDownLatch.countDown()方法来完成的；每调用一次这个方法，在构造函数中初始化的count值就减1。所以当N个线程都调 用了这个方法，count的值等于0，然后主线程就能通过await()方法，恢复执行自己的任务。

### CountDownLatch 的不足

CountDownLatch是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当CountDownLatch使用完毕后，它不能再次被使用。

### CountDownLatch相常见面试题：

解释一下CountDownLatch概念？

CountDownLatch 和CyclicBarrier的不同之处？

给出一些CountDownLatch使用的例子？

CountDownLatch 类中主要的方法？



