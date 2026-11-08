---
date created: 2024-07-17 13:49
date updated: 2026-10-27T00:37
---

# Redis

## Redis 基本概念

### 关系型数据库和非关系型数据库

|          | 关系型数据库                                           | 非关系型数据库                                         |
|----------|--------------------------------------------------------|--------------------------------------------------------|
| 数据模型 | 数据以行列形式组织在表中（表格）                       | 通常使用键值对、文档、列族、图数据结构等非表格数据模型 |
| 查询语言 | SQL                                                    | 特定 API 或自定义查询语言                              |
| 事务处理 | 支持事务                                               | 牺牲事务的 ACID 属性（提高性能和扩展性）               |
| 灵活性   | 一般                                                   | 灵活                                                   |
| 适用场景 | 高度组织和结构化数据管理的场景，如金融、会计、库存管理 | 大数据应用、实时网页应用、日志记录系统                 |

### 什么是 Redis

- 是支持键值对等多种数据结构的、基于内存的、可持久化的存储系统
- 可用于缓存、事件发布或订阅、高速队列等场景
- 支持字符串、哈希表、列表、集合、有序集合等数据结构

### Redis 有哪些特点

- 丰富的数据类型，字符串、列表、哈希表、集合、有序集合、地理空间 (Geo)、发布和订阅 (Publish/Subscribe)、流 (Stream)
- 内存存储，读写数据快
- 可持久化，可将内存中的数据保存在硬盘中，方便备份

### Redis 数据结构

- String
	- 是二进制安全的
	- 可以包含任何数据，如图片或序列化对象
	- 最大存储 `512MB` 大小
- Hash
	- 键值对集合
	- 适合存储对象
- List
	- 字符串列表，按照插入顺序排序
	- 可从头部或尾部添加元素
- Set
	- 字符串类型的无序集合
	- 通过哈希表实现，所以添加、删除、查找的复杂度都是 `O(1)`
- Sorted Set (`ZSET`)
	- 与普通集合不同的是，每个元素有一个 `Double` 类型的分数
	- 使用分数升序排序
	- 成员要唯一，分数可重复

### Redis 每种数据结构使用场景

- String
	- 信息缓存（频繁读取，且不常修改，如用户信息和视频信息，缓存 Session）
	- 计数器（记录用户访问次数，商品浏览次数，限定 IP 访问次数）
	- 分布式锁
- Hash
	- 信息缓存（购物车使用以用户 ID 为键的哈希表存储商品 ID 及其购买数量）
- List
	- 有序的，元素可重复的队列（定时排行榜，播放记录）
- Set
	- 无序的，元素不可重复（赞过的、喜欢的歌单）
- Sorted Set (`ZSet`)
	- 有序的，元素不可重复，升序排列（实时排行榜）

| | String | Hash |
| --- | --- | --- |
| 效率 | 很高 | 高 |
| 容量 | 低 | 低 |
| 灵活性 | 低 | 高 |
| 序列化 | 简单 | 复杂 |

### Redis 数据结构命令

- String

```bash
# SET 设置键值
SET mykey "Hello, World!"

# GET 获取键值
GET mykey

# GETSET 设置键值，并返回旧值
GETSET mykey "New Value"

# MSET 一次性设置多个键值
MSET key1 "value1" key2 "value2" key3 "value3"

# MGET 一次性获取多个键的值
MGET key1 key2 key3

# SETNX 仅当键不存在时设置键值
SETNX mynewkey "This will be set only if mynewkey does not exist"

# SETEX 设置键值，并设置过期时间（单位：秒）
SETEX mykeywithexpiry 60 "This value will expire after 60 seconds"

# PSETEX 设置键值，并设置过期时间（单位：毫秒）
PSETEX mykeywithmillisecondsexpiry 60000 "This value will expire after 60000 milliseconds"

# INCR 对键值进行递增操作
INCR counter

# DECR 对键值进行递减操作
DECR counter

# INCRBY 对键值进行指定数量的递增操作
INCRBY counter 10

# DECRBY 对键值进行指定数量的递减操作
DECRBY counter 5

# APPEND 向键值的末尾追加字符串
APPEND mykey " Appended String"

# STRLEN 获取键值的长度
STRLEN mykey

# SETRANGE 用指定的字符串覆盖键值的一部分
SETRANGE mykey 6 "Redis"

# GETRANGE 获取键值的一部分
GETRANGE mykey 0 5

# SETBIT 设置键值指定位的值，只能为 0 或 1
SETBIT mykey 10 1

# BITCOUNT 统计键值中设置为 1 的位数
BITCOUNT mykey

# BITOP 对多个键值进行位运算，并存储结果
BITOP AND resultkey mykey1 mykey2

# BITPOS 在键值中查找第一个设置为1或0的位的位置
BITPOS mykey 1
```

