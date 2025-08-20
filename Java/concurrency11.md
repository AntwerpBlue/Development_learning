# Java 并发编程

## Lecture 11: Lock类显式锁

Lock是`java.util.concurrent.locks`中提供的显式锁机制，用于替代传统的`synchronized`关键字，提供更灵活、更强大的线程同步功能。它的核心实现类是`ReentrantLock`，此外还有`ReadWriteLock`（读写锁）等。

### 基本使用
  
```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockDemo {
    private final Lock lock = new ReentrantLock(); // 默认非公平锁
    
    public void doTask() {
        lock.lock(); // 加锁
        try {
            // 临界区代码
            System.out.println("Lock acquired by " + Thread.currentThread().getName());
        } finally {
            lock.unlock(); // 必须在 finally 中解锁，避免死锁
        }
    }
}
```
  
- 需要手动管理锁的获取和释放，使用`lock()`方法获取锁，使用`unlock()`方法释放锁，避免死锁。
- 一般使用`try-finally`结构来确保锁的释放，即使在临界区代码抛出异常的情况下也能保证锁的正确释放。

### 高级控制方法

1. `tryLock()`

   - 作用：尝试非阻塞获取锁，成功返回 true，失败返回 false。
   - 示例：
  
    ```java
    if (lock.tryLock()) {
        try {
            // 获取锁成功，执行临界区代码
        } finally {
            lock.unlock();
        }
    } else {
        // 获取锁失败，执行其他逻辑
    }
    ```

2. `tryLock(long timeout, TimeUnit unit)`

   - 作用：超时等待获取锁，在指定时间内尝试，超时返回 false。
   - 特点：可响应中断，等待期间调用`interrupt()`方法会抛出`InterruptedException`。
   - 示例：

    ```java
    try {
        if (lock.tryLock(2, TimeUnit.SECONDS)) {  // 最多等待2秒
            try {
                // 临界区代码
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("获取锁超时");
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();  // 恢复中断状态
    }
    ```

3. `lockInterruptibly()`
  
   - 作用：可中断获取锁，如果锁被占用，线程会阻塞但可被其他线程中断。
   - 适用场景：需要响应中断的长时间任务。
   - 示例：
  
    ```java
    try {
        lock.lockInterruptibly();  // 可被中断的阻塞
        try {
            // 临界区代码
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        System.out.println("线程被中断");
        Thread.currentThread().interrupt();
    }
    ```

4. `newCondition()`

   - 作用：创建条件变量（`Condition`），用于更精细的线程等待/唤醒控制
   - 对比`synchronized`：
     - `synchronized`只能使用`wait()/notify()`
     - `Condition`支持多个等待队列（如生产者-消费者模型）
   - 示例：

    ```java
    Lock lock = new ReentrantLock();
    Condition notEmpty = lock.newCondition();  // 条件变量

    // 等待条件
    lock.lock();
    try {
        while (queue.isEmpty()) {
            notEmpty.await();  // 释放锁并等待
        }
        // 条件满足后继续执行
    } finally {
        lock.unlock();
    }

    // 唤醒等待线程
    lock.lock();
    try {
        notEmpty.signal();  // 唤醒一个等待线程
    } finally {
        lock.unlock();
    }
    ```

### 和synchronized的对比

||`synchronized`（JVM内置）|`Lock`（`ReentrantLock`等）|
|---|---|---|
|锁获取|自动加锁/释放（通过代码块或方法）|手动调用 `lock()/unlock()`（需配合 `try-finally`）|
|锁类型|非公平锁（可升级为公平锁）|可选择公平锁或非公平锁|
|中断响应|阻塞时无法中断|支持中断（`lockInterruptibly()`）|
|超时机制|不支持|支持（`tryLock(time,unit)`）|
|条件变量|仅通过`wait()/notify()`实现|支持多个`Condition`（更灵活的线程通信）|

- 选择`synchronized`：
  - **简单同步需求**：如单方法内的线程安全控制
  - **低竞争环境**：如单例模式的双重检查锁
  - **代码简洁优先**：避免手动管理锁释放
- 选择`Lock`：
  - **需要高级功能**：如可中断、超时、公平锁
  - **复杂同步逻辑**：如跨多个方法的锁控制
  - **高竞争环境**：如秒杀系统中的库存扣减
