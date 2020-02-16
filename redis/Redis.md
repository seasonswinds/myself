# Redis

### SDS(Simple dynamic string，简单动态字符串)

- struct

  - sdshdr
    - int len; 字符串长度
    - int free; 未使用的字节数
    - char[] buf; 字符串

- 与C字符串的区别

  - 常数复杂度获取字符串长度

  - 杜绝缓冲区溢出

    > 修改SDS时，先检查SDS的空间是否满足需求
    >
    > 如果不满足，先扩容

  - 减少修改字符串造成的内存重分配次数

    - 空间预分配
      - 修改后，len < 1MB，额外分配和len大小相同的空间
      - 修改后，len >= 1MB，额外分配1MB的空间
    - 惰性空间释放

  - 二进制安全，可以保存二进制数据

  - 兼容部分C字符串函数

###  ListNode - 链表

- struct
  - listNode
    - listNode *prev;
    - listNode *next;
    - void *value;
  - list
    - listNode *head;
    - listNode *tail;
    - unsigned long len;
    - void \*(\*dup)(void *ptr); 节点复制函数
    - void (\*free)(void *ptr); 节点释放函数
    - int (*match)(void *ptr, void *key); 节点值对比函数
- 特性
  - 双端
  - 无环
  - 有表头和表尾指针
  - 有长度计数器
  - 多态

### Dict字典

- struct

  - dicht

    - dictEntry **table; 哈希表数组
    - unsigned long size; 哈希表大小
    - unsigned long sizemask; hash表大小掩码，用于计算索引值，总是等于size-1
    - unsigned long used; 已有节点数量

  - dictEntry

    - void *key;

    - union

      - void *val;
      - uint64_t u64;
      - int64_t s64;

    - dictEntry *next;

      > 使用链地址法解决hash冲突
      >
      > 使用头插法提升插入效率

  - dict

    - dictType *type;

    - void *privdata;

    - dictht ht[2];

      > 一般情况下只是用ht[0];
      >
      > ht[1]只会在rehash时使用

      - rehash

        - 为ht[1]分配空间
          - 扩容，size = 第一个大于等于ht[0].used * 2的$2^n$
          - 缩容，size = 第一个大于等于ht[0].used的$2^n$
        - 对ht[0]中所有的键值对rehash到ht[1]上
        - 释放ht[0]
        - 将ht[1]设置为ht[0]
        - 在ht[1]新建一个空白hash表

      - 渐进式rehash

        - 为ht[1]分配空间

        - rehashidx值设为0，表示rehash正式开始

        - 在rehash期间，每次对字典的操作

          - 同时操作2个hash表。查询时，如果ht[0]未找到，再查询ht[1]

          - 还会将ht[0]在rehashidx索引上的所有键值对rehash到ht[1]，完成后，rehashidx自增1

        - ht[0]的所有键值对都被rehash到ht[1]，将rehashidx设置为-1，表示rehash已完成

      - 扩展与收缩

        - 负载因子

          load_factor = ht[0].used / ht[0].size

        - 扩展

          - 未执行BGSAVE或BGREWRiTEAOF命令 && load_factor >= 1
          - 正在执行BGSAVE或BGREWRiTEAOF命令 && load_factor >= 5

        - 收缩

          - load_factor < 0.1

    - int rehashidx; rehash索引，值为-1时，未执行rehash

  - dictType

    - unsigned int (*hashFunction)(const void *key); 计算hash值

      > Redis使用MurmurHash2算法来计算键的hash值

    - void \*(\*keyDup) (void *privdata, const void *key); 复制键

    - void \*(\*valDup) (void *privdata, const void *obj); 复制值

    - int (* keyCompare) (void *privdata, const void *key1, const void *key2); 对比键

    - void (\*keyDestructor) (void *privdata, void *key); 销毁键

    - void (*valDestructor) (void *privdata, void *obj); 销毁值

### SkipList - 跳表

> 跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找

- struct
  - zskiplistNode
    - zskiplistLevel[] level
      - zskiplistNode *forward; 前进指针
      - unsigned int span; 跨度
    - struct zskiplistNode * backward; 后退指针
    - double score;
    - robj *obj; 成员对象
  - zskiplist
    - structz skiplistNode *header;
    - structz skiplistNode * tail;
    - unsigned long length;
    - int level; 最大层数

### intset - 整数集合

> 集合只包含整数值元素，并且集合元素数不多时
>
> contents数组的真正类型取决于encoding属性的值

- struct

  - intset
    - uint32_t encoding;
    - uint32_t length; 包含的元素数量
    - int8_t[] contents;

- 升级

  > 添加一个新元素时，如果新元素类型比集合中所有元素的类型都要长，需要先进行升级
  >
  > 优点：
  >
  > - 提升整数集合的灵活性
  > - 尽可能的节省内存

  - 根据新元素类型，扩展整数集合数组空间大小
  - 将所有现有元素转换为新类型，并放置在正确的位置上
  - 将新元素添加到底层数组

### ziplist - 压缩列表

### 对象

- struct

  - redisObject
    - unsigned type
    - unsigned encoding
    - void *ptr

- type

  - REDIS_STRING 字符串

    - 整数值，且可以用long表示 —— int

    - 字符串，且长度小于等于39 —— embstr

      > 通过一次内存分配函数，分配一块连续空间

    - 字符串，且长度大于39 —— raw

      > 通过两次内存分配函数

  - REDIS_LIST 列表

  - REDIS_HASH 哈希

  - REDIS_SET 集合

  - REDIS_ZSET 有序集合