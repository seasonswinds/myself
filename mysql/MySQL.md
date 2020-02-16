# MySQL

### ACID特性

- ##### A(Atomicity) 原子性  

    一个事务必须被视为一个不可分割的最小工作单元，整个事务中的所有操作要么全部提交成功，要么全部失败回滚。

- ##### C(Consistency)一致性

    数据库总是从一个一致的状态转换到另一个一致的状态

- ##### I(Isolation)隔离性

    一个事务所做的修改在最终提交以前，对其他事务的可见属性
    
- ##### D(Durability) 持久性

    一旦事务提交，则其所做的修改就会永久保存到数据库中

### 事务隔离级别

- ##### Read Uncommitted(读未提交)
- ##### Read committed(读已提交)
- ##### Read Repeatable(可重复读)
- ##### Serializable(序列化)

### Redo Log

> 用来实现事务的持久性

- 内存中的重做日志缓冲（redo log buffer）
- 重做日志文件（redo log file）

#####  参数innodb_flush_log_at_trx_commit

![redo log 三层架构](https://wx2.sinaimg.cn/mw1024/ce7c2972ly1g86rxybtzgj20an081753.jpg)

- 0：表示事务提交时不进行写redo log file的操作，只记录在Log Buffer中。每隔一秒，才将Log Buffer中的数据批量write入OS cache，同时MySQL主动fsync。这个操作仅在master thread中完成（master thread每隔1秒进行一次fsync操作）

  > 性能最佳。但是如果数据库崩溃，有一秒的数据丢失。

- 1：每次事务提交，都将Log Buffer中的数据write入OS cache，同时MySQL主动fsync。

  > 强一致。InnoDB的默认设置。保证事务ACID特性。

- 2：表示事务提交时将redo log写入文件，不过仅写入文件系统的缓存中，不进行fsync操作。每次事务提交，都将Log Buffer中的数据write入OS cache；每隔一秒，MySQL主动将OS cache中的数据批量fsync。

  > 折衷。如果操作系统崩溃，最多有一秒的数据丢失。

##### 高并发业务，行业最佳实践，是使用第三种折衷配置（=2），这是因为：

- 配置为2和配置为0，性能差异并不大，因为将数据从Log Buffer拷贝到OS cache，虽然跨越用户态与内核态，但毕竟只是内存的数据拷贝，速度很快；

- 配置为2和配置为0，安全性差异巨大，操作系统崩溃的概率相比MySQL应用程序崩溃的概率，小很多，设置为2，只要操作系统不奔溃，也绝对不会丢数据。

##### 与binlog比较

| redo log                   | binlog                |
| -------------------------- | --------------------- |
| InnoDB引擎产生             | MySQL应用层产生       |
| 物理日志，记录每个页的修改 | 逻辑日志，记录SQL语句 |
| 事务进行中不断写入         | 事务提交后一次性写入  |
| 非顺序                     | 顺序                  |

##### LSN(log sequence number)

> 用于记录日志序号，它是一个不断递增的 unsigned long 类型整数，占用8字节。可以代表以下含义：

- redo log写入的总量。
- checkpoint的位置。checkpoint是redo log中的一个检查点，这个点之前的所有数据都已经刷新回磁盘。
- 页的版本，用来判断是否需要进行恢复操作。

##### crash recovery

> InnoDB引擎在启动时不管上次数据库运行时是否正常关闭，都会进行恢复操作。因为重做日志记录的是物理日志，因此恢复的速度比逻辑日志，如二进制日志要快很多。恢复的时候只需要找到redo log的checkpoint进行恢复即可。

### Undo Log

> 理论上InnoDB最多支持 96 * 1024个普通事务
>
> 用来实现事务的原子性

##### 在InnoDB引擎中，undo log分为：

- insert undo log

> 该undo log可以在事务提交后直接删除，不需要进行purge操作

- update undo log

> delete和update操作产生的undo log
> 对于一条delete语句,不会进行真正的删除，而只是将记录的delete flag设置为1
> update操作是标识旧记录为删除状态，然后新产生一条记录

### MVCC(多版本控制)

##### 事务链表

> 将当前数据库中正在活跃的所有事务(执行begin,但是还没有commit的事务)保存到一个叫trx_sys的事务链表中,事务链表中保存的都是未提交的事务,当事务提交之后会从其中删除

##### ReadView(思考：类似于CopyOnWriteArrayList实现)

> 在事务开始的时候会根据事务链表构造一个ReadView
> 通过该ReadView，新的事务可以根据查询到的所有活跃事务记录的事务ID来匹配能够看见该记录，从而实现数据库的事务隔离

##### 可见性判断逻辑

- 当检索到的数据的事务ID小于事务链表中的最小值(数据行的DB_TRX_ID < m_up_limit_id)表示这个数据在当前事务开启前就已经被其他事务修改过了,所以是可见的。
- 当检索到的数据的事务ID表示的是当前事务自己修改的数据(数据行的DB_TRX_ID = m_creator_trx_id) 时，数据可见。
- 当检索到的数据的事务ID大于事务链表中的最大值(数据行的DB_TRX_ID >= m_low_limit_id) 表示这个数据在当前事务开启后到下一次查询之间又被其他的事务修改过,那么就是不可见的。
- 如果事务链表为空,那么也是可见的,也就是当前事务开始的时候,没有其他任意一个事务在执行。
- 当检索到的数据的事务ID在事务链表中的最小值和最大值之间，从m_low_limit_id到m_up_limit_id进行遍历，取出DB_ROLL_PTR指针所指向的回滚段的事务ID，把它赋值给 trx_id_current ，然后从步骤1重新开始判断，这样总能最后找到一个可用的记录。

##### RC和RR隔离级别ReadView的实现方式

- 在RC事务隔离级别下,每次语句执行都关闭ReadView,然后重新创建一份ReadView。
- 在RR下,事务开始后第一个读操作创建ReadView,一直到事务结束关闭。

### 在InnoDB中只有字段加了索引的，才会是行级锁，否则是表级锁

### 为什么InnoDB能够保证原子性？用的什么方式？

### 为什么InnoDB能够保证一致性？用的什么方式？

### 为什么InnoDB能够保证持久性？用的什么方式？

### 为什么RU级别会发生脏读，而其他的隔离级别能够避免？

### 为什么RC级别不能重复读，而RR级别能够避免？

### 为什么InnoDB的RR级别能够防止幻读？