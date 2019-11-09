## 4种事务的隔离级别，InnoDB如何巧妙实现

事务ACID特性，其中I代表隔离性(Isolation)。

**什么是事务的隔离性？**

隔离性是指，多个用户的并发事务访问同一个数据库时，一个用户的事务不应该被其他用户的事务干扰，多个并发事务之间要相互隔离。

 

**一个事务怎么会干扰其他事务呢？**

咱们举例子来说明，假设有InnoDB表：

t(id PK, name);

 

表中有三条记录：

1, shenjian

2, zhangsan

3, lisi

 

**case 1**

事务A，先执行，处于未提交的状态：

insert into t values(4, wangwu);

 

事务B，后执行，也未提交：

select * from t;

 

如果事务B能够读取到(4, wangwu)这条记录，事务A就对事务B产生了影响，这个影响叫做“读脏”，读到了未提交事务操作的记录。

 

**case 2**

事务A，先执行：

select * from t where id=1;

 

结果集为：

1, shenjian

 

事务B，后执行，并且提交：

update t set name=xxoo where id=1;

commit;

 

事务A，再次执行相同的查询：

select * from t where id=1;

 

结果集为：

1, xxoo

 

这次是已提交事务B对事务A产生的影响，这个影响叫做“不可重复读”，一个事务内相同的查询，得到了不同的结果。

 

**case 3**

事务A，先执行：

select * from t where id>3;

 

结果集为：

NULL

 

事务B，后执行，并且提交：

insert into t values(4, wangwu);

commit;

 

事务A，首次查询了id>3的结果为NULL，于是想插入一条为4的记录：

insert into t values(4, xxoo);

 

结果集为：

Error : duplicate key!

 

事务A的内心OS是：你TM在逗我，查了id>3为空集，insert id=4告诉我PK冲突？

 

这次是已提交事务B对事务A产生的影响，这个影响叫做“幻读”。

 

可以看到，并发的事务可能导致其他事务：

- 脏读
- 不可重复读
- 幻读

 

**InnoDB实现了哪几种事务的隔离级别？**

按照SQL92标准，InnoDB实现了四种不同事务的隔离级别：

- 读未提交(Read Uncommitted)
- 读提交(Read Committed, RC)
- 可重复读(Repeated Read, RR)
- 串行化(Serializable)

 

不同事务的隔离级别，实际上是一致性与并发性的一个权衡与折衷。

 

**InnoDB的四种事务的隔离级别，分别是怎么实现的？**

InnoDB使用不同的锁策略(Locking Strategy)来实现不同的隔离级别。

 

**一，读未提交(Read Uncommitted)**

这种事务隔离级别下，select语句不加锁。

*画外音：官方的说法是*

*SELECT statements are performed in a nonlocking fashion.*

 

此时，可能读取到不一致的数据，即“脏读”。这是并发最高，一致性最差的隔离级别。

 

**二，串行化(Serializable)**

这种事务的隔离级别下，所有select语句都会被隐式的转化为select ... in share mode.

 

这可能导致，如果有未提交的事务正在修改某些行，所有读取这些行的select都会被阻塞住。

*画外音：官方的说法是*

*To force a plain SELECT to block if other transactions have modified the selected rows.*

 

这是一致性最好的，但并发性最差的隔离级别。

 

在互联网大数据量，高并发量的场景下，几乎**不会使用**上述两种隔离级别。

 

**三，可重复读(Repeated Read, RR)**

这是InnoDB默认的隔离级别，在RR下：



(1)普通的select使用快照读(snapshot read)，这是一种不加锁的一致性读(Consistent Nonlocking Read)，底层使用MVCC来实现，具体的原理在《[InnoDB并发如此高，原因竟然在这？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961444&idx=1&sn=830a93eb74ca484cbcedb06e485f611e&chksm=bd2d0db88a5a84ae5865cd05f8c7899153d16ec7e7976f06033f4fbfbecc2fdee6e8b89bb17b&scene=21#wechat_redirect)》中有详细的描述；

 

(2)加锁的select(select ... in share mode / select ... for update), update, delete等语句，它们的锁，依赖于它们是否在唯一索引(unique index)上使用了唯一的查询条件(unique search condition)，或者范围查询条件(range-type search condition)：

- **在唯一索引上使用唯一的查询条件**，会使用记录锁(record lock)，而不会封锁记录之间的间隔，即不会使用间隙锁(gap lock)与临键锁(next-key lock)
- **范围查询条件**，会使用间隙锁与临键锁，锁住索引记录之间的范围，避免范围间插入记录，以避免产生幻影行记录，以及避免不可重复的读

*画外音：这一段有点绕，多读几遍。*



关于**记录锁，间隙锁，临键锁**的更多说明，详见《[InnoDB，select为啥会阻塞insert？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961471&idx=1&sn=da257b4f77ac464d5119b915b409ba9c&chksm=bd2d0da38a5a84b5fc1417667fe123f2fbd2d7610b89ace8e97e3b9f28b794ad147c1290ceea&scene=21#wechat_redirect)》。

 

**四，读提交(Read Committed, RC)**

这是互联网最常用的隔离级别，在RC下：



(1)普通读是快照读；



(2)加锁的select, update, delete等语句，除了在外键约束检查(foreign-key constraint checking)以及重复键检查(duplicate-key checking)时会封锁区间，其他时刻都只使用记录锁；



此时，其他事务的插入依然可以执行，就可能导致，读取到幻影记录。



**总结**

- 并发事务之间相互干扰，可能导致事务出现读脏，不可重复度，幻读等问题
- InnoDB实现了SQL92标准中的四种隔离级别

(1)**读未提交**：select不加锁，可能出现读脏；

(2)**读提交(RC)**：普通select快照读，锁select /update /delete 会使用记录锁，可能出现不可重复读；

(3)**可重复读(RR)**：普通select快照读，锁select /update /delete 根据查询条件情况，会选择记录锁，或者间隙锁/临键锁，以防止读取到幻影记录；

(4)**串行化**：select隐式转化为select ... in share mode，会被update与delete互斥；

- InnoDB默认的隔离级别是RR，用得最多的隔离级别是RC



或许有朋友问，为啥没提到insert？可以查阅《[InnoDB并发插入，居然使用意向锁？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961461&idx=1&sn=b73293c71d8718256e162be6240797ef&chksm=bd2d0da98a5a84bfe23f0327694dbda2f96677aa91fcfc1c8a5b96c8a6701bccf2995725899a&scene=21#wechat_redirect)》。 