## Java 并发编程

### Lecture 1: 为什么说只有一种实现线程的方式

由于Java的跨平台性，所有底层操作都通过JVM提供的标准库实现，对不同OS的适配由JVM完成；并且Java禁止访问底层系统资源，提供较高的安全性，因此Java无法直接进行系统调用。

Java 线程的底层实现，最终都是通过 `java.lang.Thread` 类来创建和管理的.
无论你使用哪种方式（Runnable、线程池、Callable、CompletableFuture 等），最终都会依赖 Thread 类，并由 JVM 调用操作系统（OS）的线程机制（如 Linux 的 pthread、Windows 的线程 API）来执行。

#### 继承 `Thread` 类
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Hello, I am a thread!");
    }
}

MyThread t = new MyThread();
t.start();  // 最终调用 OS 线程
```

#### 实现 `Runnable` 接口
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Hello, I am a thread!");
    }
}

Thread t = new Thread(new MyRunnable());
t.start();  // 最终调用 OS 线程
```

#### `Callable + FutureTask`
```java
Callable<String> task = () -> "Result";
FutureTask<String> futureTask = new FutureTask<>(task);
Thread t = new Thread(futureTask);  // 还是 Thread！
t.start();
String result = futureTask.get();  // 获取返回值
```

#### 使用线程池
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(() -> {
    System.out.println("Hello, I am a thread!");
});
executor.shutdown();
```

