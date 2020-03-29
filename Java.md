1. [线程池ThreadPoolExecutor](#线程池threadpoolexecutor)
	1. [构造参数说明](#构造参数说明)
	2. [线程池参数设置](#线程池参数设置)
	3. [线程池执行原理](#线程池执行原理)
	4. [线程池状态](#线程池状态)
	5. [JDK提供的实现类](#jdk提供的实现类)
		1. [FixedThreadPool](#fixedthreadpool)
		2. [CachedThreadPool](#cachedthreadpool)
		3. [SingleThreadExecutor](#singlethreadexecutor)
		4. [ScheduledThreadPool](#scheduledthreadpool)
			1. [构造方法](#构造方法)
			2. [应用](#应用)
	6. [线程池的监控](#线程池的监控)

# 线程池ThreadPoolExecutor
## 构造参数说明

``` java
public ThreadPoolExecutor(int corePoolSize, // 1
                        int maximumPoolSize,  // 2
                        long keepAliveTime,  // 3
                        TimeUnit unit,  // 4
                        BlockingQueue<Runnable> workQueue, // 5
                        ThreadFactory threadFactory,  // 6
                        RejectedExecutionHandler handler ) { //7
```

 - **corePoolSize** ：核心线程数量，默认不会被回收，会一直存在于线程池中,设置allowCoreThreadTimeout=true（默认false）时，核心线程会超时关闭
 - **maximumPoolSize** ：线程池中最大线程数，包含核心线程数
 - **keepAliveTime** ： 非核心线程空闲超时时间
 - **unit** ：keepAliveTime的单位
 - **workQueue** : 阻塞队列，它一般分为
	- 直接提交队列(SynchronousQueue)
	- 有界任务队列（ArrayBlockingQueue）
	- 无界任务队列(LinkedBlockingQueue)
	- 优先任务队列（PriorityBlockingQueue）
 - **threadFactory** ：线程工厂，用于创建线程，可以自定义，重命名线程名称，一般用默认即可
 - **handler**: 拒绝策略；当任务太多来不及处理时，如何拒绝任务；
	- 	new ThreadPoolExecutor.AbortPolicy();  将抛出 RejectedExecutionException.
	- 	new ThreadPoolExecutor.CallerRunsPolicy(); 任务将由调用者线程去执行
	- 	new ThreadPoolExecutor.DiscardOldestPolicy();  丢弃任务队列中最旧任务
	- 	new ThreadPoolExecutor.DiscardPolicy(); 丢弃当前任务
## 线程池参数设置
 - 任务类型
	 - IO密集型=2Ncpu（可以测试后自己控制大小，2Ncpu一般没问题）（常出现于线程中：数据库数据交互、文件上传下载、网络数据传输等等）
	 - 计算密集型=Ncpu（常出现于线程中：复杂算法）
	 - 	java中：```Ncpu=Runtime.getRuntime().availableProcessors()```
 - 推算线程池参数：
	 - 	**maximumPoolSize** :设置为2Ncpu或Ncpu，根据任务类型属于IO密集型还是计算密集型
	 - 	**corePoolSize** 设置为 maximumPoolSize 
	 - 	**workQueue** 队列长度：L = timeout * ( corePoolSize / t )		 
	> 队列长度L，线程个数corePoolSize ,每个任务执行耗时t 秒, 每秒可以处理 corePoolSize  / t
	> 个任务，队列最后一个任务处理完总耗时timeout= L / (corePoolSize  / t) =>也是任务的最大超时时间

## 线程池执行原理

> 问题： corePoolSize 是 5 ， maximumPoolSize  是 10，workQueue  队列长度是10，现同时提交30个任务，线程池是如何工作的。
>
> 答：先创建5个核心线程来执行5个任务，然后再将10个任务存到workQueue  队列中，再创建5个非核心线程来执行5个任务，剩下的10个任务会使用拒绝策略处理

## 线程池状态


| 状态       | 解释                                                               |
| ---------- | ------------------------------------------------------------------ |
| RUNNING    | 运行态，可处理新任务并执行队列中的任务                             |
| SHUTDOWN   | 关闭态，不接受新任务，但继续处理队列中的任务                       |
| STOP       | 停止态，不接受新任务，不处理队列中任务，且打断运行中任务           |
| TIDYING    | 整理态，所有任务已经结束，workerCount = 0 ，将执行terminated()方法 |
| TERMINATED | 结束态，terminated() 方法已完成                                 |

![](images/1bbb6bfc.png)

## JDK提供的实现类
### FixedThreadPool

``` java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                        0L, TimeUnit.MILLISECONDS,
                                        new LinkedBlockingQueue<Runnable>());
}
```

 - corePoolSize与maximumPoolSize相等，即其线程全为核心线程，是一个固定大小的线程池，是其优势
 - keepAliveTime = 0 该参数默认对核心线程无效，而FixedThreadPool全部为核心线程
 - 	workQueue 为LinkedBlockingQueue（无界阻塞队列），队列最大值为Integer.MAX_VALUE。如果任务提交速度持续大余任务处理速度，会造成队列大量阻塞。因为队列很大，很有可能在拒绝策略前，内存溢出。是其劣势；

### CachedThreadPool

``` java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                        60L, TimeUnit.SECONDS,
                                        new SynchronousQueue<Runnable>());
}
```

 -	 corePoolSize = 0，maximumPoolSize = Integer.MAX_VALUE，即线程数量几乎无限制；
 -	  keepAliveTime = 60s，线程空闲60s后自动结束。
 -	  workQueue 为 SynchronousQueue 同步队列，这个队列类似于一个接力棒，入队出队必须同时传递，因为CachedThreadPool线程创建无限制，不会有队列等待，所以使用SynchronousQueue；

### SingleThreadExecutor

``` java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
                        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

 - 	单线程，同FixedThreadPool类似
 
### ScheduledThreadPool
#### 构造方法
``` java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}
		
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue());
}

```
#### 应用

``` java
public class ScheduledTest {
    public static void main(String[] args) throws IOException {
        final ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(3);
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("Thread"+ "This task is delayed to execute");
            }
        };
        scheduledThreadPool.scheduleAtFixedRate(runnable,0,1,TimeUnit.SECONDS);
        System.in.read();
    }
}
```

 - 	scheduleAtFixedRate ： 是以上一个任务开始的时间计时，period时间过去后，检测上一个任务是否执行完毕，如果上一个任务执行完毕，则当前任务立即执行，如果上一个任务没有执行完毕，则需要等上一个任务执行完毕后立即执行。
 - 	scheduleWithFixedDelay，是以上一个任务结束时开始计时，period时间过去后，立即执行。

## 线程池的监控

通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

 - getTaskCount：线程池已经执行的和未执行的任务总数；
 - getCompletedTaskCount：线程池已完成的任务数量，该值小于等于taskCount；
 - getLargestPoolSize：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize；
 - getPoolSize：线程池当前的线程数量；
 - getActiveCount：当前线程池中正在执行任务的线程数量。