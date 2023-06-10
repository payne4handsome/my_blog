---
layout: post
title: "JAVA并发编程 线程池Executors（JDK1.7）"
date: 2022-10-5
categories: 
    - "java"
tags: [java, 并发]
author: "pan"
---


>Java中对线程池提供了很好的支持，有了线程池，我们就不需要自已再去创建线程。如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。JAVA的线程池中的线程可以在执行完任务后，不销毁，继续执行其他的任务。所以了解Java的线程池对我们掌握并发编程是很有帮助的。

先看一下线程池框架Executors涉及到的核心类。
![白虚线表示依赖、蓝线继承、青色实现](/JAVA并发编程：线程池Executors（JDK1.7）/8596800-9c5f6450c3b45188.png)
+ Executor：父类，官方表述为用来解耦任务的提交，可以自已实现，比如调用线程执行该任务，或者起一个新的线程执行该任务
+ ExecutorService：比父类Executor定义了更多的接口用来提交、管理、终止任务
+ AbstractExecutorService：提供了ExecutorService默认实现
下面我就从Executors这个多线程框架开始讲起，首先看一下Executors中主要的方法
+ ThreadPoolExecutor：比AbstractExecutorService提供更多的功能，特别是大量的线程创建，销毁。在性能上更优越。
+ Executors：工厂类，提供了几个核心创建线程池的方法。

## Executors核心方法
```
public static ExecutorService newFixedThreadPool(int nThreads) {  
       return new ThreadPoolExecutor(nThreads, nThreads,  
                                     0L, TimeUnit.MILLISECONDS,  
                                     new LinkedBlockingQueue<Runnable>());  
   }  
   public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {  
       return new ThreadPoolExecutor(nThreads, nThreads,  
                                     0L, TimeUnit.MILLISECONDS,  
                                     new LinkedBlockingQueue<Runnable>(),  
                                     threadFactory);  
   }  
  
   public static ExecutorService newSingleThreadExecutor() {  
       return new FinalizableDelegatedExecutorService  
           (new ThreadPoolExecutor(1, 1,  
                                   0L, TimeUnit.MILLISECONDS,  
                                   new LinkedBlockingQueue<Runnable>()));  
   }  
   public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {  
       return new FinalizableDelegatedExecutorService  
           (new ThreadPoolExecutor(1, 1,  
                                   0L, TimeUnit.MILLISECONDS,  
                                   new LinkedBlockingQueue<Runnable>(),  
                                   threadFactory));  
   }  
  
   public static ExecutorService newCachedThreadPool() {  
       return new ThreadPoolExecutor(0, Integer.MAX_VALUE,  
                                     60L, TimeUnit.SECONDS,  
                                     new SynchronousQueue<Runnable>());  
   }  
   public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {  
       return new ThreadPoolExecutor(0, Integer.MAX_VALUE,  
                                     60L, TimeUnit.SECONDS,  
                                     new SynchronousQueue<Runnable>(),  
                                     threadFactory);  
   }  
```
如果我们要Exectutors来创建线程池，基本离不开上面几个方法，我们可以看出上面几个方法中都用到了ThreadPoolExecutor这个类，且都返回ExecutorService对象。我们先看一下ExecutorService的主要方法。
```
public interface ExecutorService extends Executor {  
    void shutdown();  
  
    List<Runnable> shutdownNow();  
  
    boolean isTerminated();  
  
    boolean awaitTermination(long timeout, TimeUnit unit)  
        throws InterruptedException;  
```

```
<T> Future<T> submit(Callable<T> task);  
  
<T> Future<T> submit(Runnable task, T result);  
  
Future<?> submit(Runnable task); 
```
+ shutdown() 启动一次顺序关闭，执行以前提交的任务，但不接受新任务
+ shutdownnow()试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表
+ submit()用来提交任务。
Executors一般如下使用
```
ExecutorService service = Executors.newFixedThreadPool(1);  
    for(int i=0;i<10;i++){  
        service.submit(new Task(i));  
    }  
    service.shutdown();  
```
Exectors调用静态方法，利用ThreadPoolExecutor来创建线程池，然后submit提交任务，我们看一下submit的代码
```
public Future<?> submit(Runnable task) {  
       if (task == null) throw new NullPointerException();  
       RunnableFuture<Object> ftask = newTaskFor(task, null);  
       execute(ftask);  
       return ftask;  
   }  
```
## ThreaPoolExecutor实现原理深入分析

