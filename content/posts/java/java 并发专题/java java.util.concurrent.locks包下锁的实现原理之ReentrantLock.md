---
layout: post
title: "java.util.concurrent.locks包下锁的实现原理之ReentrantLock"
date: 2022-10-5
categories: 
    - "java"
tags: [java, 并发]
author: "pan"
---

## 一、java concurrent包下lock类图概览
**红色连线的表示内部类**
![image.png](/java java.util.concurrent.locks包下锁的实现原理之ReentrantLock/8596800-037daeafe21e322b.png)
1、java并发包下面的锁主要就两个，ReentrantLock（实现Lock接口） 和ReentrantReadWriteLock（实现ReadWriteLock接口）。
2、ReentrantLock类构造函数如下, sync是Sync的实例，NonfairSync（非公平锁）和FairSync(公平锁)是Sync的子类。
```
public ReentrantLock() {
        sync = new NonfairSync();
    }
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
3、ReentrantReadWriteLock类构造函数如下，共有三个属性，sync、readerLock、writerLock
```
 public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```
我们看到ReentrantLock和ReentrantReadWriteLock都的实现都依赖于sync这个对象。sync是AbstractQueuedSynchronizer的实例。AbstractQueuedSynchronizer就是java并发包下面实现锁和线程同步的基础，AbstractQueuedSynchronizer就是大名鼎鼎的AQS队列，下文我们都用AQS来表示AbstractQueuedSynchronizer。
##二、ReentrantLock实现原理
### 1、如何加锁
ReentrantLock使用方式如下
```
class X {
    private final ReentrantLock lock = new ReentrantLock();
    // ...
    public void m() {
      lock.lock();  // block until condition holds
      try {
        // ... method body
      } finally {
        lock.unlock()
      }
    }
  }}
```
### lock方法的实现原理
ReentrantLock锁的实现分为公平锁(FairSync)和非公平锁(NonFairSync)，所以lock方法的实现自然有两个版本
#### 非公平锁中和公平锁中lock的实现
|非公平锁中lock的实现 |公平锁中lock的实现|
|---|---|
|final void lock() { <br>  &emsp; if (compareAndSetState(0, 1))<br>&emsp;&emsp; &emsp;setExclusiveOwnerThread(Thread.currentThread());<br>&emsp;else<br>&emsp;  &emsp;acquire(1);<br>}  |final void lock() {<br>&emsp;&emsp;acquire(1);<br> }|
acquire是AQS中方法，执行过程中还是调用的子类的实现方法，所以不要简单地以为公平锁和非公平锁的实现就如表格中那么小的差异。
##### 非公平锁代码解析
-  lock方法

```
final void lock() { 
  if (compareAndSetState(0, 1)) //设置AQS中state的值，如果state当前是0（无锁），那么设置state为1，加锁成功
    setExclusiveOwnerThread(Thread.currentThread());// 设置独占锁的拥有线程
 else
   acquire(1);
}
```
  -  acquire方法
```
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
tryAcquire方法试着加锁，主要实现在nonfairTryAcquire中
```
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) { // 当前没有锁竞争，那么CAS方式加锁
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {// 如果当前锁的拥有者急速当前线程，因为是ReentrantLock是可重入的，所以state加1，获得锁
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
如果tryAcquire加锁没有成功，那么那么需要将当前线程加入AQS队列，并且阻塞。先看addWaiter 方法
```
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) { // 如果队列已经初始化，那么加入到队列末尾
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);// 一开队列为空，那么初始化队列，并且入队列
        return node;
    }
