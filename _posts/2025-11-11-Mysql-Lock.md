---
layout: post
title: 
categories: [cate1, cate2]
description: some word here
keywords: keyword1, keyword2
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



Mysql 45讲 30章——加锁动态解析

笔者使用的mysql版本为 `8.0.29`，隔离级别为 `REPEATABLE-READ`

# 前言

Mysql45讲中提到可重复读隔离级别下，加锁包含两个“原则”，两个“优化”和一个“bug“：

原则 1：加锁的基本单位是 next-key lock。希望你还记得，next-key lock 是前开后闭区间。

原则 2：查找过程中访问到的对象才会加锁。

优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。

优化 2：索引上的等值查询，**向右遍历**时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。

> 这里的等值查询指的是：
>
> 1. 索引查不到记录
> 2. 非唯一索引查到记录，还需要继续向右遍历

一个 bug：无论是否唯一索引，范围查询均会访问到不满足条件的第一个值为止（其实二级索引也会按照这种方式扫描）。

> 范围查询且向右遍历，**对于`>=或<= 主键`的这种边界条件来说（特指主键，不包括唯一索引），如果当前记录恰好是开始/结束边界，就仅需对该记录加行锁，而不需添加gap锁。**其他情况向右遍历到最后一个值不满足等值条件，主键索引next-key lock 退化为间隙锁，非主键索引保持next-key 锁



接下来，我们的讨论还是基于下面这个表 t：

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

# 正文

## 不等号条件里的等值查询

### 降序查询

我们一起来看下这个例子，分析一下这条查询语句的加锁范围：

```sql
begin;
select * from t where id>9 and id<15 order by id desc for update;
```

我们可以通过 `select * from performance_schema.data_locks\G;` 这条语句，查看事务执行 SQL 过程中加了什么锁。

```sql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:1311:2872807263256
ENGINE_TRANSACTION_ID: 125213
            THREAD_ID: 48
             EVENT_ID: 61
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2872807263256
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:250:4:5:2872791206936
ENGINE_TRANSACTION_ID: 125213
            THREAD_ID: 48
             EVENT_ID: 61
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2872791206936
            LOCK_TYPE: RECORD
            LOCK_MODE: X,GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:250:4:3:2872791207280
ENGINE_TRANSACTION_ID: 125213
            THREAD_ID: 48
             EVENT_ID: 61
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2872791207280
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 5
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:250:4:4:2872791207280
ENGINE_TRANSACTION_ID: 125213
            THREAD_ID: 48
             EVENT_ID: 61
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2872791207280
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10
4 rows in set (0.00 sec)
```

利用上面的加锁规则，我们知道这个语句的加锁范围是主键索引上的 (0,5]、(5,10]和 (10, 15)。也就是说，id=15 这一行，并没有被加上行锁。为什么呢？

如图 1 所示，是这个表的索引 id 的示意图。

<figure style="text-align: center;">
  <img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/mysql/image.4xuytk91qy.webp" alt="" style="zoom:33%;" />
  <figcaption style="font-style: italic; text-align: center;">图1: 索引id示意图</figcaption>
</figure>
下边是真正处理记录并给记录加锁的流程，我们给这些流程编个号。

#### 1. 定位扫描区间的第一条记录

下边开始通过B+树定位某个扫描区间中的第一条记录了（对于一个扫描区间来说，只执行一次下述函数，因为只要定位到扫描区间的第一条记录之后，就可以沿着记录所在的单向链表进行查询了）：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/mysql/image.77dzdqf2bv.webp)

其中`btr_pcur_open_with_no_init`是用于定位扫描区间中的第一条记录的函数。

#### 2. 对于ORDER BY ... DESC条件形成的扫描区间的第一条记录的处理

在B+树的每层节点中，记录是按照键值从小到大的方式进行排序的。对于某个扫描区间来说，InnoDB通常是定位到扫描区间中最左边的那条记录，也就是键值最小的那条记录，然后沿着从左往右的方式向后扫描。

但是上述查询语句的语义是 `order by id desc`，要拿到满足条件的所有行，InnoDB定位到扫描区间中的第一条记录应该是该扫描区间中最右边的那条记录，也就是键值最大的那条记录（在执行`btr_pcur_open_with_no_init`时就定位到最右边的那条记录）

很显然，id=10的是扫描区间中最右边的记录

对于从右向左扫描`扫描区间`中记录的情况，针对从扫描区间中定位到的最右边的那条记录，需要做如下处理：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/mysql/image.6m4brfsusp.webp)

可以看到，对于加锁读来说，在隔离级别不小于`REPEATABLE READ`并且也没有开启`innodb_locks_unsafe_for_binlog`系统变量的情况下，会对扫描区间中最右边的那条记录的下一条记录加一个类型为`LOCK_GAP`的锁，这个类型为`LOCK_GAP`的锁其实就是`GAP锁`。

在本例中，扫描区间最右边的记录的下一条记录就是id=15的记录，接下来就会对该记录加一个(10,15)的`GAP锁`

### 升序查询

那么，当把上面的查询语句替换为升序查询，又是怎么加锁的呢？

```sql
begin;
select * from t where id>9 and id<12 order by id asc for update;
```



```sql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:1311:2872807263256
ENGINE_TRANSACTION_ID: 125214
            THREAD_ID: 48
             EVENT_ID: 77
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2872807263256
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:250:4:4:2872791206936
ENGINE_TRANSACTION_ID: 125214
            THREAD_ID: 48
             EVENT_ID: 77
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2872791206936
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 2872801758592:250:4:5:2872791207280
ENGINE_TRANSACTION_ID: 125214
            THREAD_ID: 48
             EVENT_ID: 77
        OBJECT_SCHEMA: lcx_db01
          OBJECT_NAME: t
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2872791207280
            LOCK_TYPE: RECORD
            LOCK_MODE: X,GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15
3 rows in set (0.00 sec)
```

可以看到，这个语句的加锁范围是主键索引上的(5,10]和 (10, 15)。为什么从降序改为升序查询，(0,5]这个区间就没有加锁呢？

回顾一下前文的加锁规则:

1. 首先这个查询语句的语义是 order by id asc，要拿到满足条件的所有行，优化器必须先找到“**第一个 id>9 的值**”,扫描到id=10这一行，加next-key lock (5,10]
2. 然后向右遍历，会扫描到 id=15 这一行（**这时候不是等值查询，命中加锁原则的一个 bug**），所以会加一个 next-key lock (10,15]。又因为优化②，next-key lock 退化为间隙锁(10,15)