- Hash

```bash
# HSET 设置哈希表中字段的值
HSET myhash field1 "value1"

# HGET 获取哈希表中字段的值
HGET myhash field1

# HMSET 一次性设置多个字段的值
HMSET myhash field1 "value1" field2 "value2" field3 "value3"

# HMGET 一次性获取多个字段的值
HMGET myhash field1 field2 field3

# HGETALL 获取哈希表中所有的字段和值
HGETALL myhash

# HKEYS 获取哈希表中的所有字段名
HKEYS myhash

# HVALS 获取哈希表中的所有值
HVALS myhash

# HLEN 获取哈希表中字段的数量
HLEN myhash

# HEXISTS 检查哈希表中是否存在指定的字段
HEXISTS myhash field1

# HDEL 删除哈希表中的一个或多个字段
HDEL myhash field1 field2

# HINCRBY 对哈希表中的整数字段进行增加
HINCRBY myhash field1 10

# HINCRBYFLOAT 对哈希表中的浮点数字段进行增加
HINCRBYFLOAT myhash field2 3.5

# HSCAN 增量迭代哈希表中的键值对
HSCAN myhash 0 MATCH *field* COUNT 10
```

- List

```bash
# LPUSH 将一个或多个值插入到列表的头部
LPUSH mylist "value1"
LPUSH mylist "value2" "value3"

# RPUSH 将一个或多个值插入到列表的尾部
RPUSH mylist "value4"

# LPOP 从列表的头部移除并返回第一个元素
LPOP mylist

# RPOP 从列表的尾部移除并返回最后一个元素
RPOP mylist

# BLPOP 从列表头部移除并返回第一个元素，如果列表为空，阻塞直到有可用的元素或超时
BLPOP mylist 30  # 等待30秒直到列表不为空

# BRPOP 从列表尾部移除并返回最后一个元素，如果列表为空，阻塞直到有可用的元素或超时
BRPOP mylist 30  # 等待30秒直到列表不为空

# RPOPLPUSH 移除列表的最后一个元素，并将该元素添加到另一个列表并返回它
RPOPLPUSH mylist anotherlist

# BRPOPLPUSH 从列表尾部移除最后一个元素，将其添加到另一个列表并返回它，如果列表为空，阻塞直到有可用的元素或超时
BRPOPLPUSH mylist anotherlist 30  # 等待30秒直到列表不为空

# LRANGE 获取列表中的一段元素
LRANGE mylist 0 -1  # 获取整个列表的元素
LRANGE mylist 0 2   # 获取列表中的前三个元素

# LINSERT 在列表的元素前或后插入元素
LINSERT mylist BEFORE "value3" "new_value"  # 在 "value3" 之前插入 "new_value"
LINSERT mylist AFTER "value3" "new_value"   # 在 "value3" 之后插入 "new_value"

# LSET 通过索引设置列表元素的值
LSET mylist 0 "new_value"  # 将列表中索引为0的元素设置为 "new_value"

# LREM 根据参数 COUNT 的值移除列表中与参数 VALUE 相等的元素
LREM mylist 1 "value2"  # 移除列表中的一个 "value2"

# LINDEX 通过索引获取列表中的元素
LINDEX mylist 1  # 获取列表中索引为1的元素

# LLEN 获取列表的长度
LLEN mylist

# LTRIM 对一个列表进行修剪，只保留指定区间内的元素
LTRIM mylist 1 -1  # 只保留列表中索引从1到最后的元素
```

