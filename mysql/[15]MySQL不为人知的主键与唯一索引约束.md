## MySQL不为人知的主键与唯一索引约束

今天和大家简单聊聊MySQL的约束**主键与唯一索引约束**：

PRIMARY KEY and UNIQUE Index Constraints

文章不长，保证有收获。

 

触发约束检测的时机：

- insert
- update

 

当检测到违反约束时，不同存储引擎的处理动作是不一样的。



**如果存储引擎支持事务，SQL会自动****回滚****。**



例子：

create table t1 (

id int(10) primary key

)engine=innodb;

 

insert into t1 values(1);

insert into t1 values(1);

 

其中第二条insert会因为违反约束，而导致回滚。

 

通常可以使用：

show warnings;

来查看违反约束后的错误提示。

 

**如果存储引擎不支持事务，SQL的执行会****中断****，此时可能会导致后续有符合条件的行不被操作，出现不符合预期的结果。**



例子：

create table t2 (

id int(10) unique

)engine=MyISAM;

 

insert into t2 values(1);

insert into t2 values(5);

insert into t2 values(6);

insert into t2 values(10);

 

update t2 set id=id+1;

 

**update执行后，猜猜会得到什么结果集？**

猜想一：2, 6, 7, 11

猜想二：1, 5, 6, 10

.

.

.

都不对，正确答案是：2, 5, 6, 10

 

第一行id=1，加1后，没有违反unique约束，执行成功；

第二行id=5，加1后，由于id=6的记录存在，违反uinique约束，SQL终止，修改失败；

第三行id=6，第四行id=10便不再执行；

*画外音：这太操蛋了，一个update语句，部分执行成功，部分执行失败。*



**为了避免这种情况出现，请使用InnoDB存储引擎**，InnoDB在遇到违反约束时，会自动回滚update语句，一行都不会修改成功。

*画外音：大家把存储引擎换成InnoDB，把上面的例子再跑一遍，印象更加深刻。*

 

另外，对于insert的约束冲突，可以使用：

insert … on duplicate key

指出**在违反主键或唯一索引约束时，需要进行的额外操作**。



例子：

create table t3 (

id int(10) unique,

flag char(10) default 'true'

)engine=MyISAM;

 

insert into t3(id) values(1);

insert into t3(id) values(5);

insert into t3(id) values(6);

insert into t3(id) values(10);

 

insert into t3(id) values(10) on duplicate key update flag='false';



**insert执行后，猜猜会发生什么？**

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

插入id=10的记录，会违反unique约束，此时执行update flag=’false’，于是有一行记录被update了。

 

这**相当于执行**：

update t3 set flag='false' where id=10;

 

仔细看，insert的结果返回，提示：

Query OK, 2 rows affected

有意思么？

*画外音：本文所有实验，基于MySQL5.6。*

 

**总结**，对于主键与唯一索引约束：

- 执行insert和update时，会触发约束检查
- **InnoDB**违反约束时，会回滚对应SQL
- **MyISAM**违反约束时，会中断对应的SQL，可能造成不符合预期的结果集
- 可以使用 insert … on duplicate key 来指定触发约束时的动作
- 通常使用 show warnings; 来查看与调试违反约束的ERROR


互联网大数据量高并发量业务，**为了大家的身心健康，请使用InnoDB**。