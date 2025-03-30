2、Redis：

①组成，应用？

组成：

1. 数据结构：支持字符串（String）、列表（List）、哈希（Hash）、集合（Set）、有序集合（ZSet）等。
2. 持久化：
   - RDB（快照）：定时生成内存数据的二进制快照。
   - AOF（追加日志）：记录所有写操作命令，通过重放恢复数据。
3. 高可用与分布式：
   - 主从复制：实现数据冗余和读写分离。
   - 哨兵模式（Sentinel）：提供故障自动转移。
   - 集群模式（Cluster）：分片存储数据，支持水平扩展。
4. 其他功能：事务、Lua脚本、发布订阅、Stream（消息队列）、地理空间索引等。

应用场景：

- 缓存：加速热点数据访问，减轻数据库压力。
- 分布式锁：通过 SETNX 或 Redisson 实现。
- 排行榜：利用有序集合（ZSet）实时排序。
- 消息队列：基于列表（List）或 Stream 实现异步通信。
- 计数器：使用 INCR 实现原子性计数（如点击量）。
- 会话存储：存储用户登录状态，支持分布式会话共享。



②如何使用Redisson RLock分布式锁？底层原理？看门狗机制是什么？底层原理

底层原理

1. 基于 Redis 的锁实现

Redisson 的分布式锁基于 Redis 的 EVAL 命令（执行 Lua 脚本）实现。它使用了一个 Redis 键值对来表示锁，键是锁的名称，值包含锁的持有者信息和过期时间。

基本流程：

1. 获取锁：通过 Lua 脚本尝试在 Redis 中设置一个键值对，如果键不存在则获取成功。
2. 锁的持有：为该键设置过期时间（避免死锁）。
3. 锁的释放：通过执行 Lua 脚本删除对应的键。
4. 锁的续期：通过看门狗机制延长锁的过期时间。

2. 看门狗机制

看门狗（Watchdog）机制 是 Redisson 为分布式锁提供的一种自动续期功能。它能够在客户端持有锁期间，自动延长锁的有效期，防止因为执行时间过长导致锁过期被其他客户端获取，从而破坏互斥性。

工作原理：

1. 当客户端调用 lock() 方法获取锁时（不设置过期时间），Redisson 会默认设置一个 30 秒的锁有效期。
2. 同时，它会启动一个定时任务，默认每 10 秒检查一次（锁有效期的 1/3 时间）。
3. 如果客户端仍然持有锁，定时任务会自动刷新锁的有效期为 30 秒。
4. 这个过程会一直持续，直到客户端主动释放锁，或者客户端崩溃（此时看门狗停止工作，锁会在 30 秒后自动释放）。

关键源码分析：

    private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
        Long ttl = null;
        if (leaseTime != -1) {
            ttl = tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        } else {
            ttl = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
                    TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
            scheduleExpirationRenewal(threadId);
        }
        return ttl;
    }
    
    private void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);
            renewExpiration();
        }
    }
    
    private void renewExpiration() {
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ee == null) {
                    return;
                }
    
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((success, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getName() + " expiration", e);
                        return;
                    }
    
                    if (success) {
                        renewExpiration();
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    
        ee.setTimeout(task);
    }
    
    protected RFuture<Boolean> renewExpirationAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                "return 1; " +
            "end; " +
            "return 0;",
            Collections.singletonList(getName()),
            internalLockLeaseTime,
            getLockName(threadId));
    }
    
    void cancelExpirationRenewal(Long threadId) {
        ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (task == null) {
            return;
        }
    
        if (threadId != null) {
            task.removeThreadId(threadId);
        }
    
        if (threadId == null || task.hasNoThreads()) {
            task.getTimeout().cancel();
            EXPIRATION_RENEWAL_MAP.remove(getEntryName());
        }
    }

③Redis INCR等原子操作？

Redis 提供了许多原子操作命令，如 INCR、DECR、INCRBY 等，这些命令可以确保在高并发环境下数据的一致性。

1. INCR 命令

INCR 命令用于将键中的整数值递增 1。如果键不存在，则创建键并将值设置为 1。如果键的值不是整数，则返回错误。

    Jedis jedis = new Jedis("localhost");
    String key = "counter";
    jedis.incr(key); // 将 key 的值递增 1

2. INCRBY 命令

INCRBY 命令用于将键中的整数值递增指定的增量值。如果键不存在，则创建键并将值设置为增量值。

    int increment = 5;
    jedis.incrby(key, increment); // 将 key 的值递增 5


