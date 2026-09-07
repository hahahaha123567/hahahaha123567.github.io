---
title: java.util.concurrent
date: 2018-08-13 20:47:52
tags:
- Java
- 并发
description: Java 的并行除了直接对线程进行操作，还有 java.tuil.concurrent 包中的这些高级并发特性
---

# Concurrent 包

## 1. Lock

Lock 对象的工作机制类似于同步代码块使用的 synchronized，在某一时刻只允许一个线程拥有锁对象

通过它们关联的 Condition 对象，锁对象也支持 wait/notify 机制

一个 Lock 对象和一个 synchronized 代码块之间的主要不同点是：

- synchronized 代码块不能够保证进入访问等待的线程的先后顺序
- 你不能够传递任何参数给一个 synchronized 代码块的入口。因此，对于 synchronized 代码块的访问等待设置超时时间是不可能的事情
- synchronized 块必须被完整地包含在单个方法里。而一个 Lock 对象可以把它的 lock() 和 unlock() 方法的调用放在不同的方法里

Lock 加锁的方式很多：

- lock()
- lockInterruptibly()
- tryLock()
- tryLock(long timeout, TimeUnit timeUnit)

实现：`ReentrantLock` 

### 1.1 ReadWriteLock

实现：`ReentrantReadWriteLock` 

- **读锁**：如果没有任何写操作线程锁定 ReadWriteLock，并且没有任何写操作线程要求一个写锁(但还没有获得该锁)。因此，可以有多个读操作线程对该锁进行锁定
- **写锁**：如果没有任何读操作或者写操作。因此，在写操作的时候，只能有一个线程对该锁进行锁定

## 2. Future

Future 的核心思想是：一个方法 f，计算过程可能非常耗时，等待f返回，显然不明智

可以在调用 f 的时候，立马返回一个 Future，可以通过 Future 这个数据结构去控制方法f的计算过程。

这里的**控制**包括：

1. get：获取计算结果（如果还没计算完，也是必须等待的）
2. cancel：还没计算完，可以取消计算过程
3. isDone：判断是否计算完
4. isCancelled：判断计算是否被取消

```java
Callable<String> callable = () -> {
    Thread.sleep(2000);
    return "niconiconi";
};
FutureTask<String> task = new FutureTask<>(callable);
new Thread(task).start();
String say = null;
try {
    say = task.get();
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
System.out.println(say);
```

