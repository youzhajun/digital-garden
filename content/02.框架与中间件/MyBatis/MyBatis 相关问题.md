---
title: MyBatis 相关问题
draft: true
tags:
  - 中间件
date: 2023-07-18
---
# MyBatis 中的缓存机制

MyBatis 中有 二级缓存 机制。

## 一级缓存

- 一级缓存，`sqlsession` 级别，同一个 `sqlsession` 中多次相同查询的 `sql` 语句会查询缓存。
	- 一级缓存默认是开启的，并且不能关闭（`statement` 相当于不走缓存）。可以通过 `sqlsession.clearCache()` 清理缓存。
	- 不同 `sqlsession` 是隔离的。

### 一级缓存失效的情况

- 不同 `sqlsession` 是隔离的。
- 同一个 `sqlsession` 中，使用的查询条件不同
- 同一个 `sqlsession` 中，两次查询操作之间进行了增删改操作
- 同一个 `sqlsession` 中，两次查询操作之间手动清理了缓存，提交、关闭、清理都会清理缓存

## 二级缓存

- 二级缓存针对多个 `sqlsession` 
- 二级缓存是 `mapper` 级别的缓存，根据 `mapper` 的 `namespace` 做区分。
- 二级缓存查询返回的如果是对象，则必须要序列化

步骤：
1. , 当一个 `sqlsession` 查询得到结果关闭后，会将一级缓存的内容存入到二级缓存中。
2. 其他 `sqlsession` 请求时会先请求二级缓存，再查一级缓存。
3. 当有增删改操作时，会先清空二级缓存，再清理一级缓存。


### 如何开启二级缓存

1. `springboot` 项目中配置文件添加：
   
	```yaml
	mybatis:
		configuration:
			cache-enabled: true
			
	```

2. `xml` 文件中编写sql 需要有 `<cache/> `