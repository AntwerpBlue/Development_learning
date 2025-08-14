## Java 并发编程

### Lecture 11: 常见线程池

在 Java 中，通过 `java.util.concurrent.Executors` 工具类可以快速创建 6 种常见的线程池，每种线程池适用于不同的业务场景

#### FixedThreadPool（固定大小线程池）

```java
ExecutorService executor = Executors.newFixedThreadPool(int nThreads);
```

特点：

- 固定线程数量（`corePoolSize = maximumPoolSize = nThreads`）
- 使用无界队列 `LinkedBlockingQueue`，任务可能无限堆积，需警惕 OOM（OutOfMemoryError）。

适用场景：适合负载稳定且需要限制线程数的场景（如 HTTP 请求处理）。

#### CachedThreadPool（可缓存线程池）

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

特点：

- 线程数弹性伸缩（`corePoolSize=0`，`maximumPoolSize=Integer.MAX_VALUE`）。
- 使用 `SynchronousQueue`，任务会立即执行，若没有空闲线程则创建新线程
- 空闲线程超时回收（默认 60 秒）。

适用场景：适合**短时异步任务**且任务量波动大的场景（如突发流量）。

#### SingleThreadExecutor（单线程池）

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

特点：

- 仅 1 个核心线程（`corePoolSize = maximumPoolSize = 1`）。
- 使用无界队列 `LinkedBlockingQueue`。
- 保证任务顺序执行（FIFO）。

适用场景：需要**任务串行执行**的场景（如日志顺序写入、单任务调度）。

#### ScheduledThreadPool（定时任务线程池）/ SingleThreadScheduledExecutor（单线程定时任务池）

```java
ExecutorService executor = Executors.newScheduledThreadPool(int corePoolSize);
ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
```

特点：

- 支持 定时/周期性任务（如 scheduleAtFixedRate）。
- 使用 `DelayedWorkQueue`，任务会按延迟时间排序执行。

适用场景：定时任务（如心跳检测）、周期性任务（如数据定时同步）。

#### WorkStealingPool（工作窃取线程池）

```java
ExecutorService executor = Executors.newWorkStealingPool(int parallelism);
```

特点：
    - 使用 `ForkJoinPool` 实现，支持**工作窃取算法**（空闲线程偷其他队列任务）
    - 默认并行度为 `Runtime.getRuntime().availableProcessors()`。

适用场景：计算密集型任务（如并行流处理、分治算法），能够提高 CPU 利用率，减少线程竞争。


|线程池类型|核心线程数|最大线程数|任务队列|适用场景|
|---|---|---|---|---|
|FixedThreadPool|固定 (nThreads)|固定 (nThreads)|LinkedBlockingQueue|稳定负载，限制线程数|
|CachedThreadPool|0|Integer.MAX_VALUE|SynchronousQueue|短时任务，突发流量|
|SingleThreadExecutor|1|1|LinkedBlockingQueue|单线程顺序执行|
|ScheduledThreadPool|自定义|Integer.MAX_VALUE|DelayedWorkQueue|定时/周期性任务|
|SingleThreadScheduledExecutor|1|1|DelayedWorkQueue|单线程定时任务|
|WorkStealingPool|并行度 (CPU核数)|无上限|工作窃取队列|计算密集型任务（Java 8+）|

#### ForkJoinPool（工作窃取线程池）

`ForkJoinPool`是一种专为分治任务设计的线程池，基于工作窃取算法实现高效并行计算，适合处理递归分解的可并行任务（如大规模数据处理、递归算法等）。

##### 工作窃取算法

- 每个线程维护一个双端队列，优先处理自己队列中的任务
- 当某个线程的队列为空时，会从其他线程队列的尾部偷取任务执行（减少竞争）
- 优势：最大化 CPU 利用率，避免线程闲置。

##### 分治任务

- fork()：将任务分解为子任务，并提交给线程池
- join()：等待子任务完成并获取结果

##### 示例

计算1~n的和

```java
import java.util.concurrent.*;

class SumTask extends RecursiveTask<Long> {
    private final long start;
    private final long end;
    private static final long THRESHOLD = 10_000; // 阈值：小于此值直接计算

    SumTask(long start, long end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // 小任务直接计算
            long sum = 0;
            for (long i = start; i <= end; i++) sum += i;
            return sum;
        } else {
            // 大任务拆分（分治）
            long mid = (start + end) / 2;
            SumTask leftTask = new SumTask(start, mid);
            SumTask rightTask = new SumTask(mid + 1, end);
            leftTask.fork(); // 异步执行左子任务
            return rightTask.compute() + leftTask.join(); // 合并结果
        }
    }
}

public class Main {
    public static void main(String[] args) {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        SumTask task = new SumTask(1, 1_000_000);
        long result = pool.invoke(task); // 提交任务并获取结果
        System.out.println("Sum: " + result); // 输出 500000500000
    }
}
```

#### 为什么不应该自动创建线程池

虽然通过 Executors 工具类可以快速创建线程池（如 newFixedThreadPool、newCachedThreadPool），但阿里 Java 开发规范等最佳实践明确建议 不要直接使用这些便捷方法，而是应该 手动创建 ThreadPoolExecutor。以下是具体原因和底层逻辑：

- 无界队列导致内存溢出（OOM）
  
  FixedThreadPool 和 SingleThreadExecutor 默认使用 无界队列，当任务提交速度持续高于处理速度时，队列会无限堆积任务。

- 线程数失控引发资源耗尽

  CachedThreadPool 和 ScheduledThreadPool 默认使用 SynchronousQueue，当任务数量暴增时，会无限创建线程，导致系统资源（CPU、内存）耗尽。

- 无法感知任务执行异常

  Executors 创建的线程池默认不捕获任务抛出的异常，异常信息会丢失。

- 与监控系统不兼容

    自动化创建的线程池难以集成监控（如线程活跃数、队列大小、任务耗时）。