- Set

```bash
# SADD 向集合中添加一个或多个成员
SADD myset "member1" "member2" "member3"

# SREM 移除集合中的一个或多个成员
SREM myset "member1"

# SMOVE 将一个成员从一个集合移动到另一个集合
SMOVE myset newset "member2"

# SPOP 随机移除并返回集合中的一个成员
SPOP myset

# SCARD 获取集合的成员数量
SCARD myset

# SSCAN
SSCAN myset

# SMEMBERS 返回集合中的所有成员
SMEMBERS myset

# SISMEMBER 判断成员是否存在于集合中
SISMEMBER myset "member2"

# SMISMEMBER 判断多个成员是否存在于集合中，并返回一个数组，数组的每个元素表示对应的成员是否存在
SMISMEMBER myset "member1" "member2" "nonexistent"

# SRANDMEMBER 随机返回集合中的一个成员，可以指定返回数量
SRANDMEMBER myset
SRANDMEMBER myset 2  # 随机返回两个成员

# SINTER 返回多个集合的交集
SINTER set1 set2

# SINTERSTORE 将多个集合的交集存储到新的集合中
SINTERSTORE resultset set1 set2

# SUNION 返回多个集合的并集
SUNION set1 set2

# SUNIONSTORE 将多个集合的并集存储到新的集合中
SUNIONSTORE resultset set1 set2

# SDIFF 返回多个集合的差集（第一个集合有而其他集合没有的成员）
SDIFF set1 set2

# SDIFFSTORE 将多个集合的差集存储到新的集合中
SDIFFSTORE resultset set1 set2
```

- Sorted Set (`ZSET`)

```bash
# ZADD 向有序集合添加一个或多个成员，或者更新已存在成员的分数
ZADD myzset 1 "member1" 2 "member2" 3 "member3"

# ZREM 移除有序集合中的一个或多个成员
ZREM myzset "member1"

# ZCOUNT 计算有序集合中指定分数区间的成员数量
ZCOUNT myzset 1 3

# ZCARD 获取有序集合的成员数量
ZCARD myzset

# ZSCORE 返回有序集合中成员的分数
ZSCORE myzset "member2"

# ZINCRBY 为有序集合中的成员增加分数
ZINCRBY myzset 1.5 "member2"

# ZRANGE 根据索引区间返回有序集合中的成员
ZRANGE myzset 0 -1 WITHSCORES  # 返回所有成员及其分数

# ZRANGEBYSCORE 根据分数区间返回有序集合中的成员
ZRANGEBYSCORE myzset 1 3 WITHSCORES  # 返回分数在1到3之间的成员及其分数

# ZREVRANGE 根据索引区间返回有序集合中的成员，分数从高到低
ZREVRANGE myzset 0 -1 WITHSCORES  # 返回所有成员及其分数，分数从高到低

# ZREVRANGEBYSCORE 根据分数区间返回有序集合中的成员，分数从高到低
ZREVRANGEBYSCORE myzset 3 1 WITHSCORES  # 返回分数在3到1之间的成员及其分数，分数从高到低

# ZRANK 返回有序集合中成员的索引
ZRANK myzset "member2"

# ZREVRANK 返回有序集合中成员的索引，分数从高到低
ZREVRANK myzset "member2"

# ZUNIONSTORE 计算多个有序集合的并集，并将结果存储在一个新的有序集合中
ZUNIONSTORE resultzset 2 myzset anotherzset AGGREGATE SUM  # 使用SUM聚合方式

# ZINTERSTORE 计算多个有序集合的交集，并将结果存储在一个新的有序集合中
ZINTERSTORE resultzset 2 myzset anotherzset AGGREGATE SUM  # 使用SUM聚合方式

# ZDIFFSTORE 计算多个有序集合的差集，并将结果存储在一个新的有序集合中
ZDIFFSTORE resultzset 2 myzset anotherzset

# ZPOPMAX 移除并返回有序集合中的最高分数成员
ZPOPMAX myzset

# ZPOPMIN 移除并返回有序集合中的最低分数成员
ZPOPMIN myzset

# BZPOPMAX 阻塞版本的ZPOPMAX，当有序集合为空时阻塞直到有成员可弹出或超时
BZPOPMAX myzset anotherzset 30  # 阻塞30秒

# BZPOPMIN 阻塞版本的ZPOPMIN，当有序集合为空时阻塞直到有成员可弹出或超时
BZPOPMIN myzset anotherzset 30  # 阻塞30秒

# ZSCAN 增量迭代有序集合中的元素和分数
ZSCAN myzset 0 MATCH *member* COUNT 10
```

