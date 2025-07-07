---
title: Redis相关问题
draft: false
tags:
  - 数据库
date: 2023-06-30
---

## Redis 为什么快？有哪些主要原因？ 

## Redis 常见的5种数据类型？请举例说明它们各自的适用场景。
Redis 采用的是 k-v 存储结构，他的数据类型是针对键值对中的值来说的。Redis 中有以下5种常见的数据类型：`String`、`Set`、`List`、`Shorted Set`、`Hash`
### String
`k - v(String)`, 可用于用户登录后缓存用户 `token` 方便检索。
```java
// 登录成功后生成 Token 并缓存
String token = UUID.randomUUID().toString();
String redisKey = "login:token:" + userId;
redisTemplate.opsForValue().set(redisKey, token, Duration.ofHours(2));

```
### List
链表结构，按照插入顺序排序，可以从两端进行插入和弹出。借助他的特性，可以实现一个顺序消息队列。
```java
// 生产者推送消息
String smsMessage = "{\"phone\":\"13812345678\", \"text\":\"验证码1234\"}";
redisTemplate.opsForList().leftPush("queue:sms", smsMessage);

// 消费者轮询处理消息（可以用线程或定时任务）
while (true) {
    List<String> message = redisTemplate.opsForList().rightPop("queue:sms", Duration.ofSeconds(10));
    if (message != null) {
        processSms(message.get(0));
    }
}

```
### Set
无序且不重复的元素集合，支持交、并、差等集合操作。可用于用户点赞功能 + 防重复点赞。
```java
// 点赞操作
String key = "article:likes:" + articleId;
Boolean added = redisTemplate.opsForSet().add(key, userId.toString()) > 0;
if (added) {
    // 新增点赞成功，执行后续逻辑
    articleService.incrementLikeCount(articleId);
}

```
### Sorted Set (ZSet)
带有排序功能的不重复的元素集合。可用于排行榜系统，如视频播放量排行榜
```java
// 增加视频播放量
String key = "video:rank:views";
redisTemplate.opsForZSet().incrementScore(key, videoId.toString(), 1);

// 获取排行榜前10
Set<ZSetOperations.TypedTuple<String>> top10 = redisTemplate.opsForZSet()
    .reverseRangeWithScores(key, 0, 9);

```

### Hash
`k - v(k-v)` ，可用于存储用户信息。举个例子：
```java
// 缓存用户信息
String key = "user:info:" + userId;
Map<String, Object> userMap = new HashMap<>();
userMap.put("name", "Alice");
userMap.put("age", 30);
userMap.put("email", "alice@example.com");
redisTemplate.opsForHash().putAll(key, userMap);

// 查询字段信息
String name = (String) redisTemplate.opsForHash().get(key, "name");

```

## Redis 的两种持久化方式是什么？它们各自的优缺点是什么？
Redis 的数据存储在内存中，但同时也提供了两种持久化方式将数据写到硬盘中，当因意外导致重启时会先从磁盘中的数据再读到内存中，防止数据丢失。
Redis 提供了两种持久化方式： RDB（存快照）、AOF（存命令）
### RDB（默认开启）
RDB 方式会在指定的时间间隔内，将当前 Redis 中的数据以快照形式保存到磁盘的 `.rdb` 文件中。
### AOF
AOF 会以 **日志形式** 记录每一个写操作（例如 `SET`、`LPUSH` 等），并追加写入到 `.aof` 文件中，Redis 重启时会根据日志重放命令恢复数据。
### RDB 与 AOF  各自优缺点？

| 特性    | RDB 快照       | AOF 日志         |
| ----- | ------------ | -------------- |
| 数据安全性 | 宕机后可能丢失几分钟数据 | 更安全，最多丢失 1 秒数据 |
| 性能影响  | 较小           | 较大（写入频繁时）      |
| 文件大小  | 小，压缩格式       | 大，需定期重写        |
| 恢复速度  | 快（直接加载快照）    | 慢（命令重放）        |
| 使用场景  | 备份、快速重启      | 高可靠数据需求        |


