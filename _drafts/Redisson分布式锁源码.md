# redisson中的分布式锁

### 可重入锁（Reentrant Lock）

基于Redis的Redisson分布式可重入锁`RLock` Java对象实现了`java.util.concurrent.locks.Lock`接口。

大家都知道，如果负责储存这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗检查锁的超时时间是30秒钟，也可以通过修改`Config.lockWatchdogTimeout`来另行指定。

`RLock`对象完全符合Java的Lock规范。也就是说只有拥有锁的进程才能解锁，其他进程解锁则会抛出`IllegalMonitorStateException`错误。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
RLock lock = redisson.getLock("anyLock");
// 最常见的使用方法
lock.lock();

// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
```

代码中使用

```java
@Autowired
private RedissonClient redissonClient;

public void checkAndLock() {
    // 加锁，获取锁失败重试
    RLock lock = this.redissonClient.getLock("lock");
    lock.lock();

    // 先查询库存是否充足
    Stock stock = this.stockMapper.selectById(1L);
    // 再减库存
    if (stock != null && stock.getCount() > 0){
        stock.setCount(stock.getCount() - 1);
        this.stockMapper.updateById(stock);
    }

    // 释放锁
    lock.unlock();
}
```

1. 加锁步骤

   1. 执行lua脚本
      1. 判断锁是否被占用（exists），如果没有被占用则直接获取锁（hset/hincrby）并设置过期时间（expire）
      2. 如果锁被占用，则判断是否当前线程占用的（hexists），如果是则重入（hincrby）并重置过期时间（expire）
      3. 否则获取锁失败，返回锁超时时间
   2. 启动定时任务
      1. scheduleExpreationRenewal ( thteadId )
      2. 使用定时器重置过期时间，使用Netty的时间轮
      3. 判断锁属不属于当前线程，属于则重置过期时间
      4. 重置成功，则再次开启一个定时任务
2. 解锁过程
   1. 判断锁是否是当前线程的
   
   2. 如果是当前线程则 `redis.call('hincrby', KEYS[1], ARGV[3], -1)`,并返回一个`-1`之后的counter值
   
   3. 如果counter值大于0，重置过期时间，并 return 0
   
   4. ≤0， 则删除key， return 1，并取消过期时间重置的定时器方法
   
   5. 不是当前线程，return nil
   
3. 压力测试

性能跟我们手写的区别不大。

![1606958454966](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/redis/1606958454966.49egviwasgc0.webp)

### 公平锁（Fair Lock）

基于Redis的Redisson分布式可重入公平锁也是实现了`java.util.concurrent.locks.Lock`接口的一种`RLock`对象。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockRx.html)的接口。它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。所有请求线程会在一个队列中排队，当某个线程出现宕机时，Redisson会等待5秒后继续下一个线程，也就是说如果前面有5个线程都处于等待状态，那么后面的线程会等待至少25秒。

```java
RLock fairLock = redisson.getFairLock("anyLock");
// 最常见的使用方法
fairLock.lock();

// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
fairLock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = fairLock.tryLock(100, 10, TimeUnit.SECONDS);
fairLock.unlock();
```

### 联锁（MultiLock）

基于Redis的Redisson分布式联锁[`RedissonMultiLock`](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/RedissonMultiLock.html)对象可以将多个`RLock`对象关联为一个联锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 所有的锁都上锁成功才算成功。
lock.lock();
...
lock.unlock();
```

只要有一个节点宕机都不能使用了。。。

### 红锁（RedLock）

