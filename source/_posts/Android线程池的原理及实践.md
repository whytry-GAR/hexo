---
title: Android线程池的原理及实践
date: 2020-03-18 21:01:27
tags: [Android]
---

<meta name="referrer" content="no-referrer"/>

# 提纲 

是什么（使用线程池的原因，线程池的定义，好处，线程池原理）

怎么用（常见的使用方式，以及各个参数的作用）

为什么（源码分析，设计模式分析）

关于原理在android中的部分应用，部分注意事项

## 引子（原因）

**多线程技术：**

多线程技术主要解决处理器单元内多个线程执行的问题，它可以显著减少处理器单元的闲置时间，增加处理器单元的吞吐能力

多线程的异步执行方式，虽然能够最大限度发挥多核计算机的计算能力，但是如果不加控制，反而会对系统造成负担。线程本身也要占用内存空间，大量的线程会占用内存资源并且可能会导致Out of Memory。即便没有这样的情况，大量的线程回收也会给GC带来很大的压力。

当创建线程时间+销毁线程时间远大于在线程中执行任务的时间时 或者 需要多次调用多线程异步任务时，我们可以考虑线程池

## **什么是线程池**

### **定义**

一种线程使用模式，通过池化技术维护着多个线程，对线程进行**统一管理和复用**，避免了在处理短时间任务时创建与销毁线程的代价。线程池不仅能够保证内核的充分利用，还能防止过分调度。

### 原理

池化技术，享元模式，对线程进行统一管理和复用

### 使用线程池的好处

减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。从而减少线程的创建和销毁，节约系统的开销

对线程具有统一的管理，可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存

### 简单的demo对比结果 



![img](https://upload-images.jianshu.io/upload_images/5714046-cff7028a3f419e2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 怎么使用线程池

使用线程池主要涉及两个类

ExecutorService：线程池接口，提供了众多接口api来控制线程池中的线程

ThreadPoolExecutor：实现了ExecutorService接口，并封装了一系列的api使得它具有线程池的特性，其中包括工作队列、核心线程数、最大线程数等，可以说这个就是线程池的代表类

要创建一个线程池只需要new ThreadPoolExecutor(…);就可以创建一个线程池，而如果这样创建线程池的话，我们需要配置一堆东西，非常麻烦，我们可以看一下它的构造方法就知道了：

public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler) {...}

所以，官方也不推荐使用这种方法来创建线程池，而是推荐使用**Executors**的工厂方法来创建线程池，**Executors**类是官方提供的一个工厂类，它里面封装好了众多功能不一样的线程池，从而使得我们创建线程池非常的简便，主要提供了如下五种功能不一样的线程池：

**1、newFixedThreadPool() ：**

作用：该方法返回一个固定线程数量的线程池，该线程池中的线程数量始终不变，即不会再创建新的线程，也不会销毁已经创建好的线程，自始自终都是那几个固定的线程在工作，所以该线程池可以控制线程的最大并发数。

栗子：假如有一个新任务提交时，线程池中如果有空闲的线程则立即使用空闲线程来处理任务，如果没有，则会把这个新任务存在一个任务队列中，一旦有线程空闲了，则按FIFO方式处理任务队列中的任务。

**2、newCachedThreadPool() ：**

作用：该方法返回一个可以根据实际情况调整线程池中线程的数量的线程池。即该线程池中的线程数量不确定，是根据实际情况动态调整的。

栗子：假如该线程池中的所有线程都正在工作，而此时有新任务提交，那么将会创建新的线程去处理该任务，而此时假如之前有一些线程完成了任务，现在又有新任务提交，那么将不会创建新线程去处理，而是复用空闲的线程去处理新任务。那么此时有人有疑问了，那这样来说该线程池的线程岂不是会越集越多？其实并不会，因为线程池中的线程都有一个“保持活动时间”的参数，通过配置它，如果线程池中的空闲线程的空闲时间超过该“保存活动时间”则立刻停止该线程，而该线程池默认的“保持活动时间”为60s。

