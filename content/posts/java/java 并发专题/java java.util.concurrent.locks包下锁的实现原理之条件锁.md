---
layout: post
title: "java.util.concurrent.locks包下锁的实现原理之条件锁"
date: 2022-10-5
categories: 
    - "java"
tags: [java, 并发]
author: "pan"
---

[上一篇](https://www.jianshu.com/p/94b6cfbc3e3d) 文章中我们分析了ReentrantLock的实现原理，今天在分析一下条件锁。条件锁的具体实现在AbstractQueuedSynchronizer的内部类ConditionObject类中，总之，java中的锁，离不开AQS的实现。条件锁一般如下使用。

```java
 class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length) 
         notFull.await();
       items[putptr] = x; 
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0) 
         notEmpty.await();
       Object x = items[takeptr]; 
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   } 
 }
```

## Condition锁的概念

&emsp;&emsp;Condition主要是为了在J.U.C框架中提供和Java传统的监视器风格的wait，notify和notifyAll方法类似的功能。

**JDK的官方解释如下：**

&emsp;&emsp;条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。所以调用Condition的await和signal方法一定要放在lock和unlock代码块中间。
&emsp;&emsp;在分析ReentrantLock的实现时，提到了一个队列，暂且称为AQS队列或者同步队列吧。在Condition的具体实现也有一个队列，暂且称为条件队列。**两者是相对独立的队列，因此一个Lock可以有多个Condition，Lock(AQS)的队列主要是阻塞线程的，而Condition的队列也是阻塞线程，但是它是有阻塞和通知解除阻塞的功能**
Condition阻塞时会释放Lock的锁，阻塞流程请看下面的Condition的await()方法。
## Condition锁的实现思路
Condition锁的具体实现需要借助AQS队列和Condition队列。阻塞节点会在AQS队列和Condition队列中转移。具体的实现在await和signal方法中。
+ await()就是在当前线程持有锁的基础上释放锁资源，并新建Condition节点加入到Condition的队列尾部，阻塞当前线程
+ signal()就是将Condition的头节点移动到AQS等待节点尾部，让其等待再次获取锁
### AQS队列和Condition队列的出入结点的示意图
示意图copy的[这篇博文](https://www.cnblogs.com/cm4j/p/juc_condition.html),感谢博主的分享
+ 初始化状态：AQS等待队列有3个Node，Condition队列有1个Node
![image.png](/java java.util.concurrent.locks包下锁的实现原理之条件锁/8596800-d68651a9c415b992.png)
+ 节点1执行Condition.await()
![image.png](/java java.util.concurrent.locks包下锁的实现原理之条件锁/8596800-7eef288636a718d8.png)
步骤说明：
1、添加一个新的节点到Condition队列的末尾（相当于把AQS节点的头结点移动到Condition队列的末尾）。
2、释放锁，唤醒AQS队列上阻塞的线程，AQS队列head指针后移
3、更新lastWaiter为节点1
+ 节点2执行signal()操作
![image.png](/java java.util.concurrent.locks包下锁的实现原理之条件锁/8596800-41b5773bb81618b0.png)
1、将firstWaiter后移
2、将节点4移出Condition队列
3、将节点4加入到AQS的等待队列中去
4、更新AQS的等待队列的tail
下面具体的分析一下代码
## 加锁
Condition锁加锁的代码在await方法中
```
 public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            //创建新的节点，加到Condition队列的末尾
            Node node = addConditionWaiter();
            //在ReentrantLock中已经讲过该方法，释放锁，让AQS队列中等待的线程继续执行
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //自旋操作，因为此时有可能其他线程执行了signal操作，node节点可能已经有Condition队列移除到AQS队列。所以需要这个判断
            while (!isOnSyncQueue(node)) {
            // 如果该节点不在AQS队列，在Conditioin队列，那么阻塞线程
                LockSupport.park(this);
            //设置中断处理方式
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 注意执行到这里了，肯定有其他线程执行了signal操作，acquireQueued方法之前在ReentrantLock中也分析过了。获取锁，参与锁的竞争
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
## 解锁
Condition锁解锁在调用signal方式，signal和signalAll方法的区别在于，signalAll将唤醒所有的线程参与锁的竞争，并将Condition队列中所有的节点都放到AQS队列中。
```
public final void signal() {
            // 判断锁的持有者
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```
```
        private void doSignal(Node first) {
            do {
                //firstWaiter 指针后移，从Condition队列中移除头结点
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```
```
final boolean transferForSignal(Node node) { 
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false; 
       //Condition队列移到AQS队列
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
           // 唤醒线程，要注意线程在那里被唤醒了（await中park的地方）
            LockSupport.unpark(node.thread);
        return true;
    }
```
如果ReentrantLock的实现认真分析了，那么Condition的实现代码看起来就比较简单了，如果Condition的代码看起来吃力，请回过来看上一篇ReentrantLock的文章

## 参考文献
+ [JUC.Condition学习笔记[附详细源码解析](https://www.cnblogs.com/cm4j/p/juc_condition.html)
]([https://www.cnblogs.com/cm4j/p/juc_condition.html](https://www.cnblogs.com/cm4j/p/juc_condition.html)
)