源码分析见 [彻底理解Java的Future模式 - 大诚挚 - 博客园](https://www.cnblogs.com/cz123/p/7693064.html)

## 3. Executor

Java.util.concurrent 包定义了三种 executor 接口： 

- Executor，支持加载新任务的简单接口
- ExecutorService，Executor 的子接口，增加了帮助管理任务和executor本身的生命周期的功能
- ScheduledExecutorService，ExecutorService 的子接口，支持周期性地执行任务

### 3.1 Executor

Executor 接口只定义了一个方法 execute，被设计用来代替一般的创建线程惯例 

```java
// r implements Runnable
// 自己控制线程
(new Thread(r)).start();
// 用 executor
e.execute(r);
```

对于 execute 方法的实现并没有特殊要求。低级的实现只是创建一个新的线程并立即执行

但更有可能是使用一个已经存在的工作线程（worker Threads）去执行 r，或者将 r 放在一个执行队列中等待工作线程有空的时候再执行 

### 3.2 ExecutorService

ExecutorService 接口提供了另一个相似的 submit 方法，但比 execute 更加通用

和 execute 一样，submit 接受 Runnable 对象，但也接受 Callable 对象，Callable 允许任务执行后返回一个值

Submit 方法返回 Future 对象，Future 对象被用来接收 Callable 返回的值，并管理 Callable 和 Runnable 对象所代表的任务 

常用实现：ThreadPoolExecutor

- Executors.newFixedThreadPool() : ThreadPoolExecutor
- Executors.newCachedThreadPool() : ThreadPoolExecutor 一个使用可扩展线程池的 executor，拥有这种线程池的 executor 适合执行生命周期较短的任务
- Executors.newSingleThreadExecutor() : FinalizableDelegatedExecutorService 一个只有一个工作线程的executor

### 3.3 ScheduledExecutorService

ScheduledExecutorService 接口为它的父类 ExecutorService 的行为提供计划，允许在执行 Runnable 和 Callable 任务之前停顿一段时间

接口定义了 scheduleAtFixedRate 和 scheduleWithFixedDelay，这两个方法以特定的时间间隔重复地执行特定任务 

常用实现：ScheduledThreadPoolExecutor

- Executors.newScheduledThreadPool() : ScheduledThreadPoolExecutor

### 3.4 thread pool

Thread 对象占用了大量的内存，在一个大型应用中，分配和释放线程对象会造成大量的线程管理开支 

使用工作线程可以减少创建线程的资源浪费，工作线程是不属于特定某个 Runnable 和 Callable 任务，经常用来执行多个任务 

大部分 java.util.concurrent 中的 executor 实现类使用了工作线程(worker threads)的线程池

### 3.5 the fixed thread pool

Executors.newFixedThreadPool()

##### what

一种常用的线程池是固定线程池，这种线程池有特定数量的线程在运行，如果一个线程在使用过程中意外停止(如抛出未捕获的异常)，它会自动被另一个新线程替代

##### how

任务通过一个内部的任务队列提交给线程池执行，此任务队列可以在活动任务数量大于线程池中工作线程个数时，存储多余的活动任务 

##### why

“固定大小线程池”的一个好处是”优雅的缓冲”

考虑一个web应用服务，它的每一个线程只处理一个HTTP请求。如果这个应用简单地为每一个新来的请求创建一个新的处理线程，那么，当请求数量足够多时，线程占用的资源总和将超过系统的承受能力，服务器会因此忽然停止对所有请求的应答(常见的内存溢出)

而在使用固定大小线程池后，即使请求数量超出工作线程能够处理的请求上限，但是新来的HTTP请求会被暂时存放在消息队列中，当出现空闲的工作线程后，这些HTTP请求就会得到及时的处理 

## 4. Fork/Join

Java SE 7 中的特性，fork/join 框架帮助你创建多进程应用。它被设计用来完成可以分成很多小进程的工作，目的是使用所有可用的进程来提升你的应用的性能 

像任何 ExecutorService 一样，fork/join 框架把任务分发给线程池中工作线程。不同的是，因为 fork/join 框架使用 work-stealing 算法——完成工作的工作线程可以从其他还在忙碌的线程那里偷任务来执行 

fork/join 框架的核心是 ForkJoinPool 类，一个 AbstractExecutorService 的扩展类；ForkJoinPool 实现了核心的work-stealing算法，能够执行 ForkJoinTask 任务 

### 4.1 思路

```java
if (此部分工作足够小) {
  直接干活
}
else {
  把工作分成两部分，  
  调用完成这两部分的代码，并等待结果返回
}
```

把以上代码封装成ForkJoinTask子类

特别地作为更具体的类型 RecursiveAction 或 RecursiveTask (可以返回结果)

### 4.2 例子

```java
// RecursiveAction, execute without return value
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class MyRecursiveAction extends RecursiveAction {

    private long workLoad = 0;

    public MyRecursiveAction(long workLoad) { this.workLoad = workLoad; }

    @Override
    protected void compute() {

        if(this.workLoad > 16) {
            System.out.println("Splitting workLoad : " + this.workLoad);

            List<MyRecursiveAction> subtasks = new ArrayList<>(createSubtasks());
            for(RecursiveAction subtask : subtasks){
                subtask.fork();
            }
        } else {
            System.out.println("Doing workLoad myself: " + this.workLoad);
        }
    }

    private List<MyRecursiveAction> createSubtasks() {
        List<MyRecursiveAction> subtasks = new ArrayList<>();

        MyRecursiveAction subtask1 = new MyRecursiveAction(this.workLoad / 2);
        MyRecursiveAction subtask2 = new MyRecursiveAction(this.workLoad / 2);

        subtasks.add(subtask1);
        subtasks.add(subtask2);

        return subtasks;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool(4);
        MyRecursiveAction myRecursiveAction = new MyRecursiveAction(24);

        forkJoinPool.invoke(myRecursiveAction);
    }
}
```

```java
// RecursiveTask, execute with return value
public class MyRecursiveTask extends RecursiveTask<Long> {

    private long workLoad = 0;

    public MyRecursiveTask(long workLoad) {
        this.workLoad = workLoad;
    }

    @Override
    protected Long compute() {
        if(this.workLoad > 16) {
            System.out.println("Splitting workLoad : " + this.workLoad);

            List<MyRecursiveTask> subtasks = new ArrayList<>(createSubtasks());

            for(MyRecursiveTask subtask : subtasks){
                subtask.fork();
            }

            long result = 0;
            for(MyRecursiveTask subtask : subtasks) {
                result += subtask.join();
            }
            return result;
        } else {
            System.out.println("Doing workLoad myself: " + this.workLoad);
            return workLoad * 3;
        }
    }

    private List<MyRecursiveTask> createSubtasks() {
        List<MyRecursiveTask> subtasks = new ArrayList<>();

        MyRecursiveTask subtask1 = new MyRecursiveTask(this.workLoad / 2);
        MyRecursiveTask subtask2 = new MyRecursiveTask(this.workLoad / 2);

        subtasks.add(subtask1);
        subtasks.add(subtask2);

        return subtasks;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool(4);
        MyRecursiveTask myRecursiveTask = new MyRecursiveTask(128);

        long mergedResult = forkJoinPool.invoke(myRecursiveTask);

        System.out.println("mergedResult = " + mergedResult);
    }
}
```

### 4.3 work-stealing

调度策略：

- 每一个工作线程维护自己的调度队列中的可运行任务 
- 队列以 deque 的形式被维护，不仅支持后进先出 —— `LIFO` 的 `push` 和 `pop` 操作，还支持先进先出 —— `FIFO` 的 `take` 操作 
- 对于一个给定的工作线程来说，任务所产生的子任务将会被放入到工作者自己的双端队列中 
- 工作线程使用后进先出 —— `LIFO`（最新的元素优先）的顺序，通过弹出任务来处理队列中的任务 
- 当一个工作线程的本地没有任务去运行的时候，它将使用先进先出 —— `FIFO`的规则尝试随机的从别的工作线程中拿（『窃取』）一个任务去运行 
- 当一个工作线程触及了`join`操作，如果可能的话它将处理其他任务，直到目标任务被告知已经结束（通过 `isDone` 方法）。所有的任务都会无阻塞的完成 
- 当一个工作线程无法再从其他线程中获取任务和失败处理的时候，它就会退出（通过`yield`、`sleep`和/或者优先级调整）并经过一段时间之后再度尝试直到所有的工作线程都被告知他们都处于空闲的状态；在这种情况下，他们都会阻塞直到其他的任务再度被上层调用 

使用后进先出 —— `LIFO`用来处理每个工作线程的自己任务，但是使用先进先出 —— `FIFO`规则用于获取别的任务，这是一种被广泛使用的进行递归`Fork/Join`设计的一种调优手段 

让窃取任务的线程从队列拥有者相反的方向进行操作会减少线程竞争。同样体现了递归分治算法的大任务优先策略。因此，更早期被窃取的任务有可能会提供一个更大的单元任务，从而使得窃取线程能够在将来进行递归分解 

作为上述规则的一个后果，对于一些基础的操作而言，使用相对较小粒度的任务比那些仅仅使用粗粒度划分的任务以及那些没有使用递归分解的任务的运行速度要快。尽管相关的少数任务在大多数的`Fork/Join`框架中会被其他工作线程窃取，但是创建许多组织良好的任务意味着只要有一个工作线程处于可运行的状态，那么这个任务就有可能被执行 

## 5. Concurrent Collection

### 5.1 BlockingQueue

一个线程将会持续生产新对象并将其插入到队列之中，直到队列达到它所能容纳的临界点

也就是说，它是有限的，如果该阻塞队列到达了其临界点，负责生产的线程将会在往里边插入新对象时发生阻塞，直到负责消费的线程从队列中拿走一个对象。

负责消费的线程将会一直从该阻塞队列中拿出对象。如果消费线程尝试去从一个空的队列中提取对象的话，这个消费线程将会处于阻塞之中，直到一个生产线程把一个对象丢进队列

常用实现：

1. ArrayBlockingQueue 上限不能改变
2. LinkedBlockingQueue 上限能改变
3. DelayQueue 对元素进行持有，直到到期 
4. PriorityBlockingQueue 排序规则和 java.util.PriorityQueue 相同
5. SynchronousQueue 只能放一个元素

### 5.2 BlockingDeque

如果生产者线程需要在队列的两端都可以插入数据，消费者线程需要在队列的两端都可以移除数据，可以使用 BlockingDeque 

如果双端队列已满，插入线程将被阻塞，直到一个移除线程从该队列中移出了一个元素

如果双端队列为空，移除线程将被阻塞，直到一个插入线程向该队列插入了一个新元素 

### 5.3 ConcurrentMap 

见 [ConcurrentHashMap演进从Java7到Java8](http://www.jasongj.com/java/concurrenthashmap) 

## 6. CyclicBarrier

它就是一个所有线程必须等待的一个栅栏，直到所有线程都到达这里，然后所有线程才可以继续做其他事情 

## 7. Exchanger

表示一种两个线程可以进行互相交换对象的会和点 

## 8. Semaphore

计数信号量由一个指定数量的 "许可" 初始化

每调用一次 acquire()，一个许可会被调用线程取走。每调用一次 release()，一个许可会被返还给信号量

因此，在没有任何 release() 调用时，最多有 N 个线程能够通过 acquire() 方法，N 是该信号量初始化时的许可的指定数量。 

## 9. Atomic Variable 

java.util.concurrent.atomic 包中定义了对单个变量的原子操作支持

包中的所有的类的 getter 和 setter，都像对 volatile 变量操作一样，具有原子性，即一个set操作与后续的get操作存在绝对的先后关系 

### 9.1 AtomicReference 

AtomicReference 类具备了一个很有用的方法：compareAndSet()

compareAndSet() 可以将保存在 AtomicReference 里的引用于一个期望引用进行比较，如果两个引用是一样的，将会给 AtomicReference 实例设置一个新的引用 —— **并非 equals() 的相等，而是 == 的相等** 

## 10. ThreadLocalRandom 

在并发环境中，使用 ThreadLocalRandom 替换 Math.random() 可以减少冲突，提高性能

可以将它用来在多线程或ForJoinTask中获得随机数

---

参考文章

[java.util.concurrent (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html) 

[高级并发对象(Councurrency Tutorial 7) - 春晓春晓 - ITeye博客](http://623deyingxiong.iteye.com/blog/1754030) 

[彻底理解Java的Future模式 - 大诚挚 - 博客园](https://www.cnblogs.com/cz123/p/7693064.html) 

[JVM 并发性: Java 和 Scala 并发性基础](https://www.ibm.com/developerworks/cn/java/j-jvmc1/index.html) 

[JVM 并发性: Java 8 并发性基础](https://www.ibm.com/developerworks/cn/java/j-jvmc2/index.html) 

[Java Fork/Join框架 | 并发编程网 – ifeve.com](http://ifeve.com/java-fork-join-framework) 

[Java 并发工具包 java.util.concurrent 用户指南 - CSDN博客](http://www.importnew.com/26461.html) 

[ConcurrentHashMap演进从Java7到Java8 | 技术世界](http://www.jasongj.com/java/concurrenthashmap) 


