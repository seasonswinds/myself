# 缓存模型

### Cache Aside

- 写：

  > 更新DB后，删除缓存中的KEY

- 读：

  > 先读缓存，缓存miss，穿透DB并回写Cache

- 特点：

  > Lazy计算
  >
  > 数据已DB为准，数据一致性强

- 适用场景：

  > 一致性要求高
  >
  > Cache更新逻辑复杂

### Read/Write Through

- 写：

  > Cache miss更新DB；Cache存在，同时更新Cache和DB

- 读：

  > Cache miss后由缓存服务加载并写Cache

- 特点：

  > 存储服务负责数据读写，隔离性更好，对热数据友好

- 适用场景：

  > 数据有冷热分区

### Write behind caching

- 写：

  > 只更新缓存，缓存服务异步更新DB

- 读：

  > Cache miss后由缓存服务加载并写Cache

- 特点：

  > 写性能高
  >
  > 定期异步刷新，随机写变顺序写
  >
  > 存在数据丢失可能

- 适用场景：

  > 写请求高

# 缓存分类

- 本地Cache：进程内缓存
- 进程间Cache：同机独立缓存
- 远程Cache：跨机部署的缓存

