---
layout: post
title: "CompleableFuture原理和源码分析"
date: 2022-10-5
categories: 
    - "java"
tags: [java, 并发]
author: "pan"
---

## CompleableFuture 使用场景
CompletableFuture的定义如下：
```
public class CompletableFuture<T> implements Future<T>, CompletionStage<T>
```
我们看到CompletableFuture是实现了Future的接口的，在没有CompletableFuture之前，我们可以用FutureTask来实现一个Future的功能。那么有了FutureTask那么为什么还要有CompletableFuture呢？
我任务主要是CompletableFuture有两个优点
+ 1. CompletableFuture可以实现完全的异步，而FutureTask必须通过get阻塞的方式获取结果
```
        CompletableFuture
                .supplyAsync(()-> 1+2)
                .thenAccept((v)-> System.out.println(v*v));
```
如上面的代码所示，我们完整的任务有两个阶段，一阶段是计算1+2，二阶段是计算一阶段返回结果的平方，在整个过程中，主线程完全不需要管这个任务的执行情况，也不会阻塞主线程。但是如果用FutureTask实现如上功能如下：
```
        FutureTask<Integer> futureTask1 = new FutureTask<Integer>(() -> {
            return 1 + 2;
        });
        new Thread(futureTask1).start();
        Integer periodOneResult = futureTask1.get();

        FutureTask<Integer> futureTask2 = new FutureTask<Integer>(() -> {
            return periodOneResult * periodOneResult;
        });
        new Thread(futureTask2).start();
        Integer secondOneResult = futureTask2.get();
        System.out.println(secondOneResult);
```
代码冗长不说，还需要get方法阻塞主线程去获取结果。**以上代码只是说明CompletableFuture的异步优点，实际工作中你可以把两个任务看出两个api**
+ 2. CompletableFuture可以实现复杂的任务编排，请思考下面代码的执行顺序是什么？
```
CompletableFuture<String> base = new CompletableFuture<>();
        CompletableFuture<String> completion0 = base.thenApply(s -> {
            System.out.println("completion 0");
            return s + " 0";
        });
        CompletableFuture<String> completion1 = base.thenApply(s -> {
            System.out.println("completion 1");
            return s + " 1";
        });
        CompletableFuture<String> completion2 = base.thenApply(s -> {
            System.out.println("completion 2");
            return s + " 2";
        });
        completion1.thenApply(s -> {
            System.out.println("completion 3");
            return s + " 3";
        });
        completion1.thenApply(s -> {
            System.out.println("completion 4");
            return s + " 4";
        });
        completion1.thenApply(s -> {
            System.out.println("completion 5");
            return s + " 5";
        });


        completion2.thenApply(s -> {
            System.out.println("completion 6");
            return s + " 6";
        });
        completion2.thenApply(s -> {
            System.out.println("completion 7");
            return s + " 7";
        });
        completion2.thenApply(s -> {
            System.out.println("completion 8");
            return s + " 8";
        });

        base.complete("start");
```
如果你已经知道上面代码的执行顺序，那么你就跳过本篇文章吧，因为下文的重点就是说明CompletableFuture是如何做到任务的编排的。读者也可以参考文末参考文献中的两篇文章，我一开始也是参考的这两篇文章并结合源码才明白CompletableFuture的原理。总的来说CompletableFuture的实现原理非常复杂。总结一下，CompletableFuture的两个优点
+ **异步**
+ **复杂任务的编排**
## CompletableFuture 源码分析
```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
   volatile Object result;       // Either the result or boxed AltResult
   volatile Completion stack;    // Top of Treiber stack of dependent actions
}
```
+ result属性用于存放该CompletableFuture的计算结果，只用result不等于null时候，我们才可以通get()方法获取返回值，如果result是null，那么get()会一直阻塞。注意对于计算结果为null或者为异常的时候，result是AltResult类型的（封装null或者异常信息）
+ stack是CAS实现的无锁并发栈，还记得我们上面提出的问题吗？多个任务通过thenApply方法可以进行任意的编排，那编排的重点就是通过这个stack来实现的，我们下文会详细的分析。
### 任务编排
我们之前有提到CompletableFuture是通过属性stack那实现一个无锁并发栈来实现任务编排的。那么文章开头部门给出的代码，它是如何编排的

