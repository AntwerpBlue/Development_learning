# Java 并发编程

## Lecture 10: Synchronized 关键字

`synchronized` 关键字用于修饰方法或代码块，保证同一时刻只有一个线程可以执行被修饰的代码。主要用于：

- 确保线程互斥访问共享资源
- 保证变量的可见性（遵循 happens-before 原则）
- 防止指令重排

### 使用方法

1. 修饰实例方法

    ```java
    public synchronized void method() {
        // ...
    }
    ```

    - 锁对象是当前实例对象
    - 同一实例的多个同步方法会互斥

2. 修饰静态方法

    ```java
    public static synchronized void method() {
        // ...
    }
    ```

    - 锁对象是当前类的 Class 对象
    - 所有调用该静态方法的线程都会互斥

3. 修饰代码块

    ```java
    public void method() {
        synchronized(lockObject) {
            // ...
        }
    }
    ```

    - 锁对象可以是任意对象

### 实现原理

锁对象的获取和释放是由Monitor机制管理的。每个Java对象在内存中都分为三部分：

- 对象头(header)
  - Mark Word：存储对象的锁状态、哈希码、GC年龄
  - Klass Pointer：指向对象的类元数据
  - Array Length
- 实例数据
- 对齐填充（确保对象大小为8字节倍数）

在关联的Monitor中维护了以下结构：

- `Owner`：指向持有当前锁的线程
- `EntryList`：存储竞争锁失败的线程（BLOCK状态）
- `WaitSet`：等待条件的线程队列（WAITING状态）
- `Recursions`：锁被持有的次数（可重入锁特性）

获取锁时，检查Monitor的`Owner`字段，如果为空,CAS操作将`Owner`设置为当前线程，获取锁成功；如果已经是当前线程，`Recursions++`；
如果是其他线程，进入`EntryList`阻塞等待。

释放锁时，将`Recursions--`，如果为0，则`Owner`设置为null，唤醒`EntryList`中的线程。

#### 注：Monitor也是一个对象吗

Monitor不是一个显示的Java对象，而是JVM在底层隐式关联的，实际上是一个C++对象。同时Monitor是懒加载的，默认对象并不关联Monitor，直到锁升级到重量级锁时，JVM才会创建关联的Monitor，并将Mark Word替换为指向Monitor的指针.

同样，java中的引用实际上在JVM内部也是通过指针来实现的，但java语言设计上为了避免直接内存操作，对开发者屏蔽了指针的概念。

### 锁升级

Java中的锁有四种状态，按照升级顺序，分别在Mark Word中的布局为：

|锁状态|Mark Word(64 bits)|说明|
|---|---|---|
|无锁|`unused:25` \| `identity_hashcode:31` \| `unused:1` \| `age:4` \| `biased_lock:1` \| `lock:2`|存储哈希码、分代年龄等|
|偏向锁|`thread:54` \| `epoch:2` \| `unused:1` \| `age:4` \| `biased_lock:1` \| `lock:2`|存储持有偏向锁的线程 ID|
|轻量级锁|`ptr_to_lock_record:62` \| `lock:2`|指向栈中锁记录的指针|
|重量级锁|`ptr_to_heavyweight_monitor:62` \| `lock:2`|指向 Monitor的指针|

#### 升级流程

- 初始状态：无锁
  - 对象刚被创建，没有任何线程访问
- 第一阶段：偏向锁
  - 第一个线程访问同步代码块，JVM启动偏向锁（默认在应用启动后几秒才激活）
  - 检查对象头中线程ID是否指向当前线程，是则进入同步块；否则尝试通过CAS操作替换线程ID：
    - 如果成功则获得偏向锁
    - 否则说明存在竞争，撤销偏向锁，升级为轻量级锁
- 第二阶段：轻量级锁
  - 偏向锁被撤销（第二个线程尝试获取锁）且线程竞争不激烈时，升级为轻量级锁
  - JVM在线程栈帧中创建锁记录（Lock Record），并将对象头中的Mark Word复制到锁记录中，然后CAS操作将对象头中的Mark Word替换为指向锁记录的指针：
    - 如果成功则获得轻量级锁
    - 否则锁重入或说明存在竞争，升级为重量级锁
- 第三阶段：重量级锁
  - 轻量级锁被撤销，自旋次数达到阈值后，升级为重量级锁
  - JVM创建一个Monitor对象，并将对象头中的Mark Word替换为指向Monitor对象的指针，竞争失败的线程进入Monitor的EntryList中等待
  - 使用操作系统的互斥量（mutex）来实现锁，线程的阻塞和解锁操作需要从用户态切换到内核态，开销较大

#### 偏向锁的延迟启动

在 JVM 启动阶段，虽然存在并发操作（如类加载、JIT 编译、GC 线程活动等），但这些操作不会因偏向锁延迟而导致冲突。

在延迟期内，所有对象的锁直接进入轻量级锁状态，避免偏向锁的撤销开销。JVM内部的关键系统对象（如类元数据、符号表）的同步不依赖 Java 层的 `synchronized`，而是使用：

- 底层原生锁（如 `pthread_mutex_t`）：由 JVM 的 C++ 代码直接管理
- 原子操作（CAS）：保证基础操作的线程安全
- 安全点（Safepoint）同步：在全局停顿阶段执行敏感操作

### CAS 操作

CAS（Compare-and-Swap）操作是硬件级别的原子操作，用于实现无锁算法。CAS操作包含三个参数：内存地址V、旧值A、新值B。CAS操作会检查内存地址V中的值是否等于旧值A，如果相等，则将新值B写入内存地址V，返回true；如果不相等，则返回false。