**3、newSingleThreadExecutor() ：**

作用：该方法返回一个只有一个线程的线程池，即每次只能执行一个线程任务，多余的任务会保存到一个任务队列中，等待这一个线程空闲，当这个线程空闲了再按FIFO方式顺序执行任务队列中的任务。

**4、newScheduledThreadPool() ：**

作用：该方法返回一个可以控制线程池内线程定时或周期性执行某任务的线程池。

**5、newSingleThreadScheduledExecutor() ：**

作用：该方法返回一个可以控制线程池内线程定时或周期性执行某任务的线程池。只不过和上面的区别是该线程池大小为1，而上面的可以指定线程池的大小。

获取这五种线程池

通过**Executors**的工厂方法来获取：

```java
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(5);   

ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();   

ExecutorService cachedThreadPool = Executors.newCachedThreadPool();   

ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);  

ScheduledExecutorService singleThreadScheduledPool = Executors.newSingleThreadScheduledExecutor();
```

我们可以看到通过**Executors**的工厂方法来创建线程池极其简便，其实它的内部还是通过new ThreadPoolExecutor(…)的方式创建线程池的，我们看一下这些工厂方法的内部实现：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {     

return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue());   }  

 public static ExecutorService newSingleThreadExecutor() {  

 return new FinalizableDelegatedExecutorService （new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue()));   } 

