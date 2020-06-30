#REDIS源码14-Redis EventLoop(2)

## Expire的处理

上一节从总体的流程分析了EventLoop各个阶段都做了一些什么事情，所以这里需要进一步分析在EventLoop的过程中，一些细节的事情。

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
	// 省略

  // expire
  if (server.active_expire_enabled && server.masterhost == NULL)
    activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

  // 省略
}
```

这一节具体分析expire的原理。

##Expire原理

Expire就是Redis Key的过期清理工作，因为Redis是一个基于内存的KV数据库，所以及时的清理已经过期的Key对于高效的使用内存是非常重要的。

同时又由于Redis是一个单线程的模型，所以如果每次都遍历所有Key并找出已经过期的Key来清理就会非常占用时间；而且这是在每一次EventLoop中做的，所以耗时会变的不可接受。因此如何设计出高效的清理过期的数据的算法就变的极为重要。

Expire分为Fast和Slow两种模式。

### Fast模式

在Fast模式下，整个Expire的过程不会超过*ACTIVE_EXPIRE_CYCLE_FAST_DURATION = 1ms*的时间。当然这个时间可以配置的。同时在*2 \* ACTIVE_EXPIRE_CYCLE_FAST_DURATION = 2ms*的时间内，是不会重复执行Fast模式的Expire的。

### Slow模式

在Slow模式下，每一次的超时时间都是需要动态计算的，计算公式为：*1000000\*(ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC = 25)/server.hz/100*，大致的意思是根据期望的最高CPU负载25%和server.hz（这个就是一秒内执行定时任务的次数）来计算一个动态的超时时间。言外之意其实就是如果在这一秒内执行的定时任务越多，那么分配到的Expire的执行时间也就会越少。

### 源码分析

```c
void activeExpireCycle(int type) {
  /* This function has some global state in order to continue the work
     * incrementally across calls. */
  static unsigned int current_db = 0; /* Last DB tested. */
  static int timelimit_exit = 0;      /* Time limit hit in previous call? */
  static long long last_fast_cycle = 0; /* When last fast cycle ran. */

  int j, iteration = 0;
  int dbs_per_call = CRON_DBS_PER_CALL;
  long long start = ustime(), timelimit, elapsed;

  /* When clients are paused the dataset should be static not just from the
     * POV of clients not being able to write, but also from the POV of
     * expires and evictions of keys not being performed. */
  if (clientsArePaused()) return;

  if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
    if (!timelimit_exit) return;
    // 这里就是如果上一次Fast模式的Expire距离当前时间太近，就直接返回
    if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
    last_fast_cycle = start;
  }

  /* We usually should test CRON_DBS_PER_CALL per iteration, with
     * two exceptions:
     *
     * 1) Don't test more DBs than we have.
     * 2) If last time we hit the time limit, we want to scan all DBs
     * in this iteration, as there is work to do in some DB and we don't want
     * expired keys to use memory for too much time. */
  if (dbs_per_call > server.dbnum || timelimit_exit)
    dbs_per_call = server.dbnum;

  // 这里就是计算timelimit，先用Slow模式计算一下，如果是Fast就替换。。其实感觉可以用一个if/else来代替会好一点。。。
  timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
  timelimit_exit = 0;
  if (timelimit <= 0) timelimit = 1;

  if (type == ACTIVE_EXPIRE_CYCLE_FAST)
    timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */

  // 这个就是看全局过期了多少数据
  long total_sampled = 0;
  long total_expired = 0;

  for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
    int expired;
    // 获取到db
    redisDb *db = server.db+(current_db % server.dbnum);
    current_db++;

    // 循环操作
    // 如果在一次循环结束后有超过25%的数据过期了，就在执行一次循环
    do {
      unsigned long num, slots;
      long long now, ttl_sum;
      int ttl_samples;
      iteration++;

      // 如果没有数据需要过期，就直接返回
      if ((num = dictSize(db->expires)) == 0) {
        db->avg_ttl = 0;
        break;
      }
      // slots就是总共有多少数据，expires是一个dict，所以有有两个table的size加一下就是结果
      slots = dictSlots(db->expires);
      // 当前时间戳
      now = mstime();

      // 如果当前DB的使用率不及1%，就不用费事过期数据了，因为这个dict会被resize变成一个小的hashtable
      if (num && slots > DICT_HT_INITIAL_SIZE &&
          (num*100/slots < 1)) break;

      /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. */
      expired = 0;
      ttl_sum = 0;
      ttl_samples = 0;

      // 如果数据量大于20，就设置为20
      // 这个num就是上面的expires的大小
      if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
        num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;

      // 遍历这20个数据
      while (num--) {
        dictEntry *de;
        long long ttl;

        // de就是从expires中随机拿出来的一个节点
        if ((de = dictGetRandomKey(db->expires)) == NULL) break;
        // 拿到ttl的时间
        ttl = dictGetSignedIntegerVal(de)-now;
        // 尝试执行expire操作
        if (activeExpireCycleTryExpire(db,de,now)) expired++;
        // 如果确实已经过期了，就更新一下统计信息，这个地方的统计信息是计算过期的key平均超出了多少时间才被执行到过期的。
        if (ttl > 0) {
          ttl_sum += ttl;
          ttl_samples++;
        }
        // 取样数+1
        total_sampled++;
      }
      // 总过期数量更新
      total_expired += expired;

      // 如果有取样到数据
      if (ttl_samples) {
        // 这个就是平均超出时间
        long long avg_ttl = ttl_sum/ttl_samples;

        if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
        // 更新avg_ttl
        // 之前的过期时间 * 98% + 本次的过期时间 * 2%
        db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
      }

      // 这个地方的判断就是如果真的有很多很多key需要过期，也不能一直阻塞在这个expire流程上，所以需要有一个地方尝试跳出循环，这里就是每循环16次就判断一下，如果时间到了，就赶紧结束
      if ((iteration & 0xf) == 0) {
        elapsed = ustime()-start;
        if (elapsed > timelimit) {
          timelimit_exit = 1;
          server.stat_expired_time_cap_reached_count++;
          break;
        }
      }
		// 这里就是取样的20个如果过期的数据有5个，那么就继续循环一次
    } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
  }

  elapsed = ustime()-start;
  latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);

  /* Update our estimate of keys existing but yet to be expired.
     * Running average with this sample accounting for 5%. */
  double current_perc;
  if (total_sampled) {
    current_perc = (double)total_expired/total_sampled;
  } else
    current_perc = 0;
  server.stat_expired_stale_perc = (current_perc*0.05)+
    (server.stat_expired_stale_perc*0.95);
}
```

整体的流程应该还是非常清楚的，不过有一个方法需要关注，就是尝试expire的**activeExpireCycleTryExpire**，

下面具体分析一下这个方法：

```c
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
  // 先拿到过期时间
  long long t = dictGetSignedIntegerVal(de);
  // 如果当前的时间大于过期时间，就代表这个数据以及过期了
  if (now > t) {
    // 从dict中拿到key
    sds key = dictGetKey(de);
    // 创建一个只有key的RedisObject
    robj *keyobj = createStringObject(key,sdslen(key));

    // AOF日志、以及集群的话通知slave节点等
    propagateExpire(db,keyobj,server.lazyfree_lazy_expire);
    // 下面是同步删除和异步删除
    if (server.lazyfree_lazy_expire)
      dbAsyncDelete(db,keyobj);
    else
      dbSyncDelete(db,keyobj);
    // 一个事件，给module用的，我们可以忽略
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
                        "expired",keyobj,db->id);
    // 引用-1
    decrRefCount(keyobj);
    server.stat_expiredkeys++;
    return 1;
  } else {
    return 0;
  }
}
```

下面就分析一下同步和异步删除的具体做了什么事情，不过默认是不开启异步的。

因为同步和异步往往是同步更简单，所以我们先分析同步删除：

```c
int dbSyncDelete(redisDb *db, robj *key) {
  // 如果expires里面就从expires里删除，删除的方式就是直接从expires这个dict中移除
  if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
  // 然后再从数据dict中删除，如果删除成功就看是不是作为集群需要通知其他节点
  if (dictDelete(db->dict,key->ptr) == DICT_OK) {
    if (server.cluster_enabled) slotToKeyDel(key);
    return 1;
  } else {
    return 0;
  }
}
```

同步的删除流程非常简单，就是直接把数据从对应的dict中删除即可。

那么再看一下异步删除：

```c
int dbAsyncDelete(redisDb *db, robj *key) {
  // expires的删除和同步是一样的
  if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

  // unlink的意思是就是吧这个dictEntry先从hashtable中拿出来，但是不要急着释放内存
  dictEntry *de = dictUnlink(db->dict,key->ptr);
  if (de) {
    // 获取数据
    robj *val = dictGetVal(de);
    // 这个effort就是释放这个val的内存需要多大的功夫
    // 比如如果只是一个sds，那么就是1，如果是quicklist那就是它的len，如果是dict，那么就是dict的size等等
    size_t free_effort = lazyfreeGetFreeEffort(val);

    // 这个地方就是如果effort > 64，同时这个val的引用 = 1就说明这是一个比较大的对象
    // 比较大的对象就需要放到一个free_list中，然后等待后台线程来清理
    if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
      // lazyfree_objects这个值原子+1
      atomicIncr(lazyfree_objects,1);
      // 然后通过pthread的锁，把这次释放变成一个task，放到bio_jobs这个链表上，另一个线程回来处理这个list的数据。
      bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
      // 设置空的值
      dictSetVal(db->dict,de,NULL);
    }
  }

  // 这个就是把dictEntry的内存先释放出来
  if (de) {
    dictFreeUnlinkedEntry(db->dict,de);
    // 然后通知集群
    if (server.cluster_enabled) slotToKeyDel(key);
    return 1;
  } else {
    return 0;
  }
}
```

关于异步处理其实就是如果处理比较复杂的就放到后台线程异步处理，如果处理起来不那么复杂的，也是直接处理掉，并不是无脑异步。

## dict.expires何时填充

前面的源码分析可以看到在Expire的过程中大量用到了dict.expires，那么这个地方的数据是何时填充进去的？

使用Redis expire指令的时候会把对应的数据加入到expires这个dict中。

```c
void setExpire(client *c, redisDb *db, robj *key, long long when) {
  dictEntry *kde, *de;

  kde = dictFind(db->dict,key->ptr);
  serverAssertWithInfo(NULL,key,kde != NULL);
  // 只把key加入到expires即可
  de = dictAddOrFind(db->expires,dictGetKey(kde));
  // 这个就是设置过期的时间
  dictSetSignedIntegerVal(de,when);

  // 处理集群
  int writable_slave = server.masterhost && server.repl_slave_ro == 0;
  if (c && writable_slave && !(c->flags & CLIENT_MASTER))
    rememberSlaveKeyWithExpire(db,key);
}
```

## Redis的过期策略

Redis的过期策略主要是惰性删除和定时检查。

惰性删除就是当获取到一个key的时候去检查一下他有没有过期，这个在源码主要体现在**dictFindKey**的时候，会先调用**expireIfNeeded**方法。

```c
robj *lookupKeyWrite(redisDb *db, robj *key) {
  expireIfNeeded(db,key);
  return lookupKey(db,key,LOOKUP_NONE);
}

robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {
  robj *val;

  if (expireIfNeeded(db,key) == 1) {
    // 省略
  }
  val = lookupKey(db,key,flags);
  if (val == NULL)
    server.stat_keyspace_misses++;
  else
    server.stat_keyspace_hits++;
  return val;
}
```

那么定时删除就是前面分析的EventLoop中定时去看是否需要Expire。