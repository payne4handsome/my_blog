---
layout: post
title: "java.util.concurrent.locks包下锁的实现原理之读写锁"
date: 2022-10-5
categories: 
    - "java"
tags: [java, 并发]
author: "pan"
---

java并发包已经存在[Reentrant锁](https://www.jianshu.com/p/94b6cfbc3e3d)和[条件锁](https://www.jianshu.com/p/7af838a0b8d7)，已经可以满足很多并发场景下线程安全的需求。但是在大量读少量写的场景下，并不是最优的选择。与传统锁不同的是读写锁的规则是可以共享读，但只能一个写。很多博客中总结写到**读读不互斥，读写互斥，写写互斥**。就读写这个场景下来说，如果一个线程获取了写锁，然后再获取读锁（同一个线程）也是可以的。锁降级就是这种情况。但是如果也是同一个线程，先获取读锁，再获取写锁是获取不到的（发生死锁）。所以严谨一点情况如下：
| 项目        | 非同一个线程  |  同一个线程 |
| :--------:   | :-----:  | :----:  |
|读读|不互斥|不互斥|
|读写|互斥|锁升级（不支持），**发生死锁**|
|写读|互斥|锁降级（支持），不互斥|
|写写|互斥|不互斥|
读写锁的主要特性：
+ 公平性：支持公平性和非公平性。
+ 重入性：支持重入。读写锁最多支持 65535 个递归写入锁和 65535 个递归读取锁。
+ 锁降级：遵循获取写锁，再获取读锁，最后释放写锁的次序，如此写锁能够降级成为读锁。
## ReentrantReadWriteLock
java.util.concurrent.locks.ReentrantReadWriteLock ，实现 ReadWriteLock 接口，可重入的读写锁实现类。在它内部，维护了一对相关的锁，一个用于只读操作（共享锁），另一个用于写入操作（排它锁）。
ReentrantReadWriteLock 类的大体结构如下：
```
/** 内部类  读锁 */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类  写锁 */
private final ReentrantReadWriteLock.WriteLock writerLock;

final Sync sync;

/** 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock() {
    this(false);
}

/** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

/** 返回用于写入操作的锁 */
@Override
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
/** 返回用于读取操作的锁 */
@Override
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

abstract static class Sync extends AbstractQueuedSynchronizer {
    /**
     * 省略其余源代码
     */
}
public static class WriteLock implements Lock, java.io.Serializable {
    /**
     * 省略其余源代码
     */
}

public static class ReadLock implements Lock, java.io.Serializable {
    /**
     * 省略其余源代码
     */
}
```
+ ReentrantReadWriteLock 与 ReentrantLock一样，其锁主体也是 Sync，它的读锁、写锁都是通过 Sync 来实现的。所以 ReentrantReadWriteLock 实际上只有一个锁，只是在获取读取锁和写入锁的方式上不一样。
+ 它的读写锁对应两个类：ReadLock 和 WriteLock 。这两个类都是 Lock 的子类实现。

在 ReentrantLock 中，使用 Sync ( 实际是 AQS )的 int 类型的 state 来表示同步状态，表示锁被一个线程重复获取的次数。但是，读写锁 ReentrantReadWriteLock 内部维护着一对读写锁，如果要用一个变量维护多种状态，需要采用“按位切割使用”的方式来维护这个变量，将其切分为两部分：高16为表示读，低16为表示写。

分割之后，读写锁是如何迅速确定读锁和写锁的状态呢？通过位运算。假如当前同步状态为S，那么：
+ 写状态，等于 S & 0x0000FFFF（将高 16 位全部抹去）
+ 读状态，等于 S >>> 16 (无符号补 0 右移 16 位)。
![image.png](/java java.util.concurrent.locks包下锁的实现原理之读写锁/8596800-06d27edad82c7d0e.png)
## 关键属性
读写锁由于要读读是共享的，所以实现比ReentrantLock复杂的多。在ReentrantReadWriteLock类中有几个成员属性，先单独提出来，有助于理解下面的源码分析。
```
//每个线程读锁持有数量
 static final class HoldCounter {
            int count = 0;
            // 这个getThreadId用到了UNSAFE，why？Thread本身是有getId可以获取线程id的，这里为什么要用UNSAFE是为了防止重写
            final long tid = getThreadId(Thread.currentThread());
        }

// ThreadLocal类型，这样HoldCounter 就可以与线程进行绑定。获取当前线程的HoldCounter
 private transient ThreadLocalHoldCounter readHolds;

//缓存的上一个读取线程的HoldCounter
 private transient HoldCounter cachedHoldCounter;
// 第一个获得读锁的线程（最后一个把共享锁计算从0变到1的线程）
 private transient Thread firstReader = null;
// firstReader 持有读锁的数量
 private transient int firstReaderHoldCount;
```
## 读锁获取
获取读锁调用ReentrantReadWriteLock#readLock#lock就可以，来看实现
```
 public void lock() {
             //acquireShared获取共享锁，实现在AQS的类中
            sync.acquireShared(1);
        }
```
```
 public final void acquireShared(int arg) {
      //获取读锁
        if (tryAcquireShared(arg) < 0)
            //读锁获取失败，阻塞等待
            doAcquireShared(arg);
    }
```
关键的代码在tryAcquireShared方法中
```
protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            //如果已经存在了写锁，并且获取写锁的不是当前线程，那么返回，读锁获取失败。注意获取写锁的是当前线程，是可以重入的（锁降级）
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
           //获取读锁数量（读锁是共享的，所以这个数量是所有线程读锁的累加和），位运算
            int r = sharedCount(c);
            //判断读锁是否需要阻塞，注意公平锁和非公平不一样
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                //如果下面的CAS设置成功了，那么读锁也就获取成功了
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                   // 设置firstReader
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //同一个线程重入
                    firstReaderHoldCount++;
                } else {
                    // 设置cachedHoldCounter、readHolds
                    HoldCounter rh = cachedHoldCounter;// 最后一个成功获取读锁的线程
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)// 为什么rh.count会等于0？当一个线程获取读锁释放，再次获取读锁，就是这种情况
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```
### readerShouldBlock的公平性和非公平性
+ 非公平性的readerShouldBlock
对于非公平性的读锁，为了防止写锁饿死，需要判断AQS队列的第一个等待锁的节点是不是写锁，如果是写锁，那么读锁让步；如果不是写锁，那么可以竞争。
```
 final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```
+ 公平性的readerShouldBlock
ReentrantLock中已经分析过这个方法，不是AQS队列中节点是读锁还是写锁都加到队列末尾等待，完全公平。
```
  public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
fullTryAcquireShared的代码和tryAcquireShared存在一定程度冗余，我个人觉得不要tryAcquireShared那一段代码也是可以的，这个如果哪位同学有更深的理解可以交流下。总之，fullTryAcquireShared是tryAcquireShared自旋重试的版本, 不再赘述
```
final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```
## 读锁释放
```
  public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
tryReleaseShared比较简单，主要就是修改firstReader、readHolds、state的值
```
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //释放firstReader
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```
唤醒后继节点
```
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
## 写锁获取

## 写锁释放

## 锁降级


## 参考资料
【1】[http://www.iocoder.cn/JUC/sike/ReentrantReadWriteLock/](http://www.iocoder.cn/JUC/sike/ReentrantReadWriteLock/)