### Redis 其他命令

```bash
# 删除键值对
DEL key

# 检查键是否存在
EXISTS key

# 设置键的超时时间
EXPIRE key seconds

# 重命名键
RENAME key newkey

# 随机获取键
RANDOMKEY

# 获取键的类型
TYPE key

# 列出所有键
KEYS *

# 分批获取键
SCAN cursor [MATCH pattern] [COUNT count]

# 保存
SAVE

# 清空当前数据库中的所有键值对
FLUSHDB
```

> `KEYS` 命令进行全键空间扫描
> `SAVE` 和 `FLUSHDB` 都是同步操作，会阻塞 Redis 服务器
> `SCAN` 指令可以无阻塞地提取出指定模式的 Key 列表，但是会有重复的概率，需要去重，整体所花费的时间会比 `KEYS` 久

### Redis-py 每种数据结构的迭代器

```python
import redis


pool = redis.ConnectionPool(host="localhost", port=6379, decode_responses=True)
client = redis.Redis(connection_pool=pool)

# 设置 transaction=False 可以禁用 pipeline 的 MULTI/EXEC 包裹
pipe = client.pipeline(transaction=True)
pipe.set("name", "jack")
pipe.set("role", "admin")
pipe.sadd("tags", "python", "redis")
pipe.incr("num")    # 不存在则 value 为 1，存在则 value 自增
pipe.execute()

print(client.get("name"))
print(client.get("role"))
print(client.get("num"))


def list_iter(name):
    list_count = client.llen(name)
    for index in range(list_count):
        yield client.lindex(name, index)


for key in client.scan_iter(match="user:*", count=100):
    print(key)

for item in client.hscan_iter("hash1"):
    print(item)

for member in client.sscan_iter("set3"):
    print(member)

for member, score in client.zscan_iter("zset3"):
    print(member, score)
```

### Redis-py 的 `set()` 函数

- `ex`，过期时间（秒）
- `px`，过期时间（毫秒）
- `nx`，如果设置为 `True`，则只有 Key 不存在时，操作才执行
- `xx`，如果设置为 `True`，则只有 Key 存在时，操作才执行

## Redis 持久化

### 什么是持久化

- 持久化，将数据永久保存在存储设备中
	- 应用层，关闭应用数据还在
	- 系统层，关闭系统数据还在

### Redis 为什么需要持久化

- Redis 属于内存数据库，关闭系统后数据会丢失，需要将数据持久化

### Redis 如何持久化

- RDB
	- 在指定的时间间隔对数据进行快照存储
- AOF
	- AOF 会记录每个写操作命令到日志文件中
	- 当 Redis 重启时，会通过重新执行 AOF 文件中的命令来恢复数据
- RDB+AOF
	- 优先使用 AOF
	- 通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整
- 不采用持久化
	- 只希望数据在服务器运行时存在，则使用该方式

### RDB

- 定义
	- 把当前内存数据生成快照保存到硬盘的过程