下图是代码实际的关系，我们重点需要关注两个类和三个关系
![c0-c9代表逻辑上创建顺序](/CompleableFuture原理和源码分析/8596800-f2b70becb743ea59.png)

**两个类***
+ 1. CompletableFuture，通过它，我们可以get任务的结果
+ 2. Completion，它是一个超级父类，实际源码中是用的很多子类。注意Completion中嵌套了CompletableFuture类。Completion的作用就是用来表示任务之前的关系。
**三个关系**
1.  next: 如上图中蓝线，如代码中base对象thenApply了completion1、completion2、completion3。那么这个三个对象的关系如图中的c0、c1、c2的关系，因为是栈嘛，c0先创建，但是在栈尾（first in last out）
2. dep（stack）: 该Completion中依赖的CompletableFuture。如图中青线。注意stack是来自于不同的CompletableFuture实例。
3. src：如图中红线。表示该Completion是来自于那一个CompletableFuture。如代码中base对象thenApply了completion1、completion2、completion3。那么c0、c1、c2的src都是base
**强调：dep中的stack不为null，则表示存在依赖（通thenApply创建），比如base-->thenApply-->c2, c2--thenApply-->c8; next不为空，则表示存在多个completion依赖于一个CompleteFuture，比如c0、c1、c2依赖于base。c6、c7、c8依赖于c2。所以是stack和next共同构建了我们之前提到的无锁并发栈。**其实源码的注释也说的很明白了
```
volatile Completion stack;    // Top of Treiber stack of dependent actions
```
**stack表示无锁并发栈的顶部**
```
volatile Completion next;      // Treiber stack link
```
**next链接无锁并发栈的completion**
有了上面的图示，大家再去理解源码会容易很多。
分析源码之前先看一下类的关系
![image.png](/CompleableFuture原理和源码分析/8596800-3a1f9a55ca307ae6.png)
### 源码分析
ComplableFuture有多种任务提交的方式，我们以supplyAsync为例。
####任务提交
```
// 改方法为静态方法
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
//asyncPool是默认ForkJoinPool线程池，supplier具体提交的任务
        return asyncSupplyStage(asyncPool, supplier);
    }
```
```
    static <U> CompletableFuture<U> asyncSupplyStage(Executor e,
                                                     Supplier<U> f) {
        if (f == null) throw new NullPointerException();
        CompletableFuture<U> d = new CompletableFuture<U>();
        e.execute(new AsyncSupply<U>(d, f));// 用指定的线程池执行我们的任务
        return d; 
    }
```
上面代码通过new AsyncSupply对象来提交一个任务，所以AsyncSupply可以是实现了Runnable接口的。线程池通过调用run来执行任务
```
 @SuppressWarnings("serial")
    static final class AsyncSupply<T> extends ForkJoinTask<Void>
            implements Runnable, AsynchronousCompletionTask {
        CompletableFuture<T> dep; Supplier<T> fn;
        AsyncSupply(CompletableFuture<T> dep, Supplier<T> fn) {
            this.dep = dep; this.fn = fn;
        }
        public void run() {
            CompletableFuture<T> d; Supplier<T> f;
            if ((d = dep) != null && (f = fn) != null) {
                dep = null; fn = null;
                if (d.result == null) {
                    try {
                        d.completeValue(f.get()); // f.get执行任务，并将任务的返回值提交给CompletableFuture，CompletableFuture就可以通过给来后驱f任务的结果
                    } catch (Throwable ex) {
                        d.completeThrowable(ex);// 注意异常是如何包装的
                    }
                }
                d.postComplete();//精华部分，用来触发d后面依赖的任务执行
            }
        }
    }
```
```
inal void postComplete() {
        /*
         * On each step, variable f holds current dependents to pop
         * and run.  It is extended along only one path at a time,
         * pushing others to avoid unbounded recursion.
         */
// this 是当前的CompletableFuture, f是依赖的（按照一定的顺序一直在移动，重点是移动的路径）
        CompletableFuture<?> f = this; Completion h;
        while ((h = f.stack) != null ||  // 取栈顶，不为空，则表示f存在依赖
               (f != this && (h = (f = this).stack) != null)) {
            CompletableFuture<?> d; Completion t;
            if (f.casStack(h, t = h.next)) { // 将f的stack属性指向h.next。如上图的base.stack->c1
                if (t != null) { //栈没有到底
                    if (f != this) {// 思考什么时候f！=this了
                        pushStack(h); // 只要栈没有到底，就一直将h压到栈顶
                        continue;
                    }
                    h.next = null;    // detach
                }
// tryFire也是关键的代码，如果h的存在依赖（dep.stack 不为空，则是存在依赖），则返回依赖，否则，返回null。既存在依赖f=依赖，不存在则f=this
                f = (d = h.tryFire(NESTED)) == null ? this : d;
            }
        }
    }
```
我们再回到上面的图，对着图梳理一遍逻辑。
![c0-c9代表逻辑上创建顺序](/CompleableFuture原理和源码分析/8596800-f2b70becb743ea59.png)
(1) h = f.stack，则当前h = c2
(2) f.casStack(h, t = h.next), 既base.stack -> c1, t == c1
(3) 此时f == this，所以执行f = (d = h.tryFire(NESTED)) == null ? this : d;那么此时c2中任务执行，c2 存在依赖，既c2中dep.stack!=null; 那么此时f==c2.dep。**注意此时f!=this了**
(4)再次循环，由于c8不是栈低，入栈，同样c7入栈，直到c6是栈低，那么c6执行，且c6不存在依赖，那么此时f==this。
上述步骤示意图如下（为了简化，我们去掉src线路，既红线）
![image.png](/CompleableFuture原理和源码分析/8596800-5ba7e7709821d8df.png)
此时，c2执行，c6执行，链接顺序变成base-->c7-->c8-->c1。此时f==this。c7不存在依赖，出栈执行，c8不存在依赖，出栈执行。到c1存在依赖，与前面思路类似。所以最终的线路图如下，执行循序也以在图中标明。既执行的循序是base->c2->c6->c7->c8->c1->c3->c4->c5->c0
![image.png](/CompleableFuture原理和源码分析/8596800-40d5f1c0902aebfa.png)。
下面我们接着看代码tryFire
```
 final CompletableFuture<V> tryFire(int mode) {
            CompletableFuture<V> d; CompletableFuture<T> a;
            if ((d = dep) == null ||
                !d.uniApply(a = src, fn, mode > 0 ? null : this)) //执行依赖，complete CompletableFuture d
                return null; //
            dep = null; src = null; fn = null;
            return d.postFire(a, mode); // d执行成功，检测d后面有没有任务可以接着执行
        }
```
```
 final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {
        if (a != null && a.stack != null) {
            if (mode < 0 || a.result == null)
                a.cleanStack();
            else
                a.postComplete();
        }
        if (result != null && stack != null) { //如果d执行完了，并且d.stack!=null(存在依赖)
            if (mode < 0)
                return this;
            else
                postComplete();
        }
        return null;
    }
```
以上就是代码的部分了，其实是非常复杂的，如果要想完全理解，就得去看无锁并发栈的理论设计了。
最后我们,我们再抛出一个小问题，思考下面代码的执行顺序，注意与文章开头部分的不同
```
 CompletableFuture<String> base = new CompletableFuture<>();
        CompletableFuture<String> completion0 = base.thenApply(s -> {
            System.out.println("completion 0");
            return s + " 0";
        });
        CompletableFuture<String> completion1 = base.thenApply(s -> {
            System.out.println("completion 1");
            return s + " 1";
        });
        CompletableFuture<String> completion2 = base.thenApply(s -> {
            System.out.println("completion 2");
            return s + " 2";
        });
        completion1.thenApply(s -> {
            System.out.println("completion 3");
            return s + " 3";
        }).thenApply(s -> {
            System.out.println("completion 4");
            return s + " 4";
        }).thenApply(s -> {
            System.out.println("completion 5");
            return s + " 5";
        });

        completion2.thenApply(s -> {
            System.out.println("completion 6");
            return s + " 6";
        }).thenApply(s -> {
            System.out.println("completion 7");
            return s + " 7";
        }).thenApply(s -> {
            System.out.println("completion 8");
            return s + " 8";
        });
        base.complete("start");
```
任务编排示意图如下，执行顺序，大家自行验证
![注意与上面的示意图的不同](/CompleableFuture原理和源码分析/8596800-d569f4c184e8121f.png)

# 参考文献
1. [从CompletableFuture到异步编程设计](https://www.cnblogs.com/xiangnanl/p/9939447.html)
2. [深入理解JDK8新特性CompletableFuture](https://developer.aliyun.com/article/712258)