我们先来看一下Executors线程池框架用到几个主要的类（接口）这件的关系。
ThreadPoolExecutor、AbstractExecutorService（抽象类）、ExecutorService（接口）和Executor（接口）几个之间的关系如下。

Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型，从字面意思可以理解，就是用来执行传进去的任务的；

然后ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

然后ThreadPoolExecutor继承了类AbstractExecutorService。

在ThreadPoolExecutor类中有几个非常重要的方法：
```java
execute()  
submit()  
shutdown()  
shutdownNow()  
```
execute()方法实际上是Executor接口中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。所以上面ExecutorService中submit()方法中调用的execute()方法是在ThreadPollExecutor中实现的。所以我们研究ThreadPoolExecutor这个类中首先从execute()这个方法开始，在开始之间先看一下ThreadPoolExecutor中的构造函数和一些主要的成员变量

ThreadPoolExecutor构造函数,在ThreadPoolExecutor类中提供了四个构造方法：
```java
    .....  
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,  
            BlockingQueue<Runnable> workQueue);  
  
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,  
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);  
  
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,  
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);  
  
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,  
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);  
    ...  
}  
```

从上面的代码可以得知，ThreadPoolExecutor继承了AbstractExecutorService类，并提供了四个构造器，事实上，通过观察每个构造器的源码具体实现，发现前面三个构造器都是调用的第四个构造器进行的初始化工作。

下面解释下一下构造器中各个参数的含义：

+ corePoolSize：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；
+ maximumPoolSize：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
+ keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；
+ unit：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：
```
TimeUnit.DAYS;               //天  
TimeUnit.HOURS;             //小时  
TimeUnit.MINUTES;           //分钟  
TimeUnit.SECONDS;           //秒  
TimeUnit.MILLISECONDS;      //毫秒  
TimeUnit.MICROSECONDS;      //微妙  
TimeUnit.NANOSECONDS;       //纳秒  
```
+ workQueue：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：
```
ArrayBlockingQueue;  
LinkedBlockingQueue;  
SynchronousQueue;  
```
ArrayBlockingQueue和PriorityBlockingQueue使用较少，一般使用LinkedBlockingQueue和Synchronous。线程池的排队策略与BlockingQueue有关。
+ threadFactory：线程工厂，主要用来创建线程；
+ handler：表示当拒绝处理任务时的策略，有以下四种取值：
```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。   
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。   
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）  
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务   
```
## ThreadPoolExecutor主要成员变量
```
private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务  
private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小  
                                                              //、runState等）的改变都要使用这个锁  
private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集  
  
private volatile long  keepAliveTime;    //线程存货时间      
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间  
private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）  
private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数  
  
private volatile int   poolSize;       //线程池中当前的线程数  
  
private volatile RejectedExecutionHandler handler; //任务拒绝策略  
  
private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程  
  
private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数  
  
private long completedTaskCount;   //用来记录已经执行完毕的任务个数  
```
## 线程池状态
在ThreadPoolExecutor中用ctl变量表示线程状态和线程数量：
```
/**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池状态可能的几个取值。

当创建线程池后，初始时，线程池处于RUNNING状态；

如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；

如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；

当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

### 线程池状态转移图
![image.png](/JAVA并发编程：线程池Executors（JDK1.7）/8596800-19eb59bbaa629cfb.png)

## ThreadPoolExecutor实现原理
介绍完ThreadPoolExecutor的构函数、成员变量和线程状态，我们来介绍最核心的内容。上文已经介绍过当我们把任务用ExecutorService的submit提交后是调用的ThreadPoolExecutor的execute来执行的，我们来看一下execute的代码
```
public void execute(Runnable command) {  
       if (command == null)  
           throw new NullPointerException();  
       if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {  
           if (runState == RUNNING && workQueue.offer(command)) {  
               if (runState != RUNNING || poolSize == 0)  
                   ensureQueuedTaskHandled(command);  
           }  
           else if (!addIfUnderMaximumPoolSize(command))  
               reject(command); // is shutdown or saturated  
       }  
   }  
```
我们开分析一下上面的代码，如果提交的任务command为null，则抛出空异常，
```
if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command))  
```
由于是或条件运算符，所以先计算前半部分的值，如果线程池中当前线程数不小于核心池大小，那么就会直接进入下面的if语句块了。

如果线程池中当前线程数小于核心池大小，则接着执行后半部分，也就是执行

```
addIfUnderCorePoolSize(command)
```

如果执行完addIfUnderCorePoolSize这个方法返回false，则继续执行下面的if语句块，否则整个方法就直接执行完毕了。

如果执行完addIfUnderCorePoolSize这个方法返回false，然后接着判断：
```
if (runState == RUNNING && workQueue.offer(command))
```
如果当前线程池处于RUNNING状态，则将任务放入任务缓存队列；如果当前线程池不处于RUNNING状态或者任务放入缓存队列失败，则执行：

```
addIfUnderMaximumPoolSize(command)

