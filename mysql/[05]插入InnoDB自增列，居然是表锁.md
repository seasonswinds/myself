## 插入InnoDB自增列，居然是表锁

**一，案例说明**

MySQL，InnoDB，默认的隔离级别(RR)，假设有数据表：

t(id AUTO_INCREMENT, name);

 

数据表中有数据：

1, shenjian

2, zhangsan

3, lisi

 

事务A**先**执行，还**未提交**：

insert into t(name) values(xxx);

 

事务B**后**执行：

insert into t(name) values(ooo);

 

问：事务B会不会被阻塞？

 

**二，案例分析**

InnoDB在RR隔离级别下，能解决幻读问题，上面这个案例中：

(1)事务A先执行insert，会得到一条(4, xxx)的记录，由于是自增列，故不用显示指定id为4，InnoDB会自动增长，注意此时事务并未提交；



(2)事务B后执行insert，**假设不会被阻塞**，那会得到一条(5, ooo)的记录；

 

此时，并未有什么不妥，但如果，

(3)事务A继续insert：

insert into t(name) values(xxoo);

会得到一条(6, xxoo)的记录。

 

(4)事务A再select：

select * from t where id>3;

 

得到的结果是：

4, xxx

6, xxoo

*画外音：不可能查询到5的记录，再RR的隔离级别下，不可能读取到还未提交事务生成的数据。*

 

咦，这对于事务A来说，就很奇怪了，对于AUTO_INCREMENT的列，**连续插入了两条记录**，一条是4，接下来一条变成了6，就像莫名其妙的幻影。

 

**三，自增锁（Auto-inc Locks）**

自增锁是一种特殊的**表级别锁**（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。

*画外音：官网是这么说的*

*An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.* 

 

与此同时，InnoDB提供了innodb_autoinc_lock_mode配置，可以调节与改变该锁的模式与行为。

 

**四，假如不是自增列**

上面的案例，**假设不是自增列**，又会是什么样的情形呢？

t(id unique PK, name);

 

数据表中有数据：

10, shenjian

20, zhangsan

30, lisi

 

事务A**先**执行，在10与20两条记录中插入了一行，还**未提交**：

insert into t values(11, xxx);

 

事务B**后**执行，也在10与20两条记录中插入了一行：

insert into t values(12, ooo);

 

这里，便不再使用自增锁，那：

(1)会使用什么锁？

(2)事务B会不会被阻塞呢？