public static ExecutorService newCachedThreadPool() {   

return new ThreadPoolExecutor(0, Integer.MAX_VALUE,                       60L, TimeUnit.SECONDS,new SynchronousQueue());   }
```

我们可以清楚的看到这些方法的内部实现都是通过创建一个ThreadPoolExecutor对象来创建的，正所谓万变不离其宗，所以我们要了解线程池还是得了解ThreadPoolExecutor这个线程池类

## 了解ThreadPoolExecutor

所以我们主要就是要了解ThreadPoolExecutor，从构造方法开始：

![img](https://upload-images.jianshu.io/upload_images/5714046-6c0f68f24a0ad9a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到它构造方法的参数比较多，有七个，下面一一来说明这些参数的作用：

**corePoolSize**：线程池中的核心线程数量

**maximumPoolSize**：线程池中的最大线程数量

**keepAliveTime**：这个就是上面说到的“保持活动时间“，上面只是大概说明了一下它的作用，不过它起作用必须在一个前提下，就是当线程池中的线程数量超过了corePoolSize时，它表示多余的空闲线程的存活时间，即：多余的空闲线程在超过keepAliveTime时间内没有任务的话则被销毁。而这个主要应用在缓存线程池中

**unit**：它是一个枚举类型，表示keepAliveTime的单位，常用的如：TimeUnit.SECONDS（秒）、TimeUnit.MILLISECONDS（毫秒）

**workQueue**：任务队列，主要用来存储已经提交但未被执行的任务，不同的线程池采用的排队策略不一样

**threadFactory**：线程工厂，用来创建线程池中的线程，通常用默认的即可

**handler**：通常叫做拒绝策略，1、在线程池已经关闭的情况下 2、任务太多导致最大线程数和任务队列已经饱和，无法再接收新的任务 。在上面两种情况下，只要满足其中一种时，在使用execute()来提交新的任务时将会拒绝，而默认的拒绝策略是抛一个RejectedExecutionException异常

workQueue这个任务队列却要再次说明一下，它是一个BlockingQueue<Runnable>对象，而泛型则限定它是用来存放Runnable对象的，刚刚上面讲了，不同的线程池它的任务队列实现肯定是不一样的，所以，保证不同线程池有着不同的功能的核心就是这个workQueue的实现了，细心的会发现在刚刚的用来创建线程池的工厂方法中，针对不同的线程池传入的workQueue也不一样，下面我总结一下这五种线程池分别用的是什么BlockingQueue：

1、newFixedThreadPool()—>LinkedBlockingQueue

2、newSingleThreadExecutor()—>LinkedBlockingQueue

3、newCachedThreadPool()—>SynchronousQueue

4、newScheduledThreadPool()—>DelayedWorkQueue

5、newSingleThreadScheduledExecutor()—>DelayedWorkQueue

这些队列分别表示：

**LinkedBlockingQueue**：无界的队列

**SynchronousQueue**：直接提交的队列

**DelayedWorkQueue**：等待队列

当然实现了BlockingQueue接口的队列还有：ArrayBlockingQueue（有界的队列）、PriorityBlockingQueue（优先级队列）。这些队列的详细作用就不多介绍了。

当 Executor 已经关闭，并且 Executor 将有限边界用于最大线程和工作队列容量，且已经饱和时，在方法execute(java.lang.Runnable) 中提交的新任务将被拒绝。在以上两种情况下，execute 方法都将调用其RejectedExecutionHandler 的 RejectedExecutionHandler.rejectedExecution(java.lang.Runnable, java.util.concurrent.ThreadPoolExecutor) 方法。下面提供了四种预定义的处理程序策略：

A. 在默认的 ThreadPoolExecutor.AbortPolicy 中，处理程序遭到拒绝将抛出运行时 RejectedExecutionException。

B. 在 ThreadPoolExecutor.CallerRunsPolicy 中，线程调用运行该任务的 execute 本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。

C. 在 ThreadPoolExecutor.DiscardPolicy 中，不能执行的任务将被删除。

D. 在 ThreadPoolExecutor.DiscardOldestPolicy 中，如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）。

**LinkedBlockingQueue**

1:如果未指定容量，默认容量为Integer.MAX_VALUE ，容量范围可以在构造方法参数中指定作为防止队列过度扩展。

2:此对象是 线程阻塞-安全的

3：不接受 null 元素

4:它实现了BlockingQueue接口。

5:实现了 Collection 和 Iterator 接口的所有可选 方法。

线程池ThreadPoolExecutor的使用

使用线程池，其中涉及到一个极其重要的方法，即：

execute(Runnable command)

该方法意为执行给定的任务，该任务处理可能在新的线程、已入池的线程或者正调用的线程，这由ThreadPoolExecutor的实现决定。

当一个任务通过execute(Runnable)方法欲添加到线程池时：

l 如果此时线程池中的数量小于corePoolSize，即使线程池中的线程都处于空闲状态，也要创建新的线程来处理被添加的任务。

l 如果此时线程池中的数量等于 corePoolSize，但是缓冲队列 workQueue未满，那么任务被放入缓冲队列。

l 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量小于maximumPoolSize，建新的线程来处理被添加的任务。

l 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量等于maximumPoolSize，那么通过 handler所指定的策略来处理此任务。也就是：处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。

l 当线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止。这样，线程池可以动态的调整池中的线程数。



![img](https://upload-images.jianshu.io/upload_images/5714046-95649e44c28f6660?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### **优先级线程池的优点**

优先级可以在线程实现了BlockingQueue接口的队列还有：ArrayBlockingQueue（有界的队列）、PriorityBlockingQueue（优先级队列）。这些队列的详细作用就不多介绍了。池中线程数量不足或系统资源紧张时，优先处理我们想要先处理的任务，而优先级低的则放到后面再处理，这极大改善了系统默认线程池以FIFO方式处理任务的不灵活

### 扩展线程池ThreadPoolExecutor

除了内置的功能外，ThreadPoolExecutor也向外提供了三个接口供我们自己扩展满足我们需求的线程池，这三个接口分别是：

beforeExecute() - 任务执行前执行的方法

afterExecute() -任务执行结束后执行的方法

terminated() -线程池关闭后执行的方法

这三个方法在ThreadPoolExecutor内部都没有实现

前面两个方法我们可以在ThreadPoolExecutor内部的runWorker()方法中找到，而runWorker()是ThreadPoolExecutor的内部类Worker实现的方法，Worker它实现了Runnable接口，也正是线程池内处理任务的工作线程，而Worker.runWorker()方法则是处理我们所提交的任务的方法，它会同时被多个线程访问，所以我们看runWorker()方法的实现，由于涉及到多个线程的异步调用，必然是需要使用锁来处理，而这里使用的是Lock来实现的，我们来看看runWorker()方法内主要实现：



![img](https://upload-images.jianshu.io/upload_images/5714046-55171e621df6155a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到在task.run()之前和之后分别调用了beforeExecute和afterExecute方法，并传入了我们的任务Runnable对象

而terminated()则是在关闭线程池的方法中调用，而关闭线程池有两个方法，我贴其中一个：



![img](https://upload-images.jianshu.io/upload_images/5714046-cd794180bcbd0cac?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### **优化线程池ThreadPoolExecutor**

虽说线程池极大改善了系统的性能，不过创建线程池也是需要资源的，所以线程池内线程数量的大小也会影响系统的性能，大了反而浪费资源，小了反而影响系统的吞吐量，所以我们创建线程池需要把握一个度才能合理的发挥它的优点，通常来说我们要考虑的因素有CPU的数量、内存的大小、并发请求的数量等因素，按需调整。

通常核心线程数可以设为CPU数量+1，而最大线程数可以设为CPU的数量*2+1。

获取CPU数量的方法为：

Runtime.getRuntime().availableProcessors();

java线程池大小为何会大多被设置成CPU核心数+1？https://blog.csdn.net/varyall/article/details/79583036

线程池大小设置，CPU的核心数、线程数的关系和区别，同步与堵塞完全是两码事https://blog.csdn.net/tbdp6411/article/details/78443732

**最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）\* CPU数目**

从任务的优先级，任务的执行时间长短，任务的性质（CPU密集/ IO密集），任务的依赖关系这四个角度来分析。并且近可能地使用有界的工作队列。

性质不同的任务可用使用不同规模的线程池分开处理：

CPU密集型：尽可能少的线程，Ncpu+1

IO密集型：尽可能多的线程, Ncpu*2，比如数据库连接池

混合型：CPU密集型的任务与IO密集型任务的执行时间差别较小，拆分为两个线程池；否则没有必要拆分。

### shutdown()和shutdownNow()的区别

关于线程池的停止，ExecutorService为我们提供了两个方法：shutdown和shutdownNow，这两个方法各有不同，可以根据实际需求方便的运用，如下：

1、shutdown()方法在终止前允许执行以前提交的任务。

 2、shutdownNow()方法则是阻止正在任务队列中等待任务的启动并试图停止当前正在执行的任务。

遍历线程池中的所有线程，然后逐个调用线程的interrupt方法来中断线程.

shutdown 将线程池里的线程状态设置成SHUTDOWN状态, 然后中断所有没有正在执行任务的线程. shutdownNow 将线程池里的线程状态设置成STOP状态, 然后停止所有正在执行或暂停任务的线程. 只要调用这两个关闭方法中的任意一个, isShutDown() 返回true. 当所有任务都成功关闭了, isTerminated()返回true.

## 线程池源码分析原理

之前说到线程池原理是对多个线程的管理复用，减少了线程的创建和销毁过程，也减少了创建线程的数目，从而提高了效率。也讲到各个参数的作用，接下来我们看看线程池的源码是怎么实现这些操作的。



![img](https://upload-images.jianshu.io/upload_images/5714046-0bbd8f425b3e992c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](https://upload-images.jianshu.io/upload_images/5714046-8d866b28fb6f17a2?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中AtomicInteger变量ctl的功能非常强大：利用低29位表示线程池中线程数，通过高3位表示线程池的运行状态：

1、RUNNING：-1 << COUNT_BITS，即高3位为111，该状态的线程池会接收新任务，并处理阻塞队列中的任务； 

2、SHUTDOWN： 0 << COUNT_BITS，即高3位为000，该状态的线程池不会接收新任务，但会处理阻塞队列中的任务； 

3、STOP ： 1 << COUNT_BITS，即高3位为001，该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务； 

4、TIDYING ： 2 << COUNT_BITS，即高3位为010, 所有的任务都已经终止

5、TERMINATED： 3 << COUNT_BITS，即高3位为011, terminated()方法已经执行完成

**1、先看一下线程池的executor方法（任务入口）**

![img](https://upload-images.jianshu.io/upload_images/5714046-01d1c9efc1c0515a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

判断当前活跃线程数是否小于corePoolSize,如果小于，则调用addWorker创建线程执行任务

如果不小于corePoolSize，则将任务添加到workQueue队列。

如果放入workQueue失败，则创建线程执行任务，如果这时创建线程失败(当前线程数不小于maximumPoolSize时)，就会调用reject(内部调用handler)拒绝接受任务。

为什么需要double check线程池的状态？

在多线程环境下，线程池的状态时刻在变化，而ctl.get()是非原子操作，很有可能刚获取了线程池状态后线程池状态就改变了。判断是否将command加入workque是线程池之前的状态。倘若没有double check，万一线程池处于非running状态（在多线程环境下很有可能发生），那么command永远不会执行。

**2、再看下addWorker的方法实现**



![img](https://upload-images.jianshu.io/upload_images/5714046-5f76cb7ef62d9bb8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![img](https://upload-images.jianshu.io/upload_images/5714046-679c3cc111ce441e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

•看到这里我们了解了线程池中运行状态和线程数量怎么维护，以及拒绝策略的执行判断。还有个wokers的集合来管理线程•但线程复用逻辑，主要得看worker类

**3、再到Worker里看看其实现**



![img](https://upload-images.jianshu.io/upload_images/5714046-50ff1f9fc6d88c1b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**4、接下来咱们看看runWorker方法的逻辑**



![img](https://upload-images.jianshu.io/upload_images/5714046-4ce081b9158a6162?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**5、最后在看看getTask方法实现**



![img](https://upload-images.jianshu.io/upload_images/5714046-46327e5fde759ae4?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到这里我们明白了线程是怎么达成复用的。以及为什么非核心线程会自动的被干掉。

参考：Java线程池实现原理与源码解析(jdk1.8)[https://blog.csdn.net/programmer_at/article/details/79799267#4-threadpoolexecutor%E6%BA%90%E7%A0%81](https://blog.csdn.net/programmer_at/article/details/79799267#4-threadpoolexecutor源码)

## 关于原理在android中的部分应用

### 关于AsyncTask的实现

大家都知道AsyncTask内部实现其实就是Thread+Handler。其中Handler是为了处理线程之间的通信，而这个Thread到底是指什么呢？通过AsyncTask源码可以得知，其实这个Thread是线程池，AsyncTask内部实现了两个线程池，分别是：串行线程池和固定线程数量的线程池。而这个固定线程数量则是通过CPU的数量决定的。



![img](https://upload-images.jianshu.io/upload_images/5714046-f8c14a051f35545e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在默认情况下，我们大都通过AsyncTask::execute()来执行任务的， ，而execute()内部则是调用executeOnExecutor(sDefaultExecutor, params)方法执行的，第一个参数就是指定处理该任务的线程池，而默认情况下AsyncTask是传入串行线程池（在这里不讲版本的变化），也就是任务只能单个的按顺序执行，而我们要是想让AsyncTask并行的处理任务，大家都知道调用AsyncTask::executeOnExecutor(sDefaultExecutor, params)方法传入这个参数即可：AsyncTask.THREAD_POOL_EXECUTOR。 而这个参数的意义在于为任务指定了一个固定线程数量的线程池去处理，从而达到了并行处理的功能，我们可以在源码中看到AsyncTask.THREAD_POOL_EXECUTOR这个参数就是一个固定线程数量的线程池：

### 关于message的实现

我们都知道message也有关于obtain的实现 ，通过obtain获取message对象来使用代替new一个也是google推荐的做法， 这里也有用到享元模式，通过复用已有的message对象来减少创建及回收。

message内部有个静态的消息池，采用的是链表的实现方式进行管理，最大数量为50，当调用obtain方法时，会先上锁，判断当前的节点是否为空，不为空则使用并将当前节点指向下一个，并减少poolsize。



![img](https://upload-images.jianshu.io/upload_images/5714046-9e8f1867b26f571d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应的，在释放的时候，message为recycle中会上锁后添加设置节点，增加poolsize，从而达成message的复用



![img](https://upload-images.jianshu.io/upload_images/5714046-748f4678a0cf449b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 关于对象池

在平时的逻辑中，如果我们想实现相关的享元复用，可以有多种方式，只要形成相应管理，用栈，队列，列表，数组，链表等结构进行管理，提供获取和回收方法。就可以达到相应目的。

实际上，在android的android.support.v4.util包中，就有提供Pools类来帮助我们事先缓存对象。提供了SimplePool、SynchronizedPool来创建对象池

在android.support.v4.util包下的Pools类中，分别声明了Pool接口，SimplePool实现类与SynchronizedPool实现类，其中具体的UML关系如下图所示：



![img](https://upload-images.jianshu.io/upload_images/5714046-f38f316a7f911531?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



其内部代码也比较简单，面向开发者提供了泛型的对象池，基于数组实现，其中对象池的最大容量是通过用户手动设定。从对象池中获取数据是通过acquire方法。回收当前对象到对象池中是通过release方法。

**关于acquire方法**

在acquire方法中，会从对象池中取出对象。具体列子如下图所示：



![img](https://upload-images.jianshu.io/upload_images/2824145-f78d7c7a39ac57c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

acquire()方法总会取当前对象池中存储的最后一个数据。如果有则返回。同时将该位置置为null。反之返回为null。

**关于release方法**

在release方法中，会将对象缓存到对象池中。如果当前对象已经存在，会抛出异常。反之则存储。具体列子如下图所示：





![img](https://upload-images.jianshu.io/upload_images/2824145-4a6a47a74c4d2913.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

release( T instance)方法，总会将需要回收的对象存入当前对象池中存储的最后一个数据的下一个位置。如果当前回收的对象已经存在会抛出异常。反之则成功。

**同步对象池（SynchronizedPool）**



![img](https://upload-images.jianshu.io/upload_images/5714046-c0022e54a0726df3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

SynchronizedPool的代码理解起来也同样非常简单，直接继承SimplePool。并重写了SimplePool的两个方法。并为其加上了锁，保证了多线程情况下使用的安全性。

Synchronized是通过对象内部的一个叫做监视器锁（monitor）来实现的。但是监视器锁本质又是依赖于底层的操作系统的Mutex Lock来实现的。而操作系统实现线程之间的切换这就需要从用户态转换到内核态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因。

因此，这种依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”。JDK中对Synchronized做的种种优化，其核心都是为了减少这种重量级锁的使用。JDK1.6以后，为了减少获得锁和释放锁所带来的性能消耗，提高性能，引入了“偏向锁”和“轻量级锁”。无锁 --> 偏向锁 --> 轻量级 --> 重量级

锁的方面有太多知识，这里不会铺开讲，有兴趣可以研究下做分享

让你彻底理解Synchronizedhttps://www.jianshu.com/p/d53bf830fa09

分门别类总结Java中的各种锁https://blog.csdn.net/renwei289443/article/details/79540809

JAVA锁有哪些种类https://blog.csdn.net/nalanmingdian/article/details/77800355

## 简单了解锁机制 

**概念：**

锁(lock)或互斥(mutex)是一种同步机制，用于在有许多执行线程的环境中强制对资源的访问限制。锁旨在强制实施互斥排他、并发控制策略

**Java线程阻塞：java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统介入，需要在户态与核心态之间切换，这种切换会消耗大量的系统资源**

Ø内核态: CPU可以访问内存所有数据, 包括外围设备, 例如硬盘, 网卡. CPU也可以将自己从一个程序切换到另一个程序

Ø用户态: 只能受限的访问内存, 且不允许访问外围设备. 占用CPU的能力被剥夺, CPU资源可以被其他程序获取

**Java锁的实现：**

markword是java对象数据结构中的一部分，用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，其最后2bit是锁状态标志位，表示不同级别的锁



![img](https://upload-images.jianshu.io/upload_images/5714046-673c77712f26a149.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

java对象结构

![img](https://upload-images.jianshu.io/upload_images/5714046-1d71abf370b7b187.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**对锁的不同效率进行的分类（无锁 --> 偏向锁 --> 轻量级 --> 重量级）** 

偏向锁：指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁。降低获取锁的代价。 

轻量级锁：指当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能。

 重量级锁： 指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁。重量级锁会让其他申请的线程进入阻塞，性能降低。

 **synchronized的执行过程：** 

检测Mark Word里面是不是当前线程的ID，如果是，表示当前线程处于偏向锁 

如果不是，则使用CAS将当前线程的ID替换Mard Word，如果成功则表示当前线程获得偏向锁，置偏向标志位1 

 如果失败，则说明发生竞争，撤销偏向锁，进而升级为轻量级锁。

 当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁

 如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

 如果自旋成功则依然处于轻量级状态。 

 如果自旋失败，则升级为重量级锁。

 Ps： **CAS：**即CompareAndSwap缩写，CAS需要有3个操作数：内存地址V，旧的预期值A，即将要更新的目标值B。CAS指令执行时，当且仅当内存地址V的值与预期值A相等时，将内存地址V的值修改为B，否则就什么都不做。整个比较并替换的操作是一个原子操作。（AtomicInteger）

**自旋锁：**假设持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起状态，只需要自旋（如while循环空转），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核的切换的消耗。自旋是需要消耗cup的，但不能一直占用cup自旋做无用功，所以需要设定一个自旋等待的最大时间 



**此外锁还具有多种分类和分类带来的锁名词**

Ø宏观设计理念分类——乐观锁和悲观锁

Ø从其它等待中的线程是否按顺序获取锁的角度划分——公平锁与非公平锁

Ø从能否有多个线程持有同一把锁的角度划分——互斥锁

Ø从一个线程能否递归获取自己的锁的角度划分——重入锁（递归锁）

Ø从编译器优化的角度划分——锁消除和锁粗化

Ø在不同的位置使用synchronized——类锁和对象锁

**锁优化：**

Ø减少锁的持有时间：例如避免给整个方法加锁

Ø减小锁的粒度：将大对象，拆成小对象，大大增加并行度，降低锁竞争.如此一来偏向锁，轻量级锁成功率提高.（代表类ConcurrentHashMap）

Ø读写分离锁替代独占锁：读-读不互斥，读-写互斥，写-写互斥

Ø锁分离：在读写锁的思想上做进一步的延伸,根据不同的功能拆分不同的锁,进行有效的锁分离.（代表类LinkedBlockingQueue）

Ø锁粗化：同一个锁不停的进行请求

同步和释放, 其本身也会消耗系统宝贵的资源,反而不利于性能的优化

Ø无锁：锁相比,使用CAS操作,由于其非阻塞性,因此不存在死锁问题,同时线程之间的相互影响, 也远小于锁的方式.使用无锁的方案,可以减少锁竞争以及线程频繁调度带来的系统开销.

## 关于线程的使用注意事项

在《阿里巴巴android开发手册》有提到



![img](https://upload-images.jianshu.io/upload_images/5714046-9bb5f7066edf158c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于线程池的使用注意



![img](https://upload-images.jianshu.io/upload_images/5714046-33f80193c0acc276?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于对象池复用，在《effective java》书中也有提到



![img](https://upload-images.jianshu.io/upload_images/5714046-426c2be1bba1939e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![img](https://upload-images.jianshu.io/upload_images/5714046-3a21c9e6dd63d8f3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考资料

- Java线程池实现原理与源码解析(jdk1.8)[https://blog.csdn.net/programmer_at/article/details/79799267#4-threadpoolexecutor%E6%BA%90%E7%A0%81](https://blog.csdn.net/programmer_at/article/details/79799267#4-threadpoolexecutor源码)

- 线程池的工作原理与源码解读https://www.cnblogs.com/qingquanzi/p/8146638.html、