```

如果执行addIfUnderMaximumPoolSize方法失败，则执行reject()方法进行任务拒绝处理。

回到前面：

```java
if (runState == RUNNING && workQueue.offer(command))
```

 这句的执行，如果说当前线程池处于RUNNING状态且将任务放入任务缓存队列成功，则继续进行判断：
```
if (runState != RUNNING || poolSize == 0)
```
 这句判断是为了防止在将此任务添加进任务缓存队列的同时其他线程突然调用shutdown或者shutdownNow方法关闭了线程池的一种应急措施。如果是这样就执行
```
ensureQueuedTaskHandled(command)
```

进行应急处理，从名字可以看出是保证 添加到任务缓存队列中的任务得到处理。

我们接着看2个关键方法的实现：addIfUnderCorePoolSize和addIfUnderMaximumPoolSize：

```java
addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < corePoolSize && runState == RUNNING)
            t = addThread(firstTask);        //创建线程去执行firstTask任务    
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

这个是addIfUnderCorePoolSize方法的具体实现，从名字可以看出它的意图就是当低于核心吃大小时执行的方法。下面看其具体实现，首先获取到锁，因为这地方涉及到线程池状态的变化，先通过if语句判断当前线程池中的线程数目是否小于核心池大小，有朋友也许会有疑问：前面在execute()方法中不是已经判断过了吗，只有线程池当前线程数目小于核心池大小才会执行addIfUnderCorePoolSize方法的，为何这地方还要继续判断？原因很简单，前面的判断过程中并没有加锁，因此可能在execute方法判断的时候poolSize小于corePoolSize，而判断完之后，在其他线程中又向线程池提交了任务，就可能导致poolSize不小于corePoolSize了，所以需要在这个地方继续判断。然后接着判断线程池的状态是否为RUNNING，原因也很简单，因为有可能在其他线程中调用了shutdown或者shutdownNow方法。然后就是执行
```
t = addThread(firstTask);
```
这个方法也非常关键，传进去的参数为提交的任务，返回值为Thread类型。然后接着在下面判断t是否为空，为空则表明创建线程失败（即poolSize>=corePoolSize或者runState不等于RUNNING），否则调用t.start()方法启动线程。

我们来看一下addThread方法的实现：

```java
 Thread addThread(Runnable firstTask) {
    Worker w = new Worker(firstTask);
    Thread t = threadFactory.newThread(w);  //创建一个线程，执行任务    
    if (t != null) {
        w.thread = t;            //将创建的线程的引用赋值为w的成员变量        
        workers.add(w);
        int nt = ++poolSize;     //当前线程数加1        
        if (nt > largestPoolSize)
            largestPoolSize = nt;
    }
    return t;
}
```
在addThread方法中，首先用提交的任务创建了一个Worker对象，然后调用线程工厂threadFactory创建了一个新的线程t，然后将线程t的引用赋值给了Worker对象的成员变量thread，接着通过workers.add(w)将Worker对象添加到工作集当中。
下面我们看一下Worker类的实现：

```java
private final class Worker implements Runnable {
    private final ReentrantLock runLock = new ReentrantLock();
    private Runnable firstTask;
    volatile long completedTasks;
    Thread thread;
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
    }
    boolean isActive() {
        return runLock.isLocked();
    }
    void interruptIfIdle() {
        final ReentrantLock runLock = this.runLock;
        if (runLock.tryLock()) {
            try {
	    if (thread != Thread.currentThread())
		thread.interrupt();
            } finally {
                runLock.unlock();
            }
        }
    }
    void interruptNow() {
        thread.interrupt();
    }

    private void runTask(Runnable task) {
        final ReentrantLock runLock = this.runLock;
        runLock.lock();
        try {
            if (runState < STOP &&
                Thread.interrupted() &&
                runState >= STOP)
            boolean ran = false;
            beforeExecute(thread, task);   //beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据
            //自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等            
            try {
                task.run();
                ran = true;
                afterExecute(task, null);
                ++completedTasks;
            } catch (RuntimeException ex) {
                if (!ran)
                    afterExecute(task, ex);
                throw ex;
            }
        } finally {
            runLock.unlock();
        }
    }

    public void run() {
        try {
            Runnable task = firstTask;
            firstTask = null;
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            workerDone(this);   //当任务队列中没有任务时，进行清理工作        
        }
    }
}
```

