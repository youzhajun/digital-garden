---
title: 分布式锁
draft: false
tags:
  - 中间件
date: 2023-07-04
---
# 原理

使用 `Redis` 实现分布式锁的核心原理是利用 `Redis` **单线程执行命令** 和 **SET 命令的特定选项**来保证在分布式环境中对某个“锁”资源的互斥访问。

## 核心原理

1. **互斥性：** 确保在任何时刻，只有一个客户端能持有锁。    
2. **原子性：** 获取锁和释放锁的操作必须是原子的，避免在中间状态被干扰。
3. **容错性：** 即使持有锁的客户端崩溃或网络分区，也要有机制（通常是超时）最终释放锁，防止死锁。

## 执行步骤

1. 获取锁 `setnx key value`。得到锁的线程执行以下步骤：
2. 设置过期时间  `expire key 30`
3. 执行业务代码 
4. 释放锁 `del key`

#  Redisson

Redisson 是一个基于 Redis 的 Java 客户端库，它提供了更高级的分布式对象和服务，其中分布式锁是其最核心的功能之一。与直接使用 Redis 命令相比，Redisson 的分布式锁实现了**自动续期**、**可重入**、**锁等待**等高级特性，大幅降低了使用复杂度。

- Redisson 使用 Lua 脚本保证原子性操作。

> 什么是 Lua 脚本？
> 本质：是一种轻量级脚本语言，当Redis执行Lua脚本时，**整个脚本作为一个命令执行**，期间不会被其他命令中断。


## 上锁操作

### 默认上锁

```lua
-- KEYS[1] = 锁名称（如 myLock）
-- ARGV[1] = 锁过期时间（毫秒）
-- ARGV[2] = 客户端唯一ID（格式：UUID + 线程ID）

if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hset', KEYS[1], ARGV[2], 1)  -- 首次加锁
    redis.call('pexpire', KEYS[1], ARGV[1])   -- 设置过期时间，默认时间为30s
    return nil
end

if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1) -- 重入次数+1
    redis.call('pexpire', KEYS[1], ARGV[1])    -- 刷新过期时间
    return nil
end

return redis.call('pttl', KEYS[1]) -- 返回锁剩余时间（加锁失败）
```

### 锁过期

默认情况下 `redisson` 的超时时间为 30s， 此时就会出现当业务代码执行时长超过 30s ， 从31秒到代码执行结束的这段时间内会出现别人获得锁的情况。

`redisson` 提供了锁续期的逻辑（看门狗）。

```
// 伪代码逻辑
while (!isLockReleased) {
    if (redis.get(lockKey) == clientId) { // 检查是否仍持有锁
        redis.expire(lockKey, 30_000);    // 续期30秒
    }
    Thread.sleep(10_000); // 间隔10秒
}
```

## 锁释放

```lua
-- KEYS[1] = 锁名称
-- KEYS[2] = 解锁消息通道
-- ARGV[1] = 解锁消息
-- ARGV[2] = 过期时间
-- ARGV[3] = 客户端ID

if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then 
    return nil; -- 非当前线程持有锁
end 

local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
if (counter > 0) then 
    redis.call('pexpire', KEYS[1], ARGV[2]); -- 重入次数减1
    return 0; 
else 
    redis.call('del', KEYS[1]); -- 完全释放锁
    redis.call('publish', KEYS[2], ARGV[1]); -- 发布解锁消息
    return 1; 
end 
return nil;
```



# 一些问题

## redisson 中锁的 key 应该如何设置？
