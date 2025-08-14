## Java 并发编程

### Lecture 8: 三类线程安全问题

线程安全问题主要分为三类：原子性问题（Atomicity）、可见性问题（Visibility）、有序性问题（Ordering）。

#### 原子性问题

原子性问题是由于复合操作的非原子执行导致的数据不一致。表现包括：
- 竞态条件：多个线程同时访问共享资源，导致数据不一致。
- 读-改-写操作的非原子性（如i++）
- 检查后执行的非原子性

解决方案：
- 使用synchronized关键字，将复合操作封装在同步块中，确保操作的原子性。
- 使用原子类（如AtomicInteger、AtomicBoolean等），它们提供了原子操作的方法，可以避免复合操作的线程安全问题。
- 使用Lock类（如ReentrantLock、ReadWriteLock等），它们提供了更灵活的锁机制，可以确保操作的原子性。

#### 可见性问题

可见性问题是指一个线程对共享变量的修改，其他线程无法立即看到。根本原因包括：
- CPU多级缓存架构：每个CPU都有自己的缓存，线程对共享变量的修改可能只发生在自己的缓存中，而其他线程无法立即看到。
- 指令重排序优化：编译器和CPU可能会对指令进行重排序
- JIT编译器优化：JIT编译器可能会对代码进行优化，导致变量的修改无法立即反映在内存中。

解决方案：
- 使用volatile关键字
- 使用synchronized关键字
- 使用final字段（安全发布）
- 使用原子类

#### 有序性问题

有序性问题是指代码执行顺序与预期不符。主要原因包括编译器指令重排序、CPU指令级并行优化、内存系统的重排序

解决方案：
- 使用volatile关键字（禁止指令重排）
- 使用synchronized关键字（建立happens-before关系）
- 使用静态内部类（利用类加载机制）

#### 需要注意线程安全问题的场景

1. 共享数据访问

例如，非原子操作导致计数不准
```java
private int count = 0;
public void increment() {
    count++; // 非原子操作
}
```
解决：使用原子类
```java
private final AtomicInteger count = new AtomicInteger(0);
public void increment() {
    count.incrementAndGet();
}
```

2. 状态依赖

- 检查后执行（Check-then-act）竞态条件
```java
private ExpensiveObject instance;
public ExpensiveObject getInstance() {
    if (instance == null) {          // 检查
        instance = new ExpensiveObject(); // 执行
    }
    return instance;
}
```
解决：双重检查锁定（Double-checked-lock）+volatile
```java
private volatile ExpensiveObject instance;
public ExpensiveObject getInstance() {
    ExpensiveObject result = instance;
    if (result == null) { // 第一次检查
        synchronized (this) {
            result = instance;
            if (result == null) { // 第二次检查
                instance = result = new ExpensiveObject();
            }
        }
    }
    return result;
}
```
使用局部变量result而不直接使用instance进行判断，减少对volatile变量的访问次数，提高性能（volatile变量需要保证内存可见性，防止指令重排，因此访问速度较慢），并减少线程间的竞争。

第一次检查避免不必要的同步，如果实例已经存在，那么就无需进入同步代码块，直接返回实例；第二次检查防止多个线程同时通过第一次检查后，重复创建实例，确保只有一个线程能创建实例。

- 状态标志控制

由于可见性问题导致无限循环
```java
private boolean running = true; // 非volatile

public void stop() { running = false; }

public void run() {
    while (running) { /* 工作 */ } // 可能看不到修改
}
```
解决：使用volatile或原子类
```java
private volatile boolean running = true;
// 或
private final AtomicBoolean running = new AtomicBoolean(true);
```
3. 集合操作

共享的集合访问或操作可能导致问题

- 问题：并发修改异常
```java
List<String> list = new ArrayList<>();
// 线程A
list.add("item");
// 线程B
for (String s : list) { ... } // 可能抛出ConcurrentModificationException
```
解决：使用并发集合
```java
List<String> list = new CopyOnWriteArrayList<>(); // 或
Map<String, String> map = new ConcurrentHashMap<>();
```

- 问题：非原子复合操作
```java
if (!map.containsKey(key)) {
    map.put(key, value); // 仍然可能产生竞态条件
}
```
解决：使用原子类
```java
map.putIfAbsent(key, value); // ConcurrentHashMap的方法
```

4. 资源管理

- 缓存击穿时重复初始化
```java
private Map<K,V> cache = new HashMap<>();

public V get(K key) {
    V value = cache.get(key);
    if (value == null) {
        value = computeValue(key); // 昂贵计算
        cache.put(key, value);     // 可能被多次执行
    }
    return value;
}
```
解决：使用ConcurrentHashMap
```java
private final ConcurrentMap<K,V> cache = new ConcurrentHashMap<>();

public V get(K key) {
    return cache.computeIfAbsent(key, this::computeValue);
}
```

- 连接池/对象池资源重复分配或泄露
```java
private List<Connection> pool = new ArrayList<>();

public Connection getConnection() {
    if (pool.isEmpty()) {          // 检查
        return createConnection();  // 执行
    }
    return pool.remove(0);         // 可能抛出异常
}
```
解决：使用同步控制
```java
private final BlockingQueue<Connection> pool = new LinkedBlockingQueue<>();

public Connection getConnection() throws InterruptedException {
    Connection conn = pool.poll();
    return conn != null ? conn : createConnection();
}
```