既然Worker实现了Runnable接口，那么自然最核心的方法便是run()方法了：

```java
public void run() {
    try {
        Runnable task = firstTask;
        firstTask = null;
        while (task != null || (task = getTask()) != null) {
            runTask(task);
            task = null;
        }
    } finally {
        workerDone(this);
    }
}
```

这里意思是如果传递的task不为空，则立即执行，注意这里执行完了，并不是该线程就结束了，假如一开始的时候我们corePoolSize的大小为5，那么如果提交了5个任务，那么每个任务执行 runTask(task)后并没有结束，而是每个任务都对调用task = getTask()，去队列中取任务，这也是ThreadPoolExecutor提升效率的关键地方
。

getTask是ThreadPoolExecutor类中的方法，并不是Worker类中的方法，下面是getTask方法的实现：

```
Runnable getTask() {
    for (;;) {
        try {
            int state = runState;
            if (state > SHUTDOWN)
                return null;
            Runnable r;
            if (state == SHUTDOWN)  // Help drain queue
                r = workQueue.poll();
            else if (poolSize > corePoolSize || allowCoreThreadTimeOut) //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
            	//则通过poll取任务，若等待一定的时间取不到任务，则返回null
                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
            else
                r = workQueue.take();
            if (r != null)
                return r;
            if (workerCanExit()) {    //如果没取到任务，即r为null，则判断当前的worker是否可以退出
                if (runState >= SHUTDOWN) // Wake up others
                    interruptIdleWorkers();   //中断处于空闲状态的worker
                return null;
            }
            // Else retry
        } catch (InterruptedException ie) {
            // On interruption, re-check runState
        }
    }
}
```
在getTask中，先判断当前线程池状态，如果runState大于SHUTDOWN（即为STOP或者TERMINATED），则直接返回null。

如果runState为SHUTDOWN或者RUNNING，则从任务缓存队列取任务。

如果当前线程池的线程数大于核心池大小corePoolSize或者允许为核心池中的线程设置空闲存活时间，则调用poll(time,timeUnit)来取任务，这个方法会等待一定的时间，如果取不到任务就返回null。

然后判断取到的任务r是否为null，为null则通过调用workerCanExit()方法来判断当前worker是否可以退出，我们看一下workerCanExit()的实现：

