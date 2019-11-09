## InnoDB的快照读，到底和什么相关

InnoDB是非常适合互联网业务的存储引擎，其**多版本并发控制**(Multi Version Concurrency Control, MVCC)，**快照读**(Snapshot Read)机制，能够通过读取**回滚段**(rollback segment)中数据的历史版本，在事务读取记录的时候不用加锁，以支持超高的并发。



**【并发控制，快照读，回滚段】**辅助阅读：

《[InnoDB并发如此高，原因竟然在这？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961444&idx=1&sn=830a93eb74ca484cbcedb06e485f611e&chksm=bd2d0db88a5a84ae5865cd05f8c7899153d16ec7e7976f06033f4fbfbecc2fdee6e8b89bb17b&scene=21#wechat_redirect)》



在**读提交**(Read Committed, RC)，**可重复读**(Repeated Read, RR)两个不同的事务的隔离级别下，快照读的玩法有什么差异，又和什么因素有关呢？



**【事务隔离级别】**辅助阅读：

《[4种事务的隔离级别，如何巧妙实现？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961498&idx=1&sn=058097f882ff9d32f5cdf7922644d083&chksm=bd2d0d468a5a845026b7d2c211330a6bc7e9ebdaa92f8060265f60ca0b166f8957cbf3b0182c&scene=21#wechat_redirect)》



假设有InnoDB表：
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
**提问2**：假设事务的隔离级别是读提交RC，A2, A3, A4又分别读到什么结果集呢？



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



事务的开始时间不一样，会不会影响“快照读”的结果呢？



***case 3***，仍然是并发的事务A与B（A先于B开始，B先于A结束）：

A1: start transaction;
         B1: start transaction;
         B2: insert into t values (4, wangwu);
         B3: commit;
A2: select * from t;



**提问5**：假设事务的隔离级别是可重复读RR，事务A中的A2查询，结果集是什么？

**提问6**：假设事务的隔离级别是读提交RC，A2的结果集又是什么呢？



***case 4***，事务开始的时间再换一下（B先于A开始，B先于A结束）：

​         B1: start transaction;

A1: start transaction;

​         B2: insert into t values (4, wangwu);

​         B3: commit;
A2: select * from t;



**提问7**：假设事务的隔离级别是可重复读RR，事务A中的A2查询，结果集是什么？

**提问8**：假设事务的隔离级别是读提交RC，A2的结果集又是什么呢？



同样是读取历史数据版本，快照读究竟受什么影响呢？是不是很有意思？**答案与原理**明天揭晓。