基于Redis的Redisson红锁`RedissonRedLock`对象实现了[Redlock](http://redis.cn/topics/distlock.html)介绍的加锁算法。该对象也可以用来将多个`RLock`对象关联为一个红锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 红锁在大部分节点上加锁成功就算成功。
lock.lock();
...
lock.unlock();
```

### 读写锁（ReadWriteLock）

基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器#81-可重入锁reentrant-lock)接口。

分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。

```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();

// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```



#### **Redisson加读锁逻辑原理**

```java
public class RedissonReadLock extends RedissonLock implements RLock {
    @Override
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        // 内部锁的租期时间 单位unit
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                                              //首先检查当前锁的模式（mode）
                                "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                                              //如果不存在（mode == false），
                                              //则表示没有其他线程正在持有锁，可以直接获取读锁
                                "if (mode == false) then " +
                                              // 设置锁的模式为 'read'，然后设置锁的租期
                                  "redis.call('hset', KEYS[1], 'mode', 'read'); " +
                                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                                  "redis.call('set', KEYS[2] .. ':1', 1); " +
                                  "redis.call('pexpire', KEYS[2] .. ':1', ARGV[1]); " +
                                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                              // 返回 nil 表示成功获取锁
                                  "return nil; " +
                                "end; " +
                                              //如果锁的模式是 'read' 或者是 'write' 并且当前线程已经获取了写锁，
                                              //则表示当前线程可以再次获取读锁
                                "if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then " +
                                              // 递增当前线程获取读锁的计数，并设置锁的租期
                                  "local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                                  "local key = KEYS[2] .. ':' .. ind;" +
                                              // 设置当前线程string
                                  "redis.call('set', key, 1); " +
                                  "redis.call('pexpire', key, ARGV[1]); " +
                                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                              // 返回 nil 表示成功获取锁
                                  "return nil; " +
                                "end;" +
                                              // 如果无法获取读锁，表示锁已被其他线程占用，此时返回锁的剩余租期时间
                                "return redis.call('pttl', KEYS[1]);",
                        Arrays.<Object>asList(getName(), getReadWriteTimeoutNamePrefix(threadId)), 
                        internalLockLeaseTime, getLockName(threadId), getWriteLockName(threadId));
    }
}
```

> **KEYS：**
>
> - KEYS[1](https://cloud.tencent.com/developer/tools/blog-entry?target=https://www.cnblogs.com/wang-meng/): `getName()` = anyLock
> - KEYS[2]: `getReadWriteTimeoutNamePrefix(threadId)` =  {anyLock}:UUID_01:threadId_01:rwlock_timeout
>
> **ARGV：**
>
> - ARGV[1](https://cloud.tencent.com/developer/tools/blog-entry?target=https://www.cnblogs.com/wang-meng/): internalLockLeaseTime = 30000毫秒
> - ARGV[2]: getLockName(threadId) = UUID_01:threadId_01
> - ARGV[3]: getWriteLockName(threadId) = UUID_01:threadId_01:write

**客户端A（UUID_01:threadId_01）来加读锁**

当执行上述lua脚本后：

```java
anyLock: {
  "mode": "read",
  "UUID_01:threadId_01": 1
}

{anyLock}:UUID_01:threadId_01:rwlock_timeout:1  1
```

**客户端A 第二次来加读锁**

```java
anyLock: {
  “mode”: “read”,
  “UUID_01:threadId_01”: 2
}

{anyLock}:UUID_01:threadId_01:rwlock_timeout:1  1
{anyLock}:UUID_01:threadId_01:rwlock_timeout:2  1
```

**客户端B （UUID_02:threadId_02）第一次来加读锁**

```java
anyLock: {
  "mode": "read",
  "UUID_01:threadId_01": 2,
  "UUID_02:threadId_02": 1
}

{anyLock}:UUID_01:threadId_01:rwlock_timeout:1  1
{anyLock}:UUID_01:threadId_01:rwlock_timeout:2  1
{anyLock}:UUID_02:threadId_02:rwlock_timeout:1  1
```

#### **Redisson加写锁逻辑原理**

```java
public class RedissonWriteLock extends RedissonLock implements RLock {
    @Override
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        // 内部锁的租期时间 单位unit
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                                              // 首先检查当前锁的模式（mode）
                            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                                              // 如果不存在（mode == false），则表示没有其他线程正在持有锁，
                                              // 可以直接获取写锁
                            "if (mode == false) then " +
                                  "redis.call('hset', KEYS[1], 'mode', 'write'); " +
                                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                  "return nil; " +
                              "end; " +
                                              // 如果锁的模式是 'write'，表示当前线程已经获取了写锁
                              "if (mode == 'write') then " +
                                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                                              // 直接递增当前线程获取写锁的计数，并更新锁的租期
                                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                                      "local currentExpire = redis.call('pttl', KEYS[1]); " +
                                      "redis.call('pexpire', KEYS[1], currentExpire + ARGV[1]); " +
                                      "return nil; " +
                                  "end; " +
                                "end;" +
                                              // 如果无法获取写锁，表示锁已被其他线程占用，此时返回锁的剩余租期时间
                                "return redis.call('pttl', KEYS[1]);",
                        Arrays.<Object>asList(getName()), 
                        internalLockLeaseTime, getLockName(threadId));
    }
}
```

> KEYS和ARGV参数：
>
> - KEYS[1](https://cloud.tencent.com/developer/tools/blog-entry?target=https://www.cnblogs.com/wang-meng/) = anyLock
> - ARGV[1](https://cloud.tencent.com/developer/tools/blog-entry?target=https://www.cnblogs.com/wang-meng/) = 30000
> - ARGV[2] = UUID_01:threadId_01:write
>
> 1. hget anyLock mode，此时没人加锁，mode=false
> 2. hset anyLock mode write
> 3. hset anyLock UUID_01:threadId_01:write 1
> 4. pexpire anyLock 30000

**客户端A加写锁**

```java
anyLock: {
    "mode": "write",
    "UUID_01:threadId_01:write": 1
}
```

**客户端A和客户端B，先后加读锁，客户端C来加写锁**

```java
anyLock: {
  "mode": "read",
  "UUID_01:threadId_01": 1,
  "UUID_02:threadId_02": 1
}

{anyLock}:UUID_01:threadId_01:rwlock_timeout:1    1
{anyLock}:UUID_02:threadId_02:rwlock_timeout:1    1
```

客户端C加锁失败，就会不断的尝试重试去加锁

**客户端A先加写锁、客户端A接着加读锁**

```java
anyLock: {
  "mode": "write",
  "UUID_01:threadId_01:write": 1,
  "UUID_01:threadId_01": 1
}

{anyLock}:UUID_01:threadId_01:rwlock_timeout:1    1
```

**客户端A先加读锁、客户端A接着加写锁**

![image.png](https://ask.qcloudimg.com/http-save/yehe-1020141/5riey7039h.png)

此时客户端A先加的读锁，mode=read，所以再次加写锁是不能成功的

如果是同一个客户端同一个线程，先加了一次写锁，然后加读锁，是可以加成功的，默认是在同一个线程写锁的期间，可以多次加读锁

而同一个客户端同一个线程，先加了一次读锁，是不允许再被加写锁的

#### 总结

显然还有写锁与写锁互斥的逻辑就不分析了，通过上面一些场景的分析，我们可以知道：

- 读锁与读锁非互斥
- 读锁与写锁互斥
- 写锁与写锁互斥
- 读读、写写 同个客户端同个线程都可重入
- 先写锁再加读锁可重入
- 先读锁再写锁不可重入

### 2.10.6. 信号量（Semaphore）

基于Redis的Redisson的分布式信号量（[Semaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphore.html)）Java对象`RSemaphore`采用了与`java.util.concurrent.Semaphore`相似的接口和用法。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreRx.html)的接口。

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.trySetPermits(3);
semaphore.acquire();
semaphore.release();
```

```java
private RFuture<Boolean> tryAcquireAsync0(int permits) {
    return commandExecutor.syncedEval(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                                      "local value = redis.call('get', KEYS[1]); " +
                                      "if (value ~= false and tonumber(value) >= tonumber(ARGV[1])) then " +
                                      "local val = redis.call('decrby', KEYS[1], ARGV[1]); " +
                                      "return 1; " +
                                      "end; " +
                                      "return 0;",
                                      Collections.<Object>singletonList(getRawName()), permits);
}
```

> 这段代码是一个 Lua 脚本，用于在 Redis 中执行一些操作。该脚本的功能如下：
>
> 1. 通过 Redis 的 `get` 命令获取键为 `KEYS[1]` 的值，并将其赋给变量 `value`。
> 2. 如果 value 不为 false，并且其值大于等于 ARGV[1]，则执行以下操作：
>    - 使用 Redis 的 `decrby` 命令将键为 `KEYS[1]` 的值减去 `ARGV[1]`，并将结果赋给变量 `val`。
>    - 返回值 1，表示操作成功。
> 3. 如果上述条件不满足，则返回值 0，表示操作失败。

### 2.10.7. 闭锁（CountDownLatch）

基于Redisson的Redisson分布式闭锁（[CountDownLatch](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RCountDownLatch.html)）Java对象`RCountDownLatch`采用了与`java.util.concurrent.CountDownLatch`相似的接口和用法。

```java
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.trySetCount(1);
latch.await();

// 在其他线程或其他JVM里
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.countDown();
```

需要两个方法：**一个等待，一个计数countDown**



```java
public RFuture<Void> countDownAsync() {
    return commandExecutor.evalWriteNoRetryAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                                                 "local v = redis.call('decr', KEYS[1]);" +
                                                 "if v <= 0 then redis.call('del', KEYS[1]) end;" +
                                                 "if v == 0 then redis.call(ARGV[2], KEYS[2], ARGV[1]) end;",
                                                 Arrays.<Object>asList(getRawName(), getChannelName()),
                                                 CountDownLatchPubSub.ZERO_COUNT_MESSAGE, getSubscribeService().getPublishCommand());
}
```

> 1. 使用 Redis 的 `decr` 命令递减键 `KEYS[1]` 的值。
> 2. 如果递减后的值小于等于 0，则使用 Redis 的 `del` 命令删除键 `KEYS[1]`。
> 3. 如果递减后的值等于 0，则使用指定的发布/订阅命令 `ARGV[2]`，将键 `KEYS[2]` 的值设置为 `ARGV[1]`。
>
> 这段代码逻辑上是对一个 Redis 键进行递减操作，并在递减后的值满足特定条件时执行相应的操作。