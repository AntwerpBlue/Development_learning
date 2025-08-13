## Java 并发编程

### Lecture 3: synchronized

#### 基本作用

`synchronized` 关键字用于修饰方法或代码块，保证同一时刻只有一个线程可以执行被修饰的代码。主要用于：
- 确保线程互斥访问共享资源
- 保证变量的可见性（遵循 happens-before 原则）
- 防止指令重排

#### 使用方法

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

#### 实现原理

锁对象的获取和释放是由Monitor机制管理的。每个Java对象在内存中都分为三部分：
- 对象头(header)
  - Mark Word：存储对象的锁状态、哈希码、GC年龄
  - Klass Pointer：指向对象的类元数据
  - Array Length
- 实例数据
- 对齐填充（确保对象大小为8字节倍数）

在关联的Monitor中维护了以下结构：
- Owner：指向持有当前锁的线程
- EntryList：存储竞争锁失败的线程（BLOCK状态）
- WaitSet：等待条件的线程队列（WAITING状态）
- Recursions：锁被持有的次数（可重入锁特性）

获取锁时，检查Monitor的Owner字段，如果为空,CAS（Compare-and-Swap）操作将Owner设置为当前线程，获取锁成功；如果已经是当前线程，Recursions++；
如果是其他线程，进入EntryList阻塞等待。

释放锁时，将Recursions--，如果为0，则Owner设置为null，唤醒EntryList中的线程。

#### CAS 操作

CAS操作是硬件级别的原子操作，用于实现无锁算法。CAS操作包含三个参数：内存地址V、旧值A、新值B。CAS操作会检查内存地址V中的值是否等于旧值A，如果相等，则将新值B写入内存地址V，返回true；如果不相等，则返回false。

#### 注：Monitor也是一个对象吗

Monitor不是一个显示的Java对象，而是JVM在底层隐式关联的，实际上是一个C++对象。同时Monitor是懒加载的，默认对象并不关联Monitor，直到锁升级到重量级锁时，JVM才会创建关联的Monitor，并将Mark Word替换为指向Monitor的指针.

同样，java中的引用实际上在JVM内部也是通过指针来实现的，但java语言设计上为了避免直接内存操作，对开发者屏蔽了指针的概念。