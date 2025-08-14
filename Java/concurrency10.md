## Java 并发编程

### Lecture 10: 线程池

使用线程池比手动创建线程具有显著优势：

1. **降低资源消耗**
   - **线程复用**：线程池通过复用已有线程（如核心线程）避免频繁创建/销毁线程的开销（线程创建涉及操作系统资源分配、内存占用等成本）。
   - **可控资源占用**：通过限制最大线程数（如 `maximumPoolSize`），防止线程数量爆炸导致内存耗尽（如手动创建线程可能引发 `OutOfMemoryError`）。
2. **提高响应速度**
   - **任务即时处理**：当任务到达时，线程池中若有空闲线程可直接执行，无需等待线程创建（适合突发流量场景）。
   - **减少延迟**：对比手动创建线程的初始化时间（尤其是线程栈分配、上下文设置等），线程池显著缩短任务启动延迟。
3. **增强可管理性**
   - **统一调度与控制**：
     - 通过参数（核心线程数、队列容量、拒绝策略等）灵活调整线程行为。
     - 可监控线程状态（如活跃数、完成任务数）或通过钩子扩展（如 `beforeExecute()`）。
   - **内置容错机制**：线程池自动处理线程异常（默认打印堆栈，可通过重写 `afterExecute()` 捕获），避免手动线程因未捕获异常直接退出。
4. **避免不稳定风险**
   - **防资源耗尽**：
     - 通过任务队列（如 `LinkedBlockingQueue`）缓冲突发任务，避免瞬时高负载压垮系统。
     - 拒绝策略（如丢弃、调用者运行）保障线程池过载时系统仍能降级运行，而非崩溃。
   - **防线程泄漏**：线程池会自动回收空闲线程（通过 keepAliveTime 参数配置），而手动创建线程若未正确关闭可能导致长期闲置。

#### 线程池类

在java标准库中，`java.util.concurrent` 包提供了`ThreadPoolExecutor`的实现，用于创建和管理线程池。

```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    ThreadFactory threadFactory,
    RejectedExecutionHandler handler
) 
```

1. **核心线程数 (`corePoolSize`)**: 线程池中长期存活的线程数量，即使这些线程处于空闲状态也不会被回收（除非设置 `allowCoreThreadTimeOut=true`）。
2. **最大线程数 (`maximumPoolSize`)**: 线程池允许创建的最大线程数量（包含核心线程和非核心线程），当任务队列已满且核心线程忙碌时，线程池会创建新的临时线程（不超过此值）处理任务；若任务队列无界（如 `LinkedBlockingQueue`），此参数无效（因为队列永远不会满）。
3. **空闲线程存活时间 (`keepAliveTime` + `unit`)**：非核心线程（超出 `corePoolSize` 的线程）空闲时的存活时间，超时后会被回收。若调用 `allowCoreThreadTimeOut(true)`，核心线程也会受此时间限制。
4. **任务队列 (`workQueue`)**：用于缓存待执行任务（由`execute`提交的`Runnable`任务）的阻塞队列，用于平衡任务提交与处理速度，避免瞬时高峰压垮系统。常用的队列有：
   - `LinkedBlockingQueue`：无界队列（默认 `Integer.MAX_VALUE`），可能导致内存溢出
   - `ArrayBlockingQueue`：有界队列，需指定固定容量
   - `SynchronousQueue`：不存储任务，直接移交线程执行（需配合足够大的 `maximumPoolSize`），仅用于生产者与消费者的匹配，适用于高吞吐、短任务。
5. **线程工厂 (`threadFactory`)**：用于创建新线程的工厂类，可自定义线程名称、优先级、守护线程属性等，便于监控和调试。例如：
```java
ThreadFactory factory = r -> {
    Thread t = new Thread(r, "my-thread-" + counter.getAndIncrement());
    t.setDaemon(true); // 设置为守护线程
    return t;
};
```
在这里`ThreadFactory`是一个单抽象方法接口，定义为
```java
@FunctionalInterface
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```
被标记为函数式接口，表示可以用Lambda表达式或方法引用来创建实例，只需要类型匹配即可。

6. **拒绝策略 (`RejectedExecutionHandler`)**：当线程池和队列均满时，如何处理新提交的任务。内置策略包括：
    - `AbortPolicy`（默认）：直接抛出 `RejectedExecutionException`。
    - `CallerRunsPolicy`：由提交任务的线程直接执行该任务。
    - `DiscardPolicy`：静默丢弃任务。
    - `DiscardOldestPolicy`：丢弃队列中最旧的任务，然后重试提交。