- 手动触发
	- 对应 `save` 命令，阻塞 Redis 服务器，直到 RDB 完成
	- 对于内存较大的实例会造成长时间阻塞，线上环境不宜使用
- 自动触发
		- 对应 `bgsave` 命令，Redis 进程执行 Fork 操作创建子进程，RDB 再由子进程负责，完成后自动结束
	- 阻塞只发生在 Fork 阶段，时间短

```bash
# redis.conf
# 几秒内数据修改了几次自动触发 bgsave
SAVE <SECONDS> <CHANGES>

# 关闭自动触发
SAVE ""
```

> 其他常见操作也会触发 `bgsave`
> 从节点执行全量复制操作，主节点自动执行 `bgsave` 生成 RDB 文件发送给从节点
> 默认情况执行 `SHUTDOWN` 命令时，如果没有开启 AOF，则自动执行 `bgsave`

### `bgsave` 工作流程

- 执行 `bgsave` 命令，Redis 父进程判断当前是否存在正在执行的子进程（RDB/AOF 子进程），如果存在则直接返回
- 父进程创建子进程，Fork 时父进程会阻塞
- Fork 完成后不再阻塞父进程
- 子进程创建 RDB 文件，根据父进程内存生成临时快照文件，完成后对原有文件进行原子替换
- 子进程发送信号给父进程表示完成，父进程更新统计信息

### AOF

- 以独立日志的方式记录每次写命令，重启时再执行 AOF 文件中的命令恢复数据，主要作用是解决数据持久化的实时性，是 Redis 持久化的主流方式
- 在 `appendonly.aof` 文件中配置 `appendonly yes`，默认不开启

### AOF 工作流程

- 命令写入，写入命令会追加到 `aof_buf` 缓冲区
- 文件同步，缓冲区与硬盘同步
- 文件重写，AOF 文件持续变大，需要进行重写达到压缩的目的
- 重启加载，Redis 服务器重启后，加载 AOF 文件进行数据恢复

> Redis 是单线程响应命令，如果将写文件命令直接追加到硬盘中，则性能完全取决于硬盘负载，所以先将写命令追加到 `aof_buf` 缓冲区中

### AOF 重写机制

- 目的
	- 压缩 AOF 文件（删除无效语句以保留最终数据的写入命令，合并命令）
	- 更小的文件可以被更快地加载
- 手动触发
	- 调用 `bgrewriteaof` 命令
- 自动触发
	- 根据 `auto-aof-rewrite-min-size` 和 `auto-aof-rewrite-percentage` 参数确定自动触发时机

> `auto-aof-rewrite-min-size`，表示运行 AOF 重写时文件最小体积，默认为 `64 MB`
> `auto-aof-rewrite-percentage`，代表当前 AOF 文件空间 (`aof_current_size`) 和上一次重写后 AOF 文件空间 (`aof_base_size`) 的比值

### Redis 数据恢复流程

- AOF 开启且存在 AOF 文件时，优先加载 AOF 文件
- AOF 关闭或 AOF 文件不存在时，加载 RDB 文件
- 加载 AOF/RDB 文件成功后，Redis 启动成功
- AOF/RDB 文件存在错误时，Redis 启动失败并打印错误信息

### AOF 和 RDB 优缺点

- RDB 优点
	- 文件较小
	- RDB 恢复数据远快于 AOF
- RDB 缺点
	- 不能实时持久化
- AOF 优点
	- 数据完整性较好
	- 可实时持久化
- AOF 缺点
	- 文件较大
	- 数据库恢复比较慢，不合适做冷备
	- AOF 开启后，会对写的 QPS（单位时间内能够处理事务的数量）有所影响，相对于 RDB 来说写 QPS 要下降

> AOF 适合灾难性的数据恢复
> RDB 适合全量复制

### 冷备份和热备份

- 冷备份
	- 停止或暂停写入后备份，数据更容易保持一致。
	- 代价是影响可用性，适合维护窗口或离线环境。
- 热备份
	- 服务运行中备份，不影响正常读写。
	- 需要依赖快照、复制或持久化机制保证备份一致性，方案设计比冷备份复杂。

