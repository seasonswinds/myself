# Dubbo

> Dubbo是一个 **分布式** 、**高性能**的**RPC**服务框架

### SOA(面向服务架构)有哪些特点？

- 明确的协议；
- 明确的接口；
- 合作方式的改变；
- 通信方式(XML、JSON)；

### 微服务架构与SOA有什么区别？

- 目的不同；
- 服务粒度不同；
- 协议不同；

### Dubbo的核心设计原则是什么？

- 微内核
- 插件体系
- 平等对待第三方

### Dubbo的架构

![Dubbo架构图](https://wx1.sinaimg.cn/large/ce7c2972gy1g92m35ubbdj20tg0mwgm5.jpg)

- Provider启动时:

  - 向Registry注册元数据（服务IP、端口）;
  - 订阅配置信息；

- Consumer启动时：

  - 向Registry注册元数据；

  - 订阅Provider元数据

    - 第一次订阅会拉取全量数据；

    - 在订阅的节点上注册watcher；

    - 接收到变更通知，拉取对应节点下全量数据；

      > 当微服务节点较多时会对网络造成压力

  - 订阅路由信息；

  - 订阅配置信息；

- 服务治理中心(dubbo-admin)启动时：

  - 订阅Consumer信息；
  - 订阅Provider信息；
  - 订阅路由信息；
  - 订阅配置信息；

- Registry发生数据变更会推送给订阅的Consumer、服务治理中心；

- Consumer发起RPC调用，调用前后向Monitor上报统计信息（并发数、调用接口）；

### Dubbo有哪些特性？

- 面向接口代理的高性能RPC调用
- 服务自动注册、发现
- 运行期流量调度
- 智能负载均衡、容错
- 高度可扩展
- 可视化的服务治理（依赖分析、调用统计）

### Dubbo总体分层

- 业务层
  - Service

- RPC层
  - Config
  - Proxy，代理，无论Provider还是Consumer，框架都会生成代理类，由代理层执行远程调用；
  - Registry
  - Cluster，集群容错
    - 容错策略(重试、快速失败)；
    - 负载均衡(随机、一致性Hash)
    - 路由策略
  - Monitor
  - Protocol，封装RPC调用具体过程，负责管理Invoker整个生命周期，是Dubbo的核心模型
- Remote层
  - Exchange
  - Transport
  - Serialize

### Dubbo服务的暴露过程

- 启动框架，初始化服务实例
- 通过Proxy组件调用Protocol，把服务端要暴露的接口封装成Invoker
- 转换成Exporter，用于暴露到注册中心的对象，内部属性持有Invoker对象
- 框架打开服务端口，记录服务实例到内存
- 注册到Registry

### Dubbo消费者调用过程

- 从Proxy开始，Proxy持有Invoker，触发invoke调用；

- Invoker调用Cluster，负责集群容错；

- Cluster在调用前通过Directory获取所有可以调用的远程服务Invoker列表；

- 根据路由规则过滤Invoker列表；

- 通过LoadBalance做负载均衡，选出一个可以调用的Invoker；

- 过滤器链，处理上下文、限流、计数等；

- 使用Client做数据传输，例如Netty Client；

- Codec，私有协议构造；

- Serialization序列化；

- ------

  Codec，处理协议头、半包、粘包

- 反序列化

- 分配到线程池，ThreadPool

- Server根据请求查找对应的Exporter（内部持有Invoker）

- Filter，装饰器模式包装了Invoker

- Invoker调用

### 注册中心的作用是什么？

- Provider动态加入；
- Consumer动态发现；
- 参数动态调整；
- 统一配置；

### 使用Zookeeper作为注册中心时的数据结构

- 根节点是Registry分组，值来自用户配置\<dubbo:registry\>中的group属性，默认为dubbo

- 服务接口下包含四类子目录

  - providers

    > 包含多个Provider URL元数据信息

  - consumers

    > 包含多个Consumer URL元数据信息

  - routers

    > 包含多个Consumer路由策略URL元数据信息

  - configurators(动态配置目录)

    > 包含多个动态配置URL元数据信息

### 配置信息优先级

- -D传给JVM参数优先级最高
- 代码或XML配置优先级次高
- 配置文件(properties)优先级最低

### RPC调用流程

> 每个接口都有一个RegistryDirectory实例，负责拉取和订阅服务提供者、动态配置和路由
>
> 路由和负载均衡都是在客户端实现的

- 客户端启动时从注册中心拉取和订阅服务列表

- Cluster把服务列表聚合成一个Invoker

- 通过Directory#list获取providers地址

- 触发路由操作

- 经过负载均衡选出一台机器

- 将请求交给底层I/O线程池处理

  - I/O线程池
  - 业务线程池

- 远程调用

  - Telnet

    > 序列化和反序列化直接使用fastjson处理

    建立TCP连接，传递接口、方法和JSON格式的参数进行服务调用

  - 非Telnet

    根据传递过来的接口、分组和版本信息查找Invoker对应的实例进行反射调用

### Dubbo协议

> Dubbo使用特殊符号0xdabb魔数来分割处理粘包问题

- 协议头(16字节)

  - 魔法数(0xdabb)

    - 0~7

      > 魔数高位，0xda00

    - 8~15

      > 魔数低位，0xbb

  - 数据包类型

    - 16

      > 0 - Response
      >
      > 1 - Request

  - 调用方式

    - 17

      > 0 - 单向调用
      >
      > 1 - 双向调用

  - 事件标识

    - 18

      > 0 - 当前数据包是请求包或者响应包
      >
      > 1 - 当前数据包是心跳包

  - 序列化协议编号

    - 19~23

      > 2 - Hessian2Serialization
      >
      > 3 - JavaSerialization
      >
      > 4 - CompactedJavaSerialization
      >
      > 6 - FastJsonSerialization
      >
      > 7 - NativeJavaSerialization
      >
      > 8 - KryoSerialization
      >
      > 9 - FstSerialization

  - 请求状态

    - 24~31

      > 20 - OK
      >
      > 30 - CLIENT_TIMEOUT
      >
      > 31 - SERVER_TIMEOUT
      >
      > 40 - BAD_REQUEST
      >
      > 50 - BAD_RESPONSE
      >
      > 60 - SERVICE_NOT_FOUND
      >
      > 70 - SERVICE_ERROR - 服务调用错误
      >
      > 80 - SERVER_ERROR - 服务端内部错误
      >
      > 90 - CLIENT_ERROR - 客户端错误
      >
      > 100 - SERVER_THREADPOOL_EXHAUSTED_ERROR - 服务端线程池满，拒绝执行

  - 请求唯一标识

    - 32~95

      > 8个字节

  - 报文体长度

    - 96~127

      > 4个字节

- 客户端请求协议体
  - Dubbo版本号
  - 服务接口名
  - 服务接口版本
  - 方法名
  - 参数类型
  - 方法参数值
  - 请求额外参数(attachment)
- 响应消息体
  - 返回值状态标记
    - 0 - RESPONSE_WITH_EXCEPTION - 异常返回
    - 1 - RESPONSE_VALUE - 正常返回
    - 2 - RESPONSE_NULL_VALUE - 响应空值
    - 3 - RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS - 异常返回包含隐藏参数
    - 4 - RESPONSE_VALUE_WITH_ATTACHMENTS - 响应结果包含隐藏参数
    - 5 - RESPONSE_NULL_VALUE_WITH_ATTACHMENTS - 响应空值包含隐藏参数

### Cluster层核心接口有哪些？

- Cluster，容错接口

  - Failover

    > 请求失败时，重试其他服务器
    >
    > - 可设置重试次数
    > - 下游服务负载高时，容易加重下游服务负载

  - Failfast

    > 快速失败
    >
    > 请求失败时，快速返回异常结果

  - Failsafe

    > 请求失败时，忽略异常

  - Failback

    > 请求失败时，记入失败队列，并由一个定时线程池定时重试

  - Forking

    > 同时调用多个相同服务，只要期中一个返回，立即返回结果
    >
    > - 可配置最大并行数
    > - 适用实时性要求极高的场景
    > - 浪费资源

  - Broadcast

    > 广播调用所有可用服务，任意一个节点报错则报错

  - Mock

    > - 调用失败时返回伪造结果
    > - 直接返回伪造结果，不发起远程调用

  - Available

    > 遍历所有服务列表，找到第一个可用节点，直接请求并返回结果；如果没有可用节点，抛出异常
    >
    > - 不做负载均衡

  - Mergeable

    > 自动把多个节点请求结果合并

- Directory

- Router

- LoadBalance

  - Random LoadBalance，随机

    > 按权重设置随机概率

  - RoundRobin LoadBalance，轮询

    > 按公约后的权重设置轮询比例
    >
    > - 存在慢的提供者累积请求的问题

  - LeastActive LoadBalance，最少活跃调用数

    > 活跃数指调用前后计数差
    >
    > 如果活跃数相同，则随机调用
    >
    > 慢的提供者收到更少的请求

  - ConsistentHash LoadBalance，一致性Hash

    > 相同参数的请求总是发送到同一提供者
    >
    > 某一台服务宕机时，基于虚拟节点，平摊到其他提供者，不会引起剧烈变动
    >
    > - 默认只对第一个参数Hash
    > - 默认使用160份虚拟节点

### Cluster层工作流程

- 生成Invoker对象
- 获得可调用服务列表
  - 前置校验，检查远程服务是否已经被销毁
  - 通过Directory#list方法获取所有可用服务列表
  - 使用Router接口处理服务列表，根据路由规则过滤
  - 返回可调用服务列表
- 负载均衡
- RPC调用