```
private boolean workerCanExit() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    boolean canExit;
    //如果runState大于等于STOP，或者任务缓存队列为空了
    //或者  允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
    try {
        canExit = runState >= STOP ||
            workQueue.isEmpty() ||
            (allowCoreThreadTimeOut &&
             poolSize > Math.max(1, corePoolSize));
    } finally {
        mainLock.unlock();
    }
    return canExit;
}
```
我们再看addIfUnderMaximumPoolSize方法的实现，这个方法的实现思想和addIfUnderCorePoolSize方法的实现思想非常相似，唯一的区别在于addIfUnderMaximumPoolSize方法是在线程池中的线程数达到了核心池大小并且往任务队列中添加任务失败的情况下执行的：
```
private boolean addIfUnderMaximumPoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < maximumPoolSize && runState == RUNNING)
            t = addThread(firstTask);
    } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

看到没有，其实它和addIfUnderCorePoolSize方法的实现基本一模一样，只是if语句判断条件中的poolSize < maximumPoolSize不同而已。

到这里，大部分朋友应该对任务提交给线程池之后到被执行的整个过程有了一个基本的了解，下面总结一下：

1）首先，要清楚corePoolSize和maximumPoolSize的含义；

2）其次，要知道Worker是用来起到什么作用的；

3）要知道任务提交给线程池之后的处理策略，这里总结一下主要有4点：

*   如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；

*   如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务（当corePoolSize小于

    maximumPoolSize时候才会创建新的线程

    ）；

*   如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；

*   如果线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

我们在举个例子说明下，new ThreadPoolExecutor(5, 10, 0, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(10));加入我们创建了一个这样的线程池，请问，该线程池最小可以容纳多少线程？注意最小，我们可以假设线程在线程池中驻留的时间比较长。该例子中

corePoolSize = 5；maximumPoolSize = 10；阻塞队列的大小为10。假设我们提交的线程数为x。
1、当x<=5时候，addIfUnderCorePoolSize函数将任务提交并执行，执行完任务后不结束（没有设置keepAliveTime），会一直从队列中取任务
2、当x>5<=15时候，任务被提交的到队列，等待执行，此时线程池中容纳的线程为5+10=15；
3、当x>15<=20时候，因为在提交任务，队列已经满了提交不上，当此时的corePoolSize=5<maximumPoolSize=10,所以此时我们还可以提交5个任务，那么总的提交的任务数为20个

4,、当x>20时，会采取拒绝策略。
一般化一下：corePoolSize=a,maximumPoolSize=b,队列大小为c,那么线程池的总容量可以最小为a+c+(b-a)=b+c
补充：

##  

任务缓存队列及排队策略
在前面我们多次提到了任务缓存队列，即workQueue，它用来存放等待执行的任务。

　　workQueue的类型为BlockingQueue<Runnable>，通常可以取下面三种类型：

　　1）ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；

　　2）LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；

　　3）synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
##  

任务拒绝策略

　workQueue的类型为BlockingQueue<Runnable>，通常可以取下面四种类型：

```java
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 
```

##  

线程池的关闭

　　ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是shutdown()和shutdownNow()，其中：

*   shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务

*   shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

##  

线程池容量的动态调整

　　ThreadPoolExecutor提供了动态调整线程池容量大小的方法：setCorePoolSize()和setMaximumPoolSize()，

*   setCorePoolSize：设置核心池大小

*   setMaximumPoolSize：设置线程池最大能创建的线程数目大小

　　当上述参数从小变大时，ThreadPoolExecutor进行线程赋值，还可能立即创建新的线程来执行任务。

例子：

```java
package com.sunny.thread.threadpool;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Test {
	 public static void main(String[] args) {    
		 ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS, 
				 new ArrayBlockingQueue<Runnable>(5));
		 Test t = new Test();
		 for(int i=0;i<15;i++){
			 MyTask myTask = t.new MyTask(i);
			 executor.execute(myTask);
			 System.out.println("线程池中线程数目："+executor.getPoolSize()+"，队列中等待执行的任务数目："+
			 executor.getQueue().size()+"，已执行玩别的任务数目："+executor.getCompletedTaskCount());
		 }
		 executor.shutdown();
	 } 
	 class MyTask implements Runnable {
			private int taskNum;

			public MyTask(int num) {
				this.taskNum = num;
			}

			@Override
			public void run() {
				System.out.println("正在执行task "+taskNum);
				try {
					Thread.currentThread().sleep(4000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println("task "+taskNum+"执行完毕");
			}
		}
}
```

结果：

正在执行task 0
线程池中线程数目：1，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
线程池中线程数目：2，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 1
线程池中线程数目：3，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 2
线程池中线程数目：4，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 3
线程池中线程数目：5，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：1，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：2，已执行玩别的任务数目：0
正在执行task 4
线程池中线程数目：5，队列中等待执行的任务数目：3，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：4，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
线程池中线程数目：6，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 10
线程池中线程数目：7，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 11
线程池中线程数目：8，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 12
线程池中线程数目：9，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
线程池中线程数目：10，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 13
正在执行task 14
task 0执行完毕
正在执行task 5
task 1执行完毕
正在执行task 6
task 3执行完毕
正在执行task 7
task 2执行完毕
正在执行task 8
task 10执行完毕
正在执行task 9
task 11执行完毕
task 12执行完毕
task 14执行完毕
task 4执行完毕
task 13执行完毕
task 6执行完毕
task 5执行完毕
task 7执行完毕
task 8执行完毕
task 9执行完毕

## 

参考文章：


[http://www.cnblogs.com/dolphin0520/p/3932921.html](http://www.cnblogs.com/dolphin0520/p/3932921.html)


[http://ifeve.com/java-threadpool](http://ifeve.com/java-threadpool/)

[http://blog.163.com/among_1985/blog/static/275005232012618849266/](http://blog.163.com/among_1985/blog/static/275005232012618849266/)

[http://developer.51cto.com/art/201203/321885.htm](http://developer.51cto.com/art/201203/321885.htm)

[http://blog.csdn.net/java2000_wl/article/details/22097059](http://blog.csdn.net/java2000_wl/article/details/22097059)

[http://blog.csdn.net/cutesource/article/details/6061229](http://blog.csdn.net/cutesource/article/details/6061229)

[http://blog.csdn.net/xieyuooo/article/details/8718741](http://blog.csdn.net/xieyuooo/article/details/8718741)