## Redis 异常

### 缓存穿透

- 查询不存在的数据，缓存未命中后请求直接打到数据库。
- 解决：缓存空值并设置较短 TTL；使用布隆过滤器提前拦截不存在的 Key；接口层做参数校验和限流。
- 注意：布隆过滤器有误判率，判断"不存在"一定不存在，判断"存在"不一定真的存在。

### 缓存击穿

- 热点 Key 过期瞬间，大量请求同时落到数据库。
- 解决：互斥锁，只允许一个请求回源重建缓存；热点 Key 逻辑过期，由后台异步刷新；热点数据提前续期。
- 权衡：互斥锁一致性更好但会阻塞请求；逻辑过期性能更好但可能短时间返回旧数据。

### 缓存雪崩

- 大量 Key 同时失效，或 Redis 整体不可用，导致请求集中打到数据库。
- 解决：过期时间加随机抖动；热点数据预热；限流、熔断、降级；多级缓存；Redis 高可用部署。

> 缓存击穿和缓存雪崩的区别在于，缓存击穿指并发查询同一条数据，而缓存雪崩是批量的数据过期了
> 缓存击穿和缓存雪崩最终都要查询数据库

### 缓存预热

- 定义
	- 系统上线后，将相关的数据加载到 Redis 中，避免在用户请求的时候，先查询数据库，然后再将数据回写到缓存
- 影响
	- 如果不进行预热，那么 Redis 初始状态数据为空，系统上线初期，对于高并发的流量，都会访问到数据库中，对数据库造成流量的压力
- 解决
	- 数据量不大时，工程启动进行加载缓存动作
	- 数据量较大时，设置定时任务脚本，刷新缓存
	- 数据量很大时，优先保证热点数据提前加载到缓存中

### 缓存降级

- 定义
	- 缓存降级是指缓存失效或缓存服务器挂掉的情况下，不去访问数据库，直接返回默认数据或访问服务的内存数据
- 影响
	- 通常会将部分热点数据缓存到服务的内存中，这样一旦缓存出现异常，可以直接使用服务的内存数据，从而避免数据库遭受巨大压力
	- 一般是有损的操作，所以尽量减少降级对于业务的影响程度

### Redis 内存淘汰机制

- 内存淘汰定义
	- 内存不足时，淘汰旧数据缓存新数据
- 配置最大内存
		- 通过 `redis.conf` 配置文件修改，示例：`maxmemory 1024mb`
		- 动态命令修改，示例：`CONFIG SET maxmemory 1024mb`，使用 `CONFIG GET maxmemory` 查看最大内存
	- 64 位操作系统默认最大内存大小为剩余内存，32 位操作系统默认为 `3 GB`
- 淘汰策略分类
	- `noeviction`，默认策略，对于写请求直接返回错误，不进行淘汰
	- `volatile-ttl`，从设置过期时间的 Key 中淘汰，越早过期的优先被淘汰
	- `allkeys-lru`，最近最少使用，从所有的 Key 中使用 LRU 算法进行淘汰
	- `volatile-lru`，最近最少使用，从设置过期时间的 Key 使用 LRU 淘汰
	- `allkeys-random`，从所有的 Key 随机淘汰
	- `volatile-random`，从设置过期时间的 Key 中随机淘汰
	- `allkeys-lfu`，最少使用频率，从所有的 Key 中使用 LFU 算法进行淘汰
	- `volatile-lfu`，最少使用频率，从设置过期时间的 Key 使用 LFU 淘汰
- Redis 修改的 LRU
	- 原始 LRU 是最近最少使用
	- Redis 中的 LRU 每次随机选出 `5` 个 Key，淘汰最近最少使用的 Key，可以通过 `maxmemory-samples 10` 参数修改采样数量
	- 将采样参数配置越大，越接近真实的 LRU，CPU 耗费增大
	- Redis 给每个 Key 增加了 `24 Bit` 的字段，存储该 Key 被最后访问的时间
