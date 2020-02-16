# JVM锁

### Synchronized实现原理

- 每个对象维护一个被锁次数计数器
  - 未被锁定，值为0；
  - 当一个线程获得锁，自增1；
  - 同一个线程再次获得该对象的锁时，再次自增1；
  - 同一个线程释放锁时，自减1；
  - 计数器为0时，锁释放，其他线程可以获得锁；

- 同步方法，使用ACC_SYNCHRONIZED标记
- 同步代码块，使用monitorenter、monitorexit两个指令

- ACC_SYNCHRONIZED、monitorenter和monitorexit都是基于Monitor实现
- Monitor由C++实现

### Synchronized使用场景有哪些？

![synchronized使用场景](https://wx2.sinaimg.cn/mw1024/ce7c2972ly1g94ayhgjqaj20rl0gajrn.jpg)

> 构造函数不能使用synchronized关键字，但可以在方法内部使用代码块同步

### Synchronized特性

- 原子性

  > monitorenter和monitorexit指令可以保证被synchronized关键字修饰的代码在同一时间只能被一个线程访问，锁释放前无法被其他线程访问到

- 可见性

  > 对一个变量解锁之前，必须先把此变量同步回主存中

- 有序性

  > 一个unlock操作先行发生(happen-before)于后面对同一个锁的lock操作

### Monitor

- Hoare Monitor(阻塞式条件变量)
  
  <img src="https://wx2.sinaimg.cn/mw1024/ce7c2972gy1g9s211struj20m80nkq3x.jpg" alt="Hoare Monitor" style="zoom: 33%;" />
  
  - 发出通知(signaling)的线程必须等待被通知(signaled)的线程放弃占用管程
  - 每个管程对象有两个线程队列
    - e是入口队列
    - s是已经发出通知的线程队列
  - 对于每个条件变量c，有一个线程队列
  - 发出通知的线程转入等待，但会比在线程入口的队列有更高优先权被调度
  
- Mesa Monitor(非阻塞式条件变量)
  
  <img src="https://wx3.sinaimg.cn/mw1024/ce7c2972gy1g9s230plsmj20m80uc0tz.jpg" alt="Mesa Monitor" style="zoom:33%;" />
  
  - 发出通知的线程并不会失去管程的占用权
  - 被通知的线程将会被移入管程入口

### Java Monitor

- Monitor对象存在于每个Java对象的对象头Mark World中
- ObjectMonitor关键属性
  - _owner：指向持有ObjectMonitor对象的线程
  - _WaitSet：存放处于wait状态的线程队列
  - _EntryList：存放处于等待锁block状态的线程队列
  - _recursions：锁的重入次数
  - _count：用来记录该线程获取锁的次数

![Java Monitor](https://wx4.sinaimg.cn/mw1024/ce7c2972ly1g93fc8nzcfj20et08vwf1.jpg)

- 当多个线程同时访问一段同步代码时
  - 首先进入_EntryList
  - 当某个线程获取到对象的monitor后进入\_owner区域
    - 把monitor中的\_owner变量设置为当前线程
    - 计数器\_count加1
  - 若持有monitor的线程调用wait()方法，将释放当前持有的monitor
    - _owner变量变更为null
    - _count减1
    - 该线程进入_WaitSet等待被唤醒
  - 若当前线程执行完毕也将释放monitor，并复位变量的值
  - 当一个线程释放监视器时，在入口区和等待区的等待线程都会去竞争monitor

### Monitor Object模式特性

- 互斥
- 信号机制

### Monitor Object模式参与者

- 监视器对象(Monitor Object)
- 同步方法(Sync Method)
- 监视锁(Monitor Lock)
- 监视条件(Monitor Condition)

### JVM锁优化

- 自旋锁、适应性自旋锁

- 锁粗化

- 锁消除

- 偏向锁
  
  > 偏向锁是为了消除无竞争情况下的同步原语
  
  - 获取
    - 对象Mark Word里存储了当前线程ID
      - 如果是，表示线程已经获得了锁
      - 如果不是，Mark Word中的偏向锁标识是否为1
        - 如果未设置，使用CAS竞争锁(偏向当前线程)
        - 如果已经设置，尝试使用CAS将锁偏向当前线程
  - 释放，等到竞争出现才会释放
    - 通过CAS竞争锁失败，证明当前存在多线程竞争情况
    - 当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁
    - 然后被阻塞在安全点的线程继续往下执行同步代码块
  
- 轻量级锁

  > 使用CAS取代互斥同步

  - 获取
    - 在线程栈帧中创建Lock Record
    - 将锁对象的Mark Word拷贝至Lock Record
    - 通过CAS操作将锁对象的Mark Word指向该锁记录
      - 若CAS操作成功，则轻量级锁的上锁过程成功；
      - 若CAS操作失败，判断当前线程是否已经持有了该轻量级锁
        - 若已持有，直接进入同步块
        - 若未持有，表示该锁已经被其他线程占用，膨胀为重量级锁
  - 释放
    - 通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word
      - 如果替换成功，锁释放
      - 如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。

### CAS有哪些问题？

- ABA问题，通过版本号解决
- 自旋时间过长，竞争激烈时不适合
- 只能保证一个变量的原子操作

### ReentrantLock

- 公平锁

- 非公平锁(默认)

  ```java
  public ReentrantLock() {    sync = new NonfairSync();    }
  public ReentrantLock(boolean fair) {    sync = fair ? new FairSync() : new NonfairSync();    }
  ```

- 持有一个Sync对象，所有操作由Sync对象实现

  <img src="https://wx3.sinaimg.cn/mw1024/ce7c2972ly1g95th9y4m7j20n00kswfq.jpg" alt="ReentrantLock类图" style="zoom:33%;" />

- lock()，非公平锁
  
  > ReentrantLock的非公平锁，仅通过获取锁时，是否竞争来区别，如果线程已经进入阻塞队列，则按顺序唤醒
  
  - compareAndSetState()，尝试CAS竞争锁
    - 成功，setExclusiveOwnerThread()，设置持锁线程
    - 失败，acquire(1)
      - tryAcquire()，再次尝试竞争锁
        - 如果state == 0，即无线程持有锁，尝试竞争锁，如果竞争成功，返回true
        - 如果持锁线程就是当前线程，state+1，返回true
        - 否则返回false
      
      - addWaiter(Node.EXCLUSIVE)，将当前线程插入AQS队尾
      
      - acquireQueued()
        
        - 如果当前线程节点的前驱节点是AQS队列得头结点，即当前线程节点是AQS队列第一个节点
          - 再次尝试竞争锁
        - 阻塞
        
        - 如果成功，selfInterrupt()，当前线程可中断
  
- lock()，公平锁
  
  - acquire(1)，不再使用CAS尝试竞争
    - tryAcquire()，判断是否已经有线程在等待，如果没有才竞争锁
  
- unlock()
  
  - release(1)
    - tryRelease(1)
      - state减1
        - 如果state != 0，返回false
        - 如果state == 0，将占有线程置空，返回true
    - 如果返回true

### LockSupport

- park，将当前线程阻塞
- unpark，将指定线程唤醒

> 与Object的wait/notify机制相比
>
> - 操作对象
>   - 以thread为操作对象更符合阻塞线程的直观定义
>   - 可以精准唤醒某个线程
> - park/unpark设计原理核心是“许可”，park是等待“许可”，unpark是为某线程提供“许可”
>   - unpark可以在park操作之前
>   - “许可”不可叠加

### 什么情况下适合使用ReentrantLock

- 可定时
- 可轮询
- 可中断
- 公平锁
- 非块结构

### 构建非阻塞算法的技巧

- 将执行原子修改的范围缩小到单个变量上

- 即使在包含多个步骤的更新操作中，也要确保数据结构总是处于一致的状态

  > 如果B发现A正在修改数据结构，等待直到A完成操作

- 如果B发现A正在修改数据结构，应该有足够多的信息，是的B能完成A的操作

