## 挖坑，InnoDB的七种锁

**Case 1**

MySQL，InnoDB，默认的隔离级别(RR)，假设有数据表：
t(id PK, name);



数据表中有数据：
10, shenjian
20, zhangsan
30, lisi



事务A先执行，还未提交：
insert into t values(**11**, xxx);



事务B后执行：
insert into t values(**12**, ooo);



问：事务B会不会被阻塞？



**Case 2**

MySQL，InnoDB，默认的隔离级别(RR)，假设有数据表：

t(id AUTO_INCREMENT, name);



数据表中有数据：
1, shenjian
2, zhangsan
3, lisi



事务A先执行，还未提交：

insert into t(name) values(xxx);



事务B后执行：
insert into t(name) values(ooo);



问：事务B会不会被阻塞？


本质上，这些都是InnoDB锁机制的问题。总的来说，InnoDB共有**七种类型的锁**：

(1)共享/排它锁(Shared and Exclusive Locks)
(2)意向锁(Intention Locks)
(3)记录锁(Record Locks)
(4)间隙锁(Gap Locks)
(5)临键锁(Next-key Locks)
(6)插入意向锁(Insert Intention Locks)
(7)自增锁(Auto-inc Locks)
