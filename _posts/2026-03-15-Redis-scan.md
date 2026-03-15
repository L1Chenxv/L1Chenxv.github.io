---
layout: post
title: Redis 中的 Scan 遍历
categories: [Redis]
description: some word here
keywords: Redis scan 操作
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# Redis 中的 Scan 遍历

想要获取Redis中某个集合中的所有数据，`SMEMBERS`可能是大多数人的第一选择。但它有两个明显的缺点：

1. 这个指令没有 offset、limit 参数，是要一次性吐出所有的`value`，当你看到满屏的字符串刷的没有尽头时，你就知道难受了。
2. 由于 `redis` 是单线程的，其所有操作都是原子的，而 `SMEMBERS` 算法是遍历算法，复杂度是 `O(n)`，如果实例中有千万级以上的 `value`，这个指令就会导致 Redis 服务卡顿，所有读写 Redis 的其它的指令都会被延后甚至会超时报错，可能会引起缓存雪崩甚至数据库宕机。

Redis 为了解决这个问题，它在 2.8 版本中加入了指令——scan。

对于不同的数据类型，Scan的命令也不同

- [SCAN](https://redis.com.cn/commands/scan.html) 用于遍历当前数据库中的键。
- [SSCAN](https://redis.com.cn/commands/sscan.html) 用于遍历集合键中的元素。
- [HSCAN](https://redis.com.cn/commands/hscan.html) 用于遍历哈希键中的键值对。
- [ZSCAN](https://redis.com.cn/commands/zscan.html) 用于遍历有序集合中的元素（包括元素成员和元素分值）。

因为 [SCAN](https://redis.com.cn/commands/scan.html) 、 [SSCAN](https://redis.com.cn/commands/sscan.html) 、 [HSCAN](https://redis.com.cn/commands/hscan.html) 和 [ZSCAN](https://redis.com.cn/commands/zscan.html) 四个命令的工作方式都非常相似， 所以本文一并介绍这四个命令， 但是要记住：[SSCAN](https://redis.com.cn/commands/sscan.html) 命令、 [HSCAN](https://redis.com.cn/commands/hscan.html) 命令和 [ZSCAN](https://redis.com.cn/commands/zscan.html) 命令的第一个参数总是一个存储集合的键名。而 [SCAN](https://redis.com.cn/commands/scan.html) 命令则不需要在第一个参数提供任何数据库键 —— 因为它遍历的是当前数据库中的所包含的键。



##  SCAN 命令的基本用法

什么是Redis增量遍历？[SCAN](https://redis.com.cn/commands/scan.html) 命令是一个基于游标的遍历器，每次被调用之后， 都会向用户返回一个新的游标， 用户在下次遍历时需要使用这个新游标作为 [SCAN](https://redis.com.cn/commands/scan.html) 命令的游标参数， 以此来延续之前的遍历过程。

[SCAN](https://redis.com.cn/commands/scan.html) 返回一个包含两个元素的数组， 第一个元素是用于进行下一次遍历的新游标， 而第二个元素则是一个数组， 这个数组中包含了所有被遍历的元素。当 [SCAN](https://redis.com.cn/commands/scan.html) 命令的游标参数被设置为 `0` 时， 服务器将开始一次新的遍历，而当服务器向用户返回值为 `0` 的游标时， 表示遍历已结束。例如：

```c
redis 127.0.0.1:6379> scan 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
   10) "key:7"
   11) "key:1"
redis 127.0.0.1:6379> scan 17
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```

在上面这个例子中， 第一次遍历使用 `0` 作为游标， 表示开始一次新的遍历。第二次遍历使用的是第一次遍历时返回的游标， 也就是命令回复第一个元素的值 `17` 。

## SCAN命令的有效性

因为 [SCAN](https://redis.com.cn/commands/scan.html) 命令仅仅使用游标来记录遍历状态， 所以这些命令带有以下缺点：

- 同一个元素可能会被返回多次。 处理重复元素的工作交由应用程序负责， 比如说， 可以考虑将遍历返回的元素，只用于可以安全地重复执行多次的操作上。
- 如果一个元素是在遍历过程中被添加到数据集的， 又或者是在遍历过程中从数据集中被删除的， 那么这个元素可能会被返回， 也可能不会， 这是不确定的。

## SCAN 命令每次执行返回的元素数量

[SCAN](https://redis.com.cn/commands/scan.html) 命令族并不保证每次执行都返回某个给定数量的元素。增量式命令甚至可能会返回零个元素， **但只要命令返回的游标不是 `0` ， 应用程序就不应该将遍历视作结束。**

不过命令返回的元素数量总是符合一定规则的， 在实际中：对于一个大数据集来说， 增量式遍历命令每次最多可能会返回数十个元素；而对于一个足够小的数据集来说， 小集合键、小哈希键和小有序集合键， 那么增量遍历命令将在一次调用中返回数据集中的所有元素。

最后， 用户可以通过增量式遍历命令提供的 **COUNT** 选项来指定每次遍历返回元素的最大值。

### COUNT选项

虽然 [SCAN](https://redis.com.cn/commands/scan.html) 命令不保证每次遍历所返回的元素数量， 但我们可以使用 **COUNT** 选项， 对命令的行为进行一定程度上的调整。 **COUNT** 选项的作用就是让用户告知遍历命令， 在每次遍历中**应该从数据集里返回多少元素**。虽然这个选项**只是对增量式遍历命令的一种提示**， 但是在大多数情况下， 这种提示都是有效的。

- **COUNT** 参数的默认值为 `10` 。
- 在遍历一个足够大的、由哈希表实现的数据库、集合键、哈希键或者有序集合键时， 如果用户没有使用 `MATCH` 选项， 那么命令返回的元素数量通常和 `COUNT` 选项指定的一样， 或者比 `COUNT` 选项指定的数量稍多一些。
- 在遍历一个编码为整数集合（intset，一个只由整数值构成的小集合）、 或者编码为压缩列表（ziplist，由不同值构成的一个小哈希或者一个小有序集合）时， 增量式遍历命令通常会无视 `COUNT` 选项指定的值， 在第一次遍历就将数据集包含的所有元素都返回给用户。

> 重要: **并非每次遍历都要使用相同的** `COUNT` 值。用户可以在每次遍历中按自己的需要随意改变 `COUNT` 值， 只要记得将上次遍历返回的游标用到下次遍历里面就可以了。

### MATCH 选项

和 [KEYS](https://redis.com.cn/commands/keys.html) 命令一样， [SCAN](https://redis.com.cn/commands/scan.html) 命令族也可以通过提供一个 glob 风格的模式参数， 让命令只返回和给定模式相匹配的元素， 可以通过在执行增量式遍历命令时， 通过给定 `MATCH <pattern>` 参数来实现。例如：

```c
redis 127.0.0.1:6379> sadd myset 1 2 3 foo foobar feelsgood
(integer) 6
redis 127.0.0.1:6379> sscan myset 0 match f*
1) "0"
2) 1) "foo"
   2) "feelsgood"
   3) "foobar"
redis 127.0.0.1:6379>
```

对元素的模式匹配工作是在命令从数据集中取出元素之后， 向客户端返回元素之前的这段时间内进行的，所以如果被遍历的数据集中只有少量元素和模式相匹配， 那么遍历命令或许会在多次执行中都不返回任何元素，**这也说明有可能出现COUNT 1000 个元素，实际返回给客户端的元素不足**。例如：

```c
redis 127.0.0.1:6379> scan 0 MATCH *11*
1) "288"
2) 1) "key:911"
redis 127.0.0.1:6379> scan 288 MATCH *11*
1) "224"
2) (empty list or set)
redis 127.0.0.1:6379> scan 224 MATCH *11*
1) "80"
2) (empty list or set)
redis 127.0.0.1:6379> scan 80 MATCH *11*
1) "176"
2) (empty list or set)
redis 127.0.0.1:6379> scan 176 MATCH *11* COUNT 1000
1) "0"
2)  1) "key:611"
    2) "key:711"
    3) "key:118"
    4) "key:117"
    5) "key:311"
    6) "key:112"
    7) "key:111"
    8) "key:110"
    9) "key:113"
   10) "key:211"
   11) "key:411"
   12) "key:115"
   13) "key:116"
   14) "key:114"
   15) "key:119"
   16) "key:811"
   17) "key:511"
   18) "key:11"
redis 127.0.0.1:6379>
```

## 使用错误的游标进行增量式遍历

[SCAN](https://redis.com.cn/commands/scan.html) 使用间断的、负数、超出范围或者其他非正常的游标来执行增量式遍历并不会造成服务器崩溃， 但可能会让命令产生不确定的结果。

只有两种游标是合法的：

- 在开始一个新的遍历时， 游标必须为 `0`。
- 增量式遍历命令在执行之后返回的游标值， 用于延续（continue）遍历过程的游标。

## 遍历结束的保证

[SCAN](https://redis.com.cn/commands/scan.html) 命令所使用的算法只保证在数据集的大小有界（bounded）的情况下， 遍历才会停止， 换句话说， 如果被遍历数据集的大小不断地增长的话， 增量式遍历命令可能永远也无法完成一次完整遍历。

从直觉上可以看出， 当一个数据集不断地变大时， 想要访问这个数据集中的所有元素就需要做越来越多的工作， 能否结束一个遍历取决于用户执行遍历的速度是否比数据集增长的速度更快。

## 返回值

[SCAN](https://redis.com.cn/commands/scan.html), [SSCAN](https://redis.com.cn/commands/sscan.html), [HSCAN](https://redis.com.cn/commands/hscan.html) and [ZSCAN](https://redis.com.cn/commands/zscan.html) 命令都返回一个包含两个元素的回复： 第一个元素是字符串表示的无符号 64 位整数（游标）， 第二个元素是本次被遍历的元素数组。

- [SCAN](https://redis.com.cn/commands/scan.html) key 数组。
- [SSCAN](https://redis.com.cn/commands/sscan.html) 集合成员的数组。
- [HSCAN](https://redis.com.cn/commands/hscan.html) HASH 键值对数组，一个键值对由一个键和一个值组成。
- [ZSCAN](https://redis.com.cn/commands/zscan.html) 元素数组，每个元素都是一个有序集合元素，一个有序集合元素由一个成员（member）和一个分值（score）组成。

# Spring Redis 中的 Scan

## Spring 中 SCAN 使用方式

在Spring中，可以通过`RedisTemplate`或`StringRedisTemplate`结合`ScanOptions`实现SCAN功能。

`ScanOptions` 中主要包含两个参数：

- `count`：这个参数指定了每次 `SCAN` 命令迭代时返回的键值对数量。注意，这个数量并不是 `SCAN` 命令最终返回的结果集数量，而是每次迭代时尝试返回的键值对数量。`Redis` 底层默认这个值为 10，但建议设置一个较大的值，以减少多次迭代带来的开销。然而，太大的值又可能失去分批扫描的意义。因此，这个值需要根据实际情况进行权衡。
- `match`：这个参数是一个匹配规则，类似于正则表达式，用于过滤出符合特定模式的键值对。通过 `match` 参数，可以让 `SCAN` 命令只返回和给定模式相匹配的键值对，实现模糊查询的效果。

## SCAN 注意事项

### SCAN 命令使用

在Redis终端使用SCAN命令时，每次只返回固定COUNT数量的数据，在多次遍历情况下，需要根据上次遍历返回的游标值，继续传递到下次SCAN命令中。**但Spring封装了这个过程，通过游标状态判断是否需要再次请求Redis获取下一批数据，是则直接请求，不需要人为干预。只需一次SCAN，就能获取全部数据**。

`Cursor`为`SCAN`扫描初次后返回的结果`ScanCursor`，其中包含关键的游标`cursorId`（<=0后则扫描结束），以及一次扫描的结果集等。`ScanCursor`中的`hasNext`经过重写，所以**一次扫描的结果遍历后并不代表结束**，还要看`CursorState`是否为FINISHED状态。如果当前批次已遍历完，但是状态还是`OPEN`的，则进行下一批次的`scan`，获取该批结果后更新`cursorId`、`state`、`delegate`。这边贴出部分编译后的源码：

```java
public boolean hasNext() {
    this.assertCursorIsOpen();

    while(!this.delegate.hasNext() && !ScanCursor.CursorState.FINISHED.equals(this.state)) {
        this.scan(this.cursorId);
    }

    if (this.delegate.hasNext()) {
        return true;
    } else {
        return this.cursorId > 0L;
    }
}
```

### 必须手动关闭 Cursor

这是最重要的一点。`Cursor` 保持着一个与 Redis 的活动连接。如果你遍历完数据（或者中途抛出异常）却没有显式关闭它，这个连接可能不会立即返回到连接池，导致连接泄露。

**正确姿势：使用 try-with-resources** 

Spring 的 `Cursor` 接口继承了 `Closeable`，因此可以自动关闭。

```java
ScanOptions options = ScanOptions.scanOptions().match("user:session:*").count(100).build();
try (Cursor<byte[]> cursor = redisConnection.scan(options)) {
    while (cursor.hasNext()) {
        process(cursor.next());
    }
} catch (IOException e) {
    // 异常处理
}
```

