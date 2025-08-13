## Java 并发编程

### Lecture 6: 线程六种状态的转换

#### 线程状态

Java线程在其生命周期中会在6种状态间转换

|状态|说明|
|---|---|
|NEW|初始状态，线程被创建，但尚未启动|
|RUNNABLE|可运行状态（包括正在运行和就绪）|
|BLOCKED|等待监视器锁（synchronized）|
|WAITING|等待状态，线程正在等待另一个线程执行特定操作|
|TIMED_WAITING|计时等待状态，线程正在等待另一个线程执行特定操作，但有时间限制|
|TERMINATED|终止状态，线程已经完成执行|

#### 线程状态转换

1. NEW -> RUNNABLE
   
当调用`thread.start()`时触发转换，不可逆

2. RUNNABLE <-> BLOCKED
   - RUNNABLE -> BLOCKED：尝试进入`synchronized`块/方法时锁已被其他线程持有
   - BLOCKED -> RUNNABLE：线程获取到监视器锁时，进入RUNNABLE状态

3. RUNNABLE <-> WAITING
   - 触发方式
     - `Object.wait()`
     - `Thread.join()`
     - `LockSupport.park()`
   - 无限期等待，必须显式唤醒。唤醒方式
     - `Object.notify()`/`notifyAll()`
     - `LockSupport.unpark(thread)`
     - 被join的线程终止
     - 被中断（抛出`InterruptedException`）

4. RUNNABLE <-> TIMED_WAITING
   - 触发方式
     - `Thread.sleep(long)`
     - `Object.wait(timeout)`
     - `Thread.join(timeout)`
     - `LockSupport.parkNanos()`/`parkUntil()`
   - 唤醒方式：
     - 超时后自动唤醒
     - 其他线程提前唤醒（如`notify()`）
     - 被中断（抛出`InterruptedException`）

5. RUNNABLE -> TERMINATED
   - `thread.run()`执行完毕
   - 未捕获的异常导致线程终止

#### wait/notify/notifyAll

##### wait的使用

Java要求线程在调用`Object.wait()`时必须持有该对象的监视器锁（即必须在`synchronized`块中调用），在调用wait后立即释放当前持有的锁。如果没有锁保护，可能会出现如下的竞态条件：
```java
// 伪代码展示问题
if (!condition) {       // 1. 检查条件
    obj.wait();         // 3. 此时条件可能已改变！
}
// 另一个线程可能在这之间执行：
condition = true;       // 2. 修改条件
obj.notify();           // 但此时原线程还没进入等待，通知丢失
```
为了避免上述问题，Java要求在调用`wait`时必须持有锁，并使用while循环而非if判断检查条件：
```java
synchronized (obj) {
    while (!condition) {
        obj.wait();
    }
}
```
在调用`wait`时，JVM需要
- 原子性地将线程加入锁对象头的等待集中
- 释放锁以允许其他线程获得锁并修改条件
- 在唤醒后重新获取锁

##### notify/notifyAll的使用

与`wait`类似，`notify`和`notifyAll`也必须在`synchronized`块中调用，否则会抛出`IllegalMonitorStateException`。

`notify`在使用时还需要注意通知时机，先修改条件再唤醒线程，否则可能会造成信号丢失：
```java
synchronized(lock) {
    condition = true;  // 先修改条件
    lock.notify();     // 后发送通知
}
```

||notify|notifyAll|
|---|---|---|
|唤醒线程数|随机一个|所有等待线程|
|适用场景|单消费者明确知道只需唤醒一个|多消费者/条件可能满足多个线程|
|性能|更高|更低（但通常差别不大）|
|安全性|可能造成信号丢失|更安全|

推荐使用`notifyAll`，除非
- 明确知道只有一个线程需要唤醒
- 性能敏感且能保证正确性

#### join
