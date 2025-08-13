## Java 并发编程

### Lecture 7: 生产者-消费者模式

#### wait()/notify() 实现

基础API，无需额外依赖；但需要手动处理同步和唤醒逻辑

```java
public class WaitNotifyExample {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int MAX_SIZE = 10;
    private final Object lock = new Object();

    // 生产者
    public void produce(int item) throws InterruptedException {
        synchronized (lock) {
            while (queue.size() == MAX_SIZE) {
                lock.wait();
            }
            queue.offer(item);
            lock.notifyAll();
        }
    }

    // 消费者
    public int consume() throws InterruptedException {
        synchronized (lock) {
            while (queue.isEmpty()) {
                lock.wait();
            }
            int item = queue.poll();
            lock.notifyAll();
            return item;
        }
    }
}
```

#### BlockingQueue 实现

简单可靠，保证线程安全；但功能较固定

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BlockingQueueExample {
    private final BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);

    // 生产者
    public void produce(int item) throws InterruptedException {
        queue.put(item); // 自动阻塞
    }

    // 消费者
    public int consume() throws InterruptedException {
        return queue.take(); // 自动阻塞
    }
}
```

#### Lock+Condition 实现

更灵活，多条件支持；但代码稍复杂。多用于实现定制的逻辑

```java
import java.util.concurrent.locks.*;

public class LockConditionExample {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int MAX_SIZE = 10;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public void produce(int item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == MAX_SIZE) {
                notFull.await();
            }
            queue.offer(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            int item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

