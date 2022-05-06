# JUC系列 - ThreadPoolExecutor源码解读

## 源码解读

ThreadPoolExecutor 中 工作线程是由 `BlockingQueue<Runnable> workQueue` 进行管理的，将Thread包装为Worker，任务由对应的缓存队列来存储管理



## 面试题

1. 说说线程池的实现原理

   ```java
   public ThreadPoolExecutor(int corePoolSize,
                             int maximumPoolSize,
                             long keepAliveTime,
                             TimeUnit unit,
                             BlockingQueue<Runnable> workQueue,
                             ThreadFactory threadFactory,
                             RejectedExecutionHandler handler) {
       if (corePoolSize < 0 ||
           maximumPoolSize <= 0 ||
           maximumPoolSize < corePoolSize ||
           keepAliveTime < 0)
           throw new IllegalArgumentException();
       if (workQueue == null || threadFactory == null || handler == null)
           throw new NullPointerException();
       this.corePoolSize = corePoolSize;
       this.maximumPoolSize = maximumPoolSize;
       this.workQueue = workQueue;
       this.keepAliveTime = unit.toNanos(keepAliveTime);
       this.threadFactory = threadFactory;
       this.handler = handler;
   }
   ```

   corePoolSize是线程池的核心线程数，当有任务提交到线程池后，线程池会创建一个线程来执行任务。此时，即使存在其他空闲核心线程，也仍然会创建线程，直至达到核心线程数设置数量的上限。如果还有其他任务需要继续 执行，线程池会把新的任务放到缓存队列中排队等待。另一个重要的参数是maximumPoolSize，它指的是线程池允许创建的线程数的最大值，如果新任务增长的速度远远快于线程池处理的速度，导致缓存队列满且产生积压，但是创建的线程数小于线程池的最大线程数，则线程池会再创建新的线程来执行任务。但是，当创建的线程数大于线程池的最大线程数，就会采取handler 设置的拒绝策略。例如，AbortPolicy (直接抛出异常)、CallerRunsPolicy(调用线程来处理任务)、DiscardOldestPolicy (丢弃队列最前面的任务，并 执行当前任务)、DiscardPolicy (直接丢弃)。当然，也可以自定义拒绝策 略。

   此外，参数keepAliveTime设置超过线程池核心线程数的线程存活时间，参数unit定义了超过线程池核心线程数的线程存活时间的单位。当一个线程处理完任务，处于空闲状态并超过线程存活时间时，线程池可以对其进行回 收，恢复到corePoolSize 设置的线程池核心线程数，如果线程池中的线程数 量少于corePoolSize,则该参数不起作用。换句话说，corePoolSize 设置的线 程池核心线程数就是线程池中需要保留的线程数量。workQueue 参数用于指 定存放任务的队列类型,包括ArrayBlockingQueue、LinkedBlockingQueue、 SynchronousQueue等。其中，ArrayBlockingQueue 是数组结构的有界阻塞 队列; LinkedBlockingQueue 是链表结构的阻塞队列，默认大小是 Integer.MAX_VALUE的无界队列; SynchronousQueue 是不存储元素的阻塞 队列，每次插入数据时，它必须等待另一个线程调用移除数据操作。另一个 参数ThreadFactory是线程池创建新线程的工厂类。
   
2. 为什么小于corePoolSize时，就会立即执行任务？超过corePoolSize时，任务并不会立即执行？

3. 