### 如何修改持久化方式及参数？
Redis 的持久化方式配置都在 `redis.conf` 文件中（通常路径为 `/etc/redis/redis.conf`，或者手动指定的路径）。
> RDB 配置
```java
save 900 1     # 900 秒（15 分钟）内至少 1 次写操作，则生成快照
save 300 10    # 300 秒内至少 10 次写操作
save 60 10000  # 60 秒内至少 10000 次写操作

# 禁用 RDB 快照方式（不推荐）
# save ""

```
>AOF 配置
```java
appendonly yes          # 开启 AOF
appendfilename "appendonly.aof"  # AOF 文件名（默认值）
appendfsync everysec    # 每秒执行一次 fsync（推荐，性能与安全性平衡）
# appendfsync always     # 每次写操作都 fsync（最安全，但慢）
# appendfsync no         # 不 fsync，由操作系统决定刷新时机（最快但最不安全）

```

## RDB 和 AOF 之间如何选择？
在 **生产环境中**部署 Redis 时，如何在 **RDB 与 AOF** 之间做出选择，关系到系统的性能、数据安全与故障恢复策略。
>总结：RDB 更适合备份，AOF 更适合高数据可靠性场景；推荐生产环境“同时开启”。

- **单独使用 RDB（轻量模式）：**  对性能要求高，写入频繁。Redis 异常崩溃可能导致丢失近一次快照后的所有数据
- **单独使用 AOF（高可靠性模式）：** 数据必须强一致（如秒杀订单、交易系统），宕机恢复必须尽量还原全部数据。但是文件膨胀快，启动时恢复速度比 RDB 慢。

## AOF 重写（Rewrite）的原理是什么？为什么要进行 AOF 重写？

### 为什么重写？

AOF 持久化会将 **每一个写命令**（如 `SET name Tom`、`INCR page_view`）**不断追加**到日志文件中。
随着时间推移，文件会越来越大，出现问题：
- **磁盘占用过高**
- **Redis 重启恢复时间变长**
- **很多历史命令已无实际意义**（比如一个 key 被多次 `SET`，旧命令没用了）

**AOF 重写的目的在于：** **将当前内存中实际存在的数据**，重新生成一份最小化、无冗余的 AOF 文件，替换旧文件，减小文件体积，提高恢复效率。
### 原理
Redis 使用一种 **非阻塞的“后台重写”机制**，确保重写期间仍能处理客户端请求。重写流程如下：
1. **Redis fork 子进程**（类似 RDB 快照过程）：
    - 子进程在后台扫描当前 Redis 的内存数据，将其转换为一系列 **最简的写操作命令**。  
    - 比如将一个 hash 写成一条 `HMSET` 命令，而不是记录之前所有 `HSET` 操作。    
2. **主进程继续服务客户端**：
    - 所有新写入操作继续追加到原 AOF 文件和一个临时缓冲区中。
3. **子进程生成新的 AOF 文件（临时文件）**：
    - 不包含历史无效命令，只有当前状态的最小写命令集合。
4. **子进程完成后，主进程合并缓冲区数据**：
    - 将重写期间的写命令追加到新 AOF 文件末尾。
5. **原子替换旧 AOF 文件**：
    - 新 AOF 文件准备完毕后，替换原文件，并开始使用。
### 如何触发重写机制
关于重写机制的设置也是在 Redis 的配置文件中进行配置
```java
auto-aof-rewrite-percentage 100 #当前 AOF 文件大小 > 上次重写后大小的 100%
auto-aof-rewrite-min-size 64mb #且当前大小大于 64MB
```
同时 Redis 提供了命令进行手动触发重写
```java
BGREWRITEAOF
```


# 缓存穿透

缓存穿透是由于请求一个 redis 中不存在的数据而导致的。
- 用户暴力查询一个redis、db都不存在的数据

![[Redis相关问题-1751611012278.png]]

解决方式：
- 缓存空结果。 查询缓存查不到，查询数据库查不到，对于空数据也缓存到 redis 值为nuknow，减少数据库的查询
	- 优点：实现简单
	- 缺点：大量随机值会导致过多无效缓存
- 布隆过滤器[[布隆过滤器]]。


# 缓存击穿

缓存击穿：高并发条件下，热点数据失效的一瞬间，大量的请求发送到数据库，导致数据库被压垮。