```
enq方法是入队列操作，改方法设计的很巧妙，多线程情况下通过CAS操作保证线程安全。该方法是一个死循环，在设置头结点和尾结点时候都用了CAS操作，这样就保证了设置头尾结点只有一个线程可以设置成功，CAS操作失败的再次进入循环，加到上一个CAS操作成功结点的后面，注意这个队列的头结点是空结点
```
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))// CAS操作初始化队列，头结点是空结点
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {// CAS操作设置尾结点，CAS操作失败的依次加入到队列的末尾
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
以上只是入队列操作，那么阻塞线程的逻辑在哪里呢？我们来看acquireQueued方法
```
inal boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();// 获得该节点的前驱，因为队列已经初始化，那么该队列肯定不是空的了
                if (p == head && tryAcquire(arg)) {// 有可能该节点已经变成了头结点，那么再次试着去获得锁，获取成功就不用阻塞了
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
// 不是头结点
                if (shouldParkAfterFailedAcquire(p, node) &&// 设置结点的状态为阻塞
                    parkAndCheckInterrupt()) // 阻塞线程
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
shouldParkAfterFailedAcquire方法，一开始Node结点的waitStatus肯定是0，那么设置结点的waitStatus为SIGNAL（-1），也就是需要unparking
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
执行到这里了，中途其实是有多次机会让现场获得锁的，但是如果还没有成功，那么我只能阻塞啦，这个就不能怪我手下无情了，好，那么我们看阻塞的代码
```
 private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
这里阻塞线程调用的是 LockSupport提供的park操作，park操作的实现就需要调用底层操作系统的阻塞原语啦。
> 到这里我们已经把非公平锁的思路将明白了，整个代码写的可以说是滴水不漏，构思巧妙，用到了大量的CAS操作，这也是为什么说Lock是乐观锁的原因
##### 公平锁代码解析
```
final void lock() {
            acquire(1); //lock的代码是不是比非公平锁的代码少了一点啥，自已比较下
        }
```
>下面这个方法看起来和非公平锁中是一样的，其实差别就在tryAcquire方法的具体实现上不一样，其他的地方是一样的，tryAcquire是调用的子类FairSync中的tryAcquire方法

```java
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

```
tryAcquire方法如下：
``` java
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
>细心的同学发现这段代码仅仅比非公平锁中tryAcquire方法中多了一个判断,就是下面的方法，下面的方法就是判断队列中是不是还有其他线程结点在等待。如果没有，那么当前线程可以抢占锁，如果有那么，你需要乖乖的排到队列的最后等待并且被阻塞。
```
 public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

```
##### 公平锁与非公平锁总结
>**公平锁和非公平锁的公平性和非公平主要体现在新调用lock方法的线程是不是可以去抢占锁（也就是state=0时，可不可去设置为1），非公平锁是不是可以通过CAS操作去设置state为1的；但是公平锁需要先判断AQS队列中有没有结点阻塞，如果没有，那么该线程可以设置state为1，如果有，那么乖乖去AQS队列末尾等待并且阻塞。**

这里有的同学可能会有疑问，可以错误的以为state=0那么AQS队列就是空。这个错误的。state=0，AQS可能有有很多线程结点在等待的。我们看一下unlock方法就知道了，我们前面花了很多的篇幅去讨论lock方法。unclock比较简单，我们来看下
### 解锁
无论公平锁还是非公平锁，unlock 都是调用的AQS中的release方法。
```
 public void unlock() {
        sync.release(1);
    }
```
release方法中先tryRelease一下
```
   public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
tryRelease中主要设置state的值，一般也就是把state=1，设置为0。表示当前是没有锁啦，你们可以来抢占啦。我们刚刚又说道state=0,AQS队列中可能是有很多结点还在等待中的。如果这个时候我们刚unlock了一下，刚刚把state设置0，但是还没有唤醒在队列上等待的线程。那么如果是非公平锁，那么新的线程很有可能就是抢到锁了（CAS把state设置为1）。
```
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
state 也设置为0，那么我们是不是应该把阻塞在队列中的线程唤醒了，从AQS队列的头结点开始唤醒，也就是执行下面的unparkSuccessor方法，同样的调用LockSupport中unpark函数。
```
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
一口气说了这么多，那么线程在哪里被唤醒呢？当然是在哪里阻塞在哪里唤醒，我们回到阻塞的地方。现在是在acquireQueued中parkAndCheckInterrupt方法中被阻塞的。
```
 */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
我们注意这里是一个死循环，当线程被唤醒，并且是头结点（我们是从第一个结点开始唤醒的），那么又调用tryAcquire方法参与锁的竞争。说的这里ReentrantLock将说的差不多了，反正就是一直竞争啊竞争争。。。。。累

### 条件锁

条件锁见我的另外一篇文章 [java java.util.concurrent.locks包下锁的实现原理之条件锁](https://www.jianshu.com/p/7af838a0b8d7)

### 读写锁
读写锁见我的另外一篇文章 [java java.util.concurrent.locks包下锁的实现原理之读写锁](https://www.jianshu.com/p/f51b84d7f78f)

