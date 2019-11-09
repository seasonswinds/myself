## InnoDB，快照读，在RR和RC下有何差异？

为了保证文章知识体系的完整性，先简单解释下**快照读**，**读提交**，**可重复读**。



**快照读**(Snapshot Read)

MySQL数据库，InnoDB存储引擎，为了提高并发，使用MVCC机制，在并发事务时，通过读取数据行的历史数据版本，不加锁，来提高并发的一种不加锁一致性读(Consistent Nonlocking Read)。



**读提交**(Read Committed)

- 数据库领域，事务隔离级别的一种，简称RC
- 它解决“读脏”问题，保证读取到的数据行都是已提交事务写入的
- 它可能存在“读幻影行”问题，同一个事务里，连续相同的read可能读到不同的结果集



**可重复读**(Repeated Read)

- 数据库领域，事务隔离级别的一种，简称RR
- 它不但解决“读脏”问题，还解决了“读幻影行”问题，同一个事务里，连续相同的read读到相同的结果集



在**读提交**(RC)，**可重复读**(RR)两个不同的事务的隔离级别下，**快照读**有什么不同呢？



先说**结论**：

- 事务总能够读取到，自己写入(update /insert /delete)的行记录
- RC下，快照读总是能读到最新的行数据快照，当然，必须是已提交事务写入的
- RR下，某个事务首次read记录的时间为T，未来不会读取到T时间之后已提交事务写入的记录，以保证连续相同的read读到相同的结果集

*画外音：可以看到*

*(1)和并发事务的开始时间没关系，和事务首次read的时间有关；*

*(2)由于不加锁，和互斥关系也不大；*



这些就能解答《[InnoDB的快照读，到底和什么相关？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961511&idx=1&sn=2be06ffcb0335da5bf85f3e648b0fd7e&chksm=bd2d0d7b8a5a846d47e4a3b7f2fd3584f21b4007b31b9d297c960fae9dfb5003e3d9a4c6bb3e&scene=21#wechat_redirect)》中的问题了，InnoDB表：

t(id PK, name);
 
表中有三条记录：
1, shenjian
2, zhangsan
3, lisi



***case 1***，两个并发事务A，B执行的时间序列如下（A先于B开始，B先于A结束）：

A1: start transaction;
         B1: start transaction;
A2: select * from t;
         B2: insert into t values (4, wangwu);
A3: select * from t;
         B3: commit;
A4: select * from t;



**提问1**：假设事务的隔离级别是可重复读RR，事务A中的三次查询，A2, A3, A4分别读到什么结果集？



**回答**：RR下

(1)A2读到的结果集肯定是{1, 2, 3}，这是事务A的第一个read，假设为时间T；

(2)A3读到的结果集也是{1, 2, 3}，因为B还没有提交；

(3)A4读到的结果集还是{1, 2, 3}，因为事务B是在时间T之后提交的，A4得读到和A2一样的记录；


**提问2**：假设事务的隔离级别是读提交RC，A2, A3, A4又分别读到什么结果集呢？



**回答**：RC下

(1)A2读到的结果集是{1, 2, 3}；

(2)A3读到的结果集也是{1, 2, 3}，因为B还没有提交；

(3)A4读到的结果集还是{1, 2, 3, 4}，因为事务B已经提交；



***case 2***，仍然是上面的两个事务，只是A和B开始时间稍有不同（B先于A开始，B先于A结束）：

​         B1: start transaction;

A1: start transaction;

A2: select * from t;
         B2: insert into t values (4, wangwu);
A3: select * from t;
         B3: commit;
A4: select * from t;



**提问3**：假设事务的隔离级别是可重复读RR，事务A中的三次查询，A2, A3, A4分别读到什么结果集？

**提问4**：假设事务的隔离级别是读提交RC，A2, A3, A4的结果集又是什么呢？



**回答**：事务的开始时间不一样，不会影响“快照读”的结果，所以结果集和case 1一样。



***case 3***，仍然是并发的事务A与B（A先于B开始，B先于A结束）：

A1: start transaction;
         B1: start transaction;
         B2: insert into t values (4, wangwu);
         B3: commit;
A2: select * from t;



**提问5**：假设事务的隔离级别是可重复读RR，事务A中的A2查询，结果集是什么？

**提问6**：假设事务的隔离级别是读提交RC，A2的结果集又是什么呢？



**回答**：在RR下，

A2是事务A的第一个read，假设为时间T，它能读取到T之前提交事务写入的数据行，故结果集为{1, 2, 3, 4}。在RC下，没有疑问，一定是{1, 2, 3, 4}。



***case 4***，事务开始的时间再换一下（B先于A开始，B先于A结束）：

​         B1: start transaction;

A1: start transaction;

​         B2: insert into t values (4, wangwu);

​         B3: commit;
A2: select * from t;



**提问7**：假设事务的隔离级别是可重复读RR，事务A中的A2查询，结果集是什么？

**提问8**：假设事务的隔离级别是读提交RC，A2的结果集又是什么呢？



**回答**：事务的开始时间不一样，不会影响“快照读”的结果，所以结果集和case 3一样。



啰嗦说了这么多，用昨天一位网友“山峰”同学的话**总结**：

- RR下，事务在第一个Read操作时，会建立Read View
- RC下，事务在每次Read操作时，都会建立Read View