# JVM基础

### Java虚拟机发展史

- Sun Classic VM

  > 世界上第一款商用Java虚拟机
  >
  > 纯解释方式执行

- Exact VM

  > Solaris平台
  >
  > 两级即使编译器、编译器与解释器混合工作
  >
  > 准确式内存管理

- Sun HotSpot VM

  > 目前使用最广泛的Java虚拟机
  >
  > 准确式内存管理
  >
  > 热点代码探测技术，通过执行计数器找出最具有编译价值的代码，通知JIT编译器以方法为单位进行编译，分别触发标准编译和栈上替换(OSR)编译动作

- JRockit VM

  > 曾号称世界上速度最快的Java虚拟机
  >
  > 专门为服务器硬件和服务器端应用场景高度优化的虚拟机
  >
  > 不关注启动速度
  >
  > 不包含解析器实现，全部代码都依靠即时编译器编译后执行

### JVM运行时数据区域

- 程序计数器

  > 线程私有
  >
  > 线程执行字节码行号指示器
  >
  > - 如果执行Java方法，记录正在执行的虚拟机字节码指令地址
  > - 如果执行Native方法，值为空(Undefined)

- 虚拟机栈

  > 线程私有
  >
  > 生命周期与线程相同
  >
  > 每个方法执行时都会创建**栈帧**
  >
  > - 栈帧
  >   - 局部变量表
  >   - 操作数栈
  >   - 动态链接
  >   - 方法出口

- 本地方法栈

- 堆

  > 线程共享
  >
  > 虚拟机启动时创建
  >
  > Java堆中可能划分出多个线程私有的分配缓冲区(Thread Local Allocation Buffer, TLAB)

- 方法区

  > 线程共享
  >
  > - 已加载的类信息
  > - 常量
  > - 静态变量
  > - 即时编译器编译后的代码

- 直接内存

### 对象的创建

- 检查new指令参数是否能在常量池中定位到一个类的符号引用

- 检查符号引用代表的类是否已经被加载、解析和初始化

  - 未加载的先执行类加载

- 分配内存

  - 指针碰撞(Bump the Pointer)

    > 堆中内存绝对规整

  - 空闲列表(Free List)

  - 解决并发问题

    - 同步，CAS+失败重试

    - 按照线程划分不同的空间，TLAB

      > 预先为每个线程分配一小块内存(TLAB)，只有TLAB用完并分配新的TLAB时，才同步锁定

- 初始化为0值

- 设置对象头

  - 对象类型
  - 类元数据信息指针
  - 对象hashcode
  - GC分代年龄
  
- 执行构造函数，\<init\>方法

### 对象内存布局

- 对象头(Header)

  - Mark World
    - hashcode
    - GC分代年龄
    - 锁状态标志
    - 线程持有的锁
    - 偏向线程ID
    - 偏向时间戳
  - 类型指针，指向类元数据

- 实例数据(Instance Data)

- 对齐填充(Padding)

  > 对象大小必须是8的整数倍

### 对象访问定位

- 句柄

  > Java堆中划分一块内存作为句柄池
  >
  > reference中存储对象句柄地址
  >
  > 句柄包含了对象实例数据与类型数据的地址信息

  - 对象移动时只会改变句柄中的实例数据指针

- 直接指针

  > reference中直接存储对象地址
  >
  > 对象需保存类型信息

  - 速度更快，节省了一次指针定位的时间开销

### 如何判断对象是否存活？

- 引用计数算法(Reference Counting)
  - 实现简单，判定效率高
  - 很难解决循环引用的问题
- 可达性分析(Reachability Analysis)
  - GC Roots
    - 虚拟机栈中引用的对象
    - 方法区中类静态属性引用的对象
    - 方法区中常量引用的对象
    - 本地方法栈中JNI引用的对象

### 引用

- 强引用(Strong Reference)

  > 只要强引用存在，对象永远不会被回收

- 软引用(Soft Reference)

  > 还有用但非必须的对象
  >
  > 系统将要内存溢出之前，将会把这些对象列入回收范围进行第二次回收

- 弱引用(Weak Reference)

  > 只能生存到下一次垃圾回收之前

- 虚引用(Phantom Reference)

  > 不会影响对象生存时间
  >
  > 无法通过虚引用来取得一个对象实例
  >
  > 唯一目的是能再对象被回收时收到一个系统通知

### 类什么时候可以回收？

- 该类所有实例都已经被回收
- 加载该类的ClassLoader已经被回收
- 该类对应的java.lang.Class对象没有被引用

### 垃圾收集算法

- 标记清除

  > 首先标记所有需要回收的对象，标记完成后统一回收

  - 问题
    - 效率不高
    - 产生内存碎片

- 复制

  > 将可用内存分为两块，一块用完了，将存活对象复制到另一块，然后清空

  - 优点
    - 没有内存碎片

  - 问题
    - 空间浪费

- 标记整理

  > 首先标记所有需要回收的对象，让所有存活对象都向一端移动，然后直接清理边界以外的内存

### CMS垃圾回收器执行过程

- 初始标记

  > Stop the World

- 并发标记

- 重新标记

  > Stop the World

- 并发清除

### 内存分配与回收

- 对象优先在Eden分配

- 大对象直接进入老年代

  > 避免在Eden区及两个Survivor区之间发生大量内存复制

- 长期存活对象进入老年代

- 动态对象年龄判定

  > 如果在Survivor空间中相同年龄所有对象大小总和大于Survivor空间一半，年龄大于等于该年龄的对象直接进入老年代

- 空间分配担保

### 类的生命周期

- 加载(Loading)

  - 通过全限定名获取二进制字节流

    - 从ZIP包中获取，jar、ear、war

    - 从网络获取

    - 运行时计算生成

      > 动态代理

    - 由其他文件生成，JSP

    - 从数据库中读取

  - 将字节流代表的静态存储结构转化为方法区的运行时数据结构

  - 在内存中生成java.lang.Class对象

- 连接(Linking)

  - 验证(Verification)

    > 确保Class文件字节流中包含的信息符合虚拟机要求，并且不会危害虚拟机自身安全

    - 文件格式验证
    - 元数据验证
    - 字节码验证
    - 符号引用验证

  - 准备(Preparation)

    > 为类变量分配内存并设置类变量初始值
    >
    > 在方法区中进行分配

  - 解析(Resolution)

    > 将常量池内的符号引用替换为直接引用
    >
    > - 符号引用(Symbolic References)
    >
    >   > 以一组符号来表述所引用的目标
    >
    > - 直接引用(Direct References)
    >
    >   > 指针、相对偏移量、句柄

    - 类或接口解析
    - 字段解析
    - 类方法解析
    - 接口方法解析

- 初始化(Initialization)

  - 执行类构造器方法，\<cinit\>()
  - 以下情况必须立即对类进行初始化：
    - 遇到new、getstatic、putstatic或invokestatic字节码指令
    - 对类进行反射调用
    - 初始化一个类时，如果其父类还未初始化，需要先触发其父类的初始化
    - 虚拟机启动时，会初始化要执行的主类
    - java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且方法句柄对应的类没有进行过初始化，需要先触发初始化

- 使用(Uing)

- 卸载(Unloading)

### 热点探测(Hot Spot Detection)

- 基于采样的热点探测(Sample Based Hot Spot Detection)

  > 周期性的检查各个线程的栈顶，如果发现某个方法经常出现在栈顶，那这个方法就是热点方法

  - 优点
    - 简单、高效
    - 容易获取方法调用关系

  - 缺点
    - 难以精确确认方法热度，容易因为线程阻塞或外界因素影响而扰乱探测

- 基于计数器的热点探测(Counter Based Hot Spot Detection)

  > 为每个方法，甚至代码块，建立一个计数器，统计方法执行次数，如果超过阈值，就是热点方法

  - 方法调用计数器(Invocation Counter)

  - 回边计数器(Back Edge Counter)

    > 统计一个方法中循环体的执行次数

### 内存间交互操作

- lock

  > 作用于主内存变量，把一个变量标识为一条线成独占的状态

- unlock

  > 作用于主内存变量，释放锁

- read

  > 作用于主内存变量，主内存 => 工作内存

- load

  > 作用于工作内存变量，read操作的值 => 工作内存

- use

  > 作用于工作内存变量，工作内存 => 执行引擎

- assign

  > 作用于工作内存变量，执行引擎 => 工作内存

- store

  > 作用于工作内存变量，工作内存 => 主内存

- write

  > 作用于主内存变量，store操作的值 => 主内存

### 先行发生(happens-before)原则

> Java内存模型中定义的两项操作之间的偏序关系

- 程序次序规则(Program Order Rule)，控制流顺序

- 管程锁定规则(Monitor Lock Rule)

  > unlock操作发生在后面对同一个锁的lock操作之前

- volatile变量规则(Volatile Variable Rule)

- 线程启动规则(Thread Start Rule)

  > 线程的start()方法发生于此线程的每个动作之前

- 线程终止规则(Thread Termination Rule)

- 线程中断规则(Thread Interruption Rule)

  > 线程interrupt()方法的调用，发生在被中断线程的代码检测到中断事件之前

- 对象终结规则(Finalizer Rule)

  > 对象初始化完成发生于finalize()方法之前

- 传递性(Transitivity)

### 线程实现

- 内核线程实现
- 用户线程实现
- 用户线程+轻量级进程实现

### 线程调度

- 协同式线程调度(Cooperative Threads Scheduling)

  > 线程的执行事件由线程本身控制
  >
  > 线程把自己的工作执行完成之后，主动通知系统切换到另外一个线程

  - 优点
    - 实现简单
    - 没有线程同步的问题
  - 缺点
    - 线程执行时间不可控制

- 抢占式线程调度(Preemptive Threads Scheduling)

  > 每个线程由系统分配执行时间

### 线程状态

- 新建(New)
- 运行(Runable)
- 等待
  - 无限期等待(Waiting)
    - Object.wait()
    - Thread.join()
    - LockSupport.park()
  - 限期等待(Timed Waiting)
- 阻塞(Blocked)
- 结束(Terminated)

### Jstack中的线程状态

- **BLOCKED**：阻塞
- **WAITING**：等待
- **TIMED_WAITING**：限时等待
- **Deadlock**：死锁
- **Runnable**：执行中
- **Waiting on condition**：等待条件、谓词
- **Waiting on monitor entry**：等待获取监视器
- **Suspended**：暂停
- **Parked**：停止