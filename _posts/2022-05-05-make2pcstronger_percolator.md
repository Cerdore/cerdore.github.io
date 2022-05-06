---
layout: post
title: 2PC加强版--Percolator
date: 2022-05-05 22:51:14 +0800
categories: 技术 数据事务
tags: 2PC 分布式事务 一致性 数据库
keywords: 事务
description: 从2PC到Percolator
---

## Reference

[1]. https://zhuanlan.zhihu.com/p/53197633

[2]. 

[TOC]

# 2PC

问题：如何保证多节点对某件事情的原子性

难点：节点可能宕机、网络可能分区、请求可能超时

方案： 2PC。Prepare + commit 两个阶段

缺点：

+ 同步阻塞：协调者(Coordinator)宕机，Worker节点不清楚该怎么办了，事务可能会block
+ 延时高：需要持久化 decision log

# Percolator

动机：为BigTable提供跨行跨表的事务原子性，同时解决 2PC 的同步阻塞问题。

方案：结合共识（Consensus）和 2PC 算法，将decision log 高可用，其他的信息也会保存primary key的decision log，有一台机器宕机没有影响。

缺点：

+  延时高：decision log不仅需要持久化，还需要达成共识

【补充图片。】

## 隔离级别：SI 快照隔离级别

SI隔离级别的问题：两个都有读A读B的事务，事务1中间set a=b, 事务2中间set b = a，最后可能会导致a和b交换。可能无法保证可串行化。

> 对于读：读操作都能够从一个带[时间戳](https://www.zhihu.com/search?q=时间戳&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"53197633"})的稳定快照获取
>
> 对于写：较好地处理写-写冲突：若事务并发更新同一个记录，最多只有一个会提交成功

全局时钟： tidb通过pd获取全局时钟。

时钟 TSO

并发控制：MVCC，读写不互相阻塞。读事务可以无锁快照读。就是存储开销大。

## 实现

## 预备概念

Primary Key：有个事务要写一批key，任意选一个key当这个事务的主键，通过这个主键判断事务状态（commit， rollback）。

PreWrite

Commit

stateless cooordinator：client

lazy方式的锁清理

## Percolator 存储布局

Percolator在BigTable上抽象了五个COLUMN，其中三个跟事务相关。

**Data Column** CF_DEFAULT  存储值

*{key, start_ts} -> value*

**Lock Column** CF_LOCK:  存储事务的锁信息

tinysql中：*{key -> lock_info}*

论文中：{key, start_ts} -> {primary_key, lock_type, ...}

primary_key: 事务primary引用。在执行Percolate事务时，会从待修改的keys中选择一个(随机选择)作为 Primary，其余的则作为 Secondaries.

prewrite阶段和data column一起写入。事务产生的锁，未提交的事务会写本项，记录primary lock的位置。事务成功提交后，该记录会被清理。

**Write Column** CF_WRITE **已提交**的数据信息。存储提交时间和类型

*{key, commit_ts} -> start_ts* 

*{key, commit_ts} -> wrIte_info* 

判断本次事务是否提交成功。只有该列正确写入后，事务的修改才会真正被其他事务可见。读请求会首先在该COLUMN中寻找最新一次提交的start timestamp，这决定了接下来从DATA COLUMN的哪个key读取最新数据。

## 工作流程

1. 通过 tso，给一个事务分配一个start timestamp
2. 预写这个事务的所有 key。写入到kv存储里加锁
3. 分配一个 commit timestamp。
4. 提交 primary keys。根据主键的情况进行判断，如果主键提交成功，secondary keys 可以进行异步提交。
5. 根据提交状态决定是否回滚

### PreWrite

1. 客户端获取全局唯一时间戳作为当前事务的 *start_ts*；
2. 客户端会从所有key中选出一个作为 *Primary*，其余的作为 *Secondaries*。并将所有的key/value数据写入请求并行地发往对应的存储节点。存储节点对key的处理如下：
   1. 写-写 冲突检查：从WRITE COLUMN列中获取当前key的最新数据，若其 commit_ts 大于等于 *start_ts*，说明在该事务的更新过程中其他事务提交过对该key的修改，返回*WriteConflict*错误
   2. 检查key是否已被锁，如果是，返回*KeyIsLock*的错误
   3. 向LOCK COLUMN列写入 `{start_ts, key, primary_ref}`为当前key加锁。若当前key被选为primary， *primary_ref* 标记为 Primary 。若为 Secondary，则指向 *primary* 的信息
   4. 向DATA COLUMN列写入数据 `{key, start_ts, value}`

### Commit

1. 从Oracle获取时间戳作为事务的提交时间 *commit_ts* 
2. 向primary key所在的存储节点发送commit请求
3. 步骤2正确完成后该事务即可标记为成功，接下来异步并行地向secondary keys所在的节点发送commit请求
4. 存储节点对于客户端请求的处理：

> \1. 获取key的lock,检查其合法性，若非法，则返回失败
>
> \2. 将 `{key, commit_ts, start_ts}`写入WRITE COLUMN
>
> \3. 从LOCK COLUMN中删除key的锁记录以释放锁

值得说明的是，一旦 Primary 节点提交成功后，整个事务就算提交成功了。

在某些实现中（如TiDB），Commit阶段并非并行执行，而是先向 Primary 节点发起commit请求，成功后即可响应客户端成功且后台异步地再向 Secondaries 发起commit。

### 读取数据

读取数据时，需要从 WRITE COLUMN 中寻找有无读取的key。寻找一个最新的start_ts，然后通过 {key + start_ts}在DATA COLUMN中找到value。

## 锁的处理

TODO