- Redis 改进的 LRU
	- Redis 3.0 对近似 LRU 算法进行了一些优化。新算法会维护一个候选池（大小为 `16`），池中的数据根据访问时间进行排序，第一次随机选取的 Key 都会放入池中，随后每次随机选取的 Key 只有在访问时间小于池中最小的时间才会放入池中，直到候选池被放满
	- 当放满后，如果有新的 Key 需要放入，则将池中最后访问时间最大（最近被访问）的移除
	- 当需要淘汰的时候，则直接从池中选取最近访问时间最小（最久没被访问）的 key 淘汰掉就行
- Redis 中的 LFU
	- 最少使用频率，很少被访问的优先被淘汰，被访问的多的则被留下来
	- 比 LRU 更确切地表示热点数据（如使用 LRU 时，数据只被访问了一次却表示为热点数据，而有些数据可能将来会被访问却被淘汰）

## Redis 事务

### Redis 有事务机制吗

- 有，使用 `MULTI` 开启一个事务
- 每次操作的命令都会加入到一个队列中，但命令此时并没有真正执行
- 使用 `EXEC` 命令提交事务，开始顺序地执行队列的命令

### Redis 事务是否原子性

- Redis 事务保证命令按顺序连续执行，中间不会被其他客户端命令插入。
- 如果入队阶段出现语法错误，`EXEC` 时整个事务不会执行。
- 如果执行阶段某条命令因类型等运行时错误失败，其他命令仍会继续执行，Redis 不会自动回滚。
- 因此 Redis 事务不是关系型数据库意义上的 ACID 事务。

### Redis 不支持回滚

- Redis 选择不支持回滚，是为了保持实现简单和执行高性能。
- 生产中应在业务层保证命令类型正确，必要时用 Lua 脚本把复杂逻辑封装成单次原子执行。

### Redis 事务相关命令

- `MULTI`，开启一个事务，总是返回 `OK`，`MULTI` 执行之后, 客户端可以继续向服务器发送任意多条命令，这些命令不会立即被执行，而是被放到一个队列中，当 `EXEC` 命令被调用时，所有队列中的命令才会被执行
- `WATCH`，被 `WATCH` 的键会被监视，并会发觉这些键是否被改动过了。如果有至少一个被监视的键在 `EXEC` 执行之前被修改了，那么整个事务都会被取消， `EXEC` 返回 `nil-reply` 来表示事务已经失败
- `UNWATCH`，取消 `WATCH` 命令对所有键的监视，一般用于 `DISCARD` 和 `EXEC` 命令之前，如果在 `DISCARD` 和 `EXEC` 命令之后执行没有意义（`WATCH` 效果已经产生，`DISCARD` 执行时会取消对所有键的监视）
- `DISCARD`，事务被放弃，事务队列清空，客户端从事务状态中退出
- `EXEC`，负责执行事务中的命令，如果成功开启事务后执行 `EXEC`，那么所有命令都会被执行，如果开启事务但没有开启 `EXEC` 则所有命令都不会执行

> 在事务运行期间虽然 Redis 命令可能会执行失败，但是 Redis 依然会执行事务内剩余的命令而不会执行回滚操作

## Redis 分布式

### Redis 主从复制

- 定义
	- 将 Redis 服务器（主节点 Master）的数据复制到其他服务器（从节点 Slave）
	- 只能由主节点复制到从节点（单向的）
- 作用
	- 数据冗余，实现热备份，是持久化之外的数据冗余方式
	- 故障恢复，主节点宕机时，使用从节点服务
	- 负载均衡，提供读写分离，主节点提供写服务，从节点提供读服务（提高并发量）
	- 高可用，哨兵和集群的基础
- 流程阶段
	- 建立连接
		- 在主从节点之间建立连接，为数据同步做好准备
	- 数据同步
		- 建立以后，便可以开始进行数据同步，是从节点数据的初始化
		- 数据同步阶段是主从复制最核心的阶段，根据主从节点当前状态的不同，可以分为全量复制和部分复制
	- 命令传播
		- 这个阶段主节点将自己执行的写命令发送给从节点，从节点接收命令并执行，从而保证主从节点数据的一致性
		- 命令传播是异步的过程，即主节点发送写命令后并不会等待从节点的回复

### Redis 哨兵机制

- 背景
	- 主从模式下，主节点故障时需要手动将从节点晋升为主节点
	- 提供了 Redis Sentinel（哨兵机制）解决这个问题，Redis Sentinel 可管理多个 Redis 实例，可以实现对 Redis 的监控、通知、自动故障转移（主要作用，高可用）
	- 哨兵模式通常由一组 Sentinel 节点和一组（或多组）主从复制节点组成
- 原理
	- 心跳机制
	- 故障转移
		- 每个哨兵都会定时进行心跳检测，如果主节点超时，则认为主节点不可用（主观下线）
		- 该哨兵询问其他哨兵对主节点的判断，当 `quorum`（法定人数） 个哨兵认为主节点下线时，则执行客观下线
		- 哨兵间基于 Raft 算法选出一个领导者来进行故障转移的工作
		- 选举完哨兵领导者后
			- 在从节点选出一个新节点
			- 从节点执行 `slaveof no one` 命令让其称为主节点
			- 使旧主节点变成从节点
			- 哨兵领导者会向剩余的从节点发送命令，让其余从节点复制新主节点的数据

> 通过 Sentinel 和 Redis Cluster，Redis 实现了心跳机制，保证了系统的高可用性和稳定性
> Sentinel 适用于单点故障场景，而 Redis Cluster 则适用于分布式场景

### Redis 心跳机制

- 哨兵实现
	- 哨兵会周期性地向 Redis 服务器发送 `PING` 命令，以检测服务器是否正常工作
	- 如果服务器无响应，哨兵会将服务器标记为不可用，并根据预设的故障转移策略来执行自动故障转移
- 集群实现
	- 每个节点都会周期性地向其他节点发送心跳消息，以保持连接
	- 如果一个节点在一定时间内没有收到其他节点的心跳消息，它会将对应的节点标记为失败，并向其他节点发送信息，最后进行故障转移

### Redis 集群机制

- 背景
	- 主从模式还是哨兵模式都只能由主节点写数据，在海量数据高并发场景，单个节点写数据容易出现瓶颈，Cluster 模式可以实现多个节点同时写数据
- 原理
	- Redis Cluster 采用无中心结构，每个节点都保存数据，节点之间互相连接从而知道整个集群状态
	- 本质是多个主从复制结构的组合，一个主从复制结构看作一个节点

## Redis 提升

### Memcache 和 Redis 区别

- Memcache 将所有数据存储在内存中，Redis 部分存储在硬盘（持久化）
- Memcache 支持的数据类型较少，Redis 支持的数据类型丰富
- 底层模型不同，Redis 构建了 VM 机制

### 根据固定的前缀找出 Key

- 使用 `KEYS` 命令（进行全键空间扫描），配合正则表达式搜索

```bash
KEYS user*
```

- 使用 `SCAN` 命令，无阻塞的提取出指定模式的 Key 列表（需要去重）

### 大量键同时过期，需要注意什么

- 可能导致缓存雪崩
- 过期时间点可能出现卡顿（Redis 是单线程的）
- 在过期时间加上随机值，让过期时间尽量分散

### Redis 常用客户端

- Jedis
- Redis-py
	- 官方客户端
	- 支持同步和异步
	- 支持连接池
- Redis-py-cluster
	- 集群的官方客户端
	- 提供与 Redis-py 相同的接口
- Aioredis
	- 支持异步，使用了 `asyncio`
	- 适用于高并发和异步的场景
- Toredis
	- 支持异步，使用了 `asyncio`
	- 支持高级特性（订阅/发布等）
- Redis-py-sentinel
	- 哨兵的官方客户端
	- 提供与 Redis-py 相同的接口
