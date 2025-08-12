## Java 并发编程

### Lecture 4: 原子操作

原子操作是指不可分割的操作，在执行过程中不会被线程调度机制中断，要么完全执行成功，要么完全不执行，没有中间状态。

在Java中，基本数据类型（除long/double）、引用类型与volatile变量（包括long/double）的读写操作是原子的。Java还提供了`java.util.concurrent.atomic`包来实现更复杂的原子操作
```java
AtomicInteger atomicInt = new AtomicInteger(0);

// 原子递增
atomicInt.incrementAndGet(); 

// CAS操作
atomicInt.compareAndSet(expect, update);
```

#### 原子类

原子类通过CAS操作来实现线程安全。Java原子类使用`Unsafe`类，提供底层硬件级别的原子操作，能够直接操作内存。例如在`AtomicInteger`中
```java
public class AtomicInteger {
    private volatile int value;
    private static final long valueOffset;
    
    static {
        try {
            valueOffset = Unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
}
```
value以volatile修饰以确保线程可见性，valueOffset代表value在内存中的位置。以`getAndAddInt`为例:
```java
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // 读取当前值
    } while (!compareAndSwapInt(o, offset, v, v + delta)); // CAS尝试更新
    return v;
}
```
通过循环CAS实现"读取-修改-写入"的原子性

#### volatile关键字

在多线程环境下，一个线程对共享变量的修改，其他线程可能无法立即看到，这种现象称为可见性问题。这是由于：
- CPU缓存架构：现代CPU有多级缓存，线程可能读取的是缓存中的旧值
- 指令重排序：编译器和处理器可能优化指令执行顺序

当一个变量被声明为`volatile`时，对该变量的修改会立刻刷新到主存；每次读取该变量也会直接从主存获取新值，从而确保可见性。

在JVM中，通过插入内存屏障指令实现`volatile`的语义
- 写屏障（Store Barrier）：确保 volatile 写操作前的所有普通写操作都刷新到主内存
- 读屏障（Load Barrier）：确保 volatile 读操作后的所有普通读操作都从主内存读取

同时`volatile`变量遵循happens-before规则，即对一个`volatile`变量的写操作，happens-before于后续对这个变量的读操作

#### 原子类在高并发下的性能降低

在高并发环境下，原子类性能不佳基于以下原因：
- CAS自旋开销：在高并发情况下，大量线程同时执行CAS操作，导致大量自旋，浪费CPU资源
- 伪共享问题：当多个原子变量位于同一个缓存行中时，一个线程对其中一个变量的修改会导致整个缓存行失效，其他线程访问该缓存行中的其他变量时需要重新从主存加载，即便它们操作的是不同的变量

为了解决高并发下的性能问题，主要途径包括：
- 降低并发竞争：分散竞争点、批处理减少CAS次数
- 消除伪共享：使用缓存行填充、使用`@Contended`注解
- 内置优化策略：如`LongAdder`代替`AtomicLong`、分段锁

#### 原子类、volatile和happens-before的关系

#### volatile与原子类的比较

`volatile`和原子类都可以保证可见性，并禁止指令重排

|特性|volatile|原子类|
|---|---|---|
|原子性保证|仅保证单次读/写的原子性|保证复合操作(如i++)的原子性|
|性能开销|较低|略高(CAS操作需要重试)|
|适用操作|简单状态标志|计数器等需要原子更新的场景|

因此在只需要保证单变量读写的可见性，变量不参与复合运算时，`volatile`是更好的选择；而在需要原子性的读-改-写操作时，原子类更为合适

#### 原子类与synchronized的比较

原子类与`synchronized`都可以保证可见性和原子性

|特性|原子类|synchronized|
|---|---|---|
|实现机制|基于JVM内置锁(Monitor)|基于CAS(Compare-And-Swap) CPU指令|
|阻塞行为|获取不到锁时会阻塞线程|采用自旋(忙等待)，不阻塞线程|
|锁粒度|方法/代码块级别|单个变量级别|
|性能特征|高竞争下表现更好|低竞争下性能更优|
|功能特性|支持等待/通知机制(wait/notify)|仅提供原子更新功能|

在操作非常简单（如计数器递增），竞争程度低，只需要保护单个变量时有限选择原子类；而在需要保护复杂逻辑或多变量操作的高竞争环境下，选择`synchronized`更为合适