## 别废话，各种SQL到底加了什么锁

原创： 58沈剑 架构师之路  *2018-08-31*

这个月花了一些功夫写InnoDB：并发控制，MVCC，索引，锁... 



有朋友留言：你TM讲了这么多，**锁**分了这么多类型，又和**事务隔离级别**相关，又和**索引**相关，究竟能不能直接告诉我，一个SQL到底加了什么锁！？



我竟无言以对。



好吧，做过简单梳理之后，今天尝试着直接回答，尽量做到不重不漏，各种SQL语句究竟加了什么锁。



**一、普通select**

(1)在读未提交(Read Uncommitted)，读提交(Read Committed, RC)，可重复读(Repeated Read, RR)这三种事务隔离级别下，普通select使用**快照读**(snpashot read)，不加锁，并发非常高；



(2)在串行化(Serializable)这种事务的隔离级别下，普通select会升级为select ... in share mode;



**【快照读】**辅助阅读：

《[InnoDB，并发如此之高的原因](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961444&idx=1&sn=830a93eb74ca484cbcedb06e485f611e&chksm=bd2d0db88a5a84ae5865cd05f8c7899153d16ec7e7976f06033f4fbfbecc2fdee6e8b89bb17b&scene=21#wechat_redirect)》



**【事务隔离级别】**辅助阅读：

《[InnoDB，巧妙的实现四种事务的隔离级别](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961498&idx=1&sn=058097f882ff9d32f5cdf7922644d083&chksm=bd2d0d468a5a845026b7d2c211330a6bc7e9ebdaa92f8060265f60ca0b166f8957cbf3b0182c&scene=21#wechat_redirect)》



**二、加锁select**

加锁select主要是指：

- select ... for update
- select ... in share mode



(1)如果，在唯一索引(unique index)上使用唯一的查询条件(unique search condition)，会使用记录锁(record lock)，而不会封锁记录之间的间隔，即不会使用间隙锁(gap lock)与临键锁(next-key lock)；



**【记录锁，间隙锁，临键锁】**辅助阅读：

《[InnoDB，索引记录上的三种锁](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961471&idx=1&sn=da257b4f77ac464d5119b915b409ba9c&chksm=bd2d0da38a5a84b5fc1417667fe123f2fbd2d7610b89ace8e97e3b9f28b794ad147c1290ceea&scene=21#wechat_redirect)》



举个栗子，假设有InnoDB表：

t(id PK, name);

 

表中有三条记录：

1, shenjian

2, zhangsan

3, lisi



SQL语句：

select * from t where id=1 for update;

只会封锁记录，而不会封锁区间。



(2)其他的查询条件和索引条件，InnoDB会封锁被扫描的索引范围，并使用间隙锁与临键锁，避免索引范围区间插入记录；



**三、update与delete**

(1)和加锁select类似，如果在唯一索引上使用唯一的查询条件来update/delete，例如：

update t set name=xxx where id=1;

也只加记录锁；



(2)否则，**符合查询条件**的索引记录之前，都会加排他临键锁(exclusive next-key lock)，来封锁索引记录与之前的区间；



(3)尤其需要特殊说明的是，如果update的是聚集索引(clustered index)记录，则对应的普通索引(secondary index)记录也会被隐式加锁，这是由InnoDB索引的实现机制决定的：普通索引存储PK的值，检索普通索引本质上要二次扫描聚集索引。



**【索引底层实现】**辅助阅读：

《[索引，底层是如何实现的？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961486&idx=1&sn=b319a87f87797d5d662ab4715666657f&chksm=bd2d0d528a5a84446fb88da7590e6d4e5ad06cfebb5cb57a83cf75056007ba29515c85b9a24c&scene=21#wechat_redirect)》



**【聚集索引与普通索引的实现差异】**辅助阅读：

《[InnoDB，聚集索引与普通索引有什么不同？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961494&idx=1&sn=34f1874c1e36c2bc8ab9f74af6546ec5&chksm=bd2d0d4a8a5a845c566006efce0831e610604a43279aab03e0a6dde9422b63944e908fcc6c05&scene=21#wechat_redirect)》



**四、insert**

同样是写操作，insert和update与delete不同，它会用排它锁封锁被插入的索引记录，而不会封锁记录之前的范围。



同时，会在插入区间加插入意向锁(insert intention lock)，但这个并不会真正封锁区间，也不会阻止相同区间的不同KEY插入。



**【插入意向锁】**辅助阅读：

《[InnoDB，插入意向锁](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961461&idx=1&sn=b73293c71d8718256e162be6240797ef&chksm=bd2d0da98a5a84bfe23f0327694dbda2f96677aa91fcfc1c8a5b96c8a6701bccf2995725899a&scene=21#wechat_redirect)》



了解不同SQL语句的加锁，对于分析多个事务之间的并发与互斥，以及事务死锁，非常有帮助。



如果还没有厌倦这个话题，后文分解。