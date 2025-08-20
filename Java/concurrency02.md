# Java 并发编程

## Lecture 2: 线程的停止

### 正确的停止线程

1. 使用中断机制（interruption）

    ```java
    public class ProperThreadStop implements Runnable {
        @Override
        public void run() {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // 正常工作代码
                    Thread.sleep(1000); // 会响应中断
                } catch (InterruptedException e) {
                    // 恢复中断状态（重要！）
                    Thread.currentThread().interrupt();
                    // 执行清理工作后退出
                    break;
                }
            }
            System.out.println("线程优雅终止");
        }

        public static void main(String[] args) throws InterruptedException {
            Thread thread = new Thread(new ProperThreadStop());
            thread.start();
            Thread.sleep(3000);
            thread.interrupt(); // 请求中断
        }
    }
    ```

2. 使用Future取消任务（适用于线程池）

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<?> future = executor.submit(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 任务代码
    }
});

// 取消任务
future.cancel(true); // true表示尝试中断线程
```

### 为什么使用volatile关键字是错误的

```java
public class FlawedStop {
    private volatile boolean stopped = false;

    public void run() {
        while (!stopped) {
            // 工作代码
            try {
                Thread.sleep(1000); // 阻塞操作
            } catch (InterruptedException e) {
                // 忽略中断
            }
        }
    }

    public void stop() {
        stopped = true;
    }
}
```

- 无法中断阻塞操作
  - 当线程在sleep、wait、I/O操作时，即使`volatile`标志改变，也不会立即停止线程
  - 必须等待阻塞操作完成后才会检查标志位
- 中断信号丢失
  - 阻塞方法在被中断时会抛出`InterruptedException`并清除中断状态
  - 仅用`volatile`标志无法重新设置中断状态，导致中断信号丢失
- 资源释放问题
  - 无法保证线程能执行必要的清理工作
  - 可能导致文件、数据库连接等资源泄漏
- 无法响应系统关闭
  - JVM关闭时，无法通过这种机制通知线程退出

### 恢复中断

当线程被中断时，如果该线程正处于阻塞状态（如`sleep()`、`wait()`或`join()`），这些阻塞方法会抛出`InterruptedException`，同时清除线程的中断状态（设为`false`）。恢复中断是指在捕获这个异常后，重新设置线程的中断状态。

|代码执行点|中断状态|说明|
|:---:|:---:|:---:|
|线程启动后|false|初始状态|
|调用interrupt()后|true|中断请求已发出|
|进入sleep()时|true|阻塞前状态|
|抛出InterruptedException时|false|**异常抛出后JVM自动清除中断状态**|
|调用interrupt()恢复后|true|手动恢复中断状态|
