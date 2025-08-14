## Java 并发编程

### Lecture 9: 多线程性能问题

#### 线程管理开销

1. 线程创建/销毁成本
   - 创建代价：每个线程需要分配1MB左右栈内存
   - 销毁代价：系统调用和资源回收
2. 上下文切换开销
   - 保存当前线程状态（寄存器、PC）
   - 恢复目标线程状态
   - 更新CPU缓存和TLB（Translation Lookaside Buffer） 

#### 同步机制开销

1. 锁竞争
   - 阻塞代价：线程从RUNNABLE到BLOCKED状态转换
   - 醒后的缓存失效（Cache Miss）
2. 内存屏障

#### 内存系统开销

1. 缓存一致性协议开销
    - MSEI状态转换
    ```mermaid
        graph LR
        M(Modified) --> S(Shared)
        S --> E(Exclusive)
        E --> M
        S --> I(Invalid)
        I --> E
    ```
    - 总线风暴：多核同时写同一缓存行时产生大量缓存一致性流量
  
2. 伪共享
    ```java
    class Data {
        volatile long x; // 与y在同一缓存行
        volatile long y;
    }
    ```

#### 资源争用问题

1. CPU资源争用
   - 超线程干扰：共享执行单元的硬件线程相互阻塞
   - CPU缓存抖动：频繁切换线程导致缓存命中率下降
  
2. 内存带宽争用
   - 多核带宽竞争：当多个线程密集访问内存时

#### 并发设计缺陷

1. 过度同步
   - 锁范围过大

2. 线程数量不合理
   - CPU密集型任务创建过多线程
   - I/O密集型任务线程不足