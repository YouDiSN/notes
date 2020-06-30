# REDIS源码22-LRU、LFU

在EventLoop(2)中分析了Expire的基本流程，主要就是分为slow和fast两种，然后从RedisDB中获取20个key看是否过时需要清理，如果20个里面有5个以上过时了就继续一次循环。

不过这是常规情况下的过期清理，Redis由于是一个基于内存的数据库，所以是有可能把所有内存全部耗尽的，从而触发频繁的内存swap，这样对Redis的性能影响是非常非常大的。因此Redis可以配置最大可使用的内存上限，同时也提供了多种可选的内存淘汰策略，当然这些策略只有在内存将要耗尽的时候才会生效，平时的还是依靠惰性删除和定时清理来实现的。

## 可选择的内存淘汰模型

```c
#define MAXMEMORY_VOLATILE_LRU ((0<<8)|MAXMEMORY_FLAG_LRU)
#define MAXMEMORY_VOLATILE_LFU ((1<<8)|MAXMEMORY_FLAG_LFU)
#define MAXMEMORY_VOLATILE_TTL (2<<8)
#define MAXMEMORY_VOLATILE_RANDOM (3<<8)
#define MAXMEMORY_ALLKEYS_LRU ((4<<8)|MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_LFU ((5<<8)|MAXMEMORY_FLAG_LFU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_RANDOM ((6<<8)|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_NO_EVICTION (7<<8)
```

可以看到就有这些的情况，

- LRU就是淘汰最长时间没有使用
- LFU就是使用最不多的情况
- TTL就是根据配置了超时时间的key中，最接近过期时间的优先淘汰
- VOLATILE就是只针对配置了超时时间的key
- ALLKEYS就是左右的key都要参与淘汰
- NO_EVICTION也是默认的策略，就是当内存使用达到一个极限的时候，变成只读模式，写入数据统统失败。

这么些个策略不一一分析了，就挑选MAXMEMORY_ALLKEYS_LRU、MAXMEMORY_ALLKEYS_LFU和MAXMEMORY_ALLKEYS_RANDOM这三个来分析一下。

##ALLKEYS_LRU

说到LRU，最最经典的实现就是Java的LinkedHashMap，通过HashMap+双向链表的方式来实现，另外维护一个链表，当节点被修改或读取等操作后把自己从原来的链表节点移除然后放到头节点这样的做法。

但是Redis并没有这样的实现，而是使用了自己的实现。

```c
typedef struct redisObject {
  unsigned type:4;
  unsigned encoding:4;
  unsigned lru:LRU_BITS; /* 24个bit
  												* LRU time (relative to global lru_clock) or
                          * LFU data (least significant 8 bits frequency
                          * and most significant 16 bits access time). */
  int refcount;
  void *ptr;
} robj;
```

可以看到在RedisObject这个结构体中，有一个专门的字段lru来存储LRU和LFU的信息，使用了24个bit。着24个bit在LRU模式下表示毫秒数，在LFU模式下表示16个bit的访问时间和8个bit的访问频率。

### 何时更新lru

那么知道每一个RedisObject上都有lru后，第一个问题就是什么时候更新的lru数据，以及在回收时如何使用的lru。

先看在哪里更新，就是从一个redisDB中根据key寻找对应的value时，就会更新这个字段

```c
robj *lookupKey(redisDb *db, robj *key, int flags) {
  dictEntry *de = dictFind(db->dict,key->ptr);
  // 如果dictEntry存在
  if (de) {
    robj *val = dictGetVal(de);

		// 没有RDB和AOF子进程时修改，因为子进程会共用父进程的空间
    if (server.rdb_child_pid == -1 &&
        server.aof_child_pid == -1 &&
        !(flags & LOOKUP_NOTOUCH))
    {
      if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        updateLFU(val);
      } else {
        // 如果是LRU模式，就直接更新一下时钟
        val->lru = LRU_CLOCK();
      }
    }
    return val;
  } else {
    return NULL;
  }
}

void updateLFU(robj *val) {
  // 这个就是前24个bit，最右边8个bit就是一个计数，这个方法就是获取这个计数
  unsigned long counter = LFUDecrAndReturn(val);
  // 否则使用一个随机更新的算法，算法如下
  counter = LFULogIncr(counter);
  // 分钟级时间戳，取时间戳最右边的16位放在lru的最左边16位，然后counter放在lru的最右边
  val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}

uint8_t LFULogIncr(uint8_t counter) {
  if (counter == 255) return 255;
  double r = (double)rand()/RAND_MAX;
  double baseval = counter - LFU_INIT_VAL;
  if (baseval < 0) baseval = 0;
  double p = 1.0/(baseval*server.lfu_log_factor+1);
  if (r < p) counter++;
  return counter;
}
```

### 如何使用lru字段

上面看到了每次从RedisDB中获取Key对应的RedisObject的时候更新，那么什么时候会判断是不是需要进行内存回收呢？

主要就是在处理每一个RedisClient发过来的命令之前，会先判断是不是需要进行内存回收，方法就是志气啊你分析过的processCommand方法。

```c
int processCommand(client *c) {
	// 省略

	// 寻找Redis命令
  c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
	// 省略其他逻辑

  // 如果配置了maxmemory，同时没有因为lua脚本超时
  if (server.maxmemory && !server.lua_timedout) {
    // 尝试freeMemoryIfNeededAndSafe
    int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
    if (server.current_client == NULL) return C_ERR;

    // 如果内存不够用了，就返回错误
    if (out_of_memory &&
        (c->cmd->flags & CMD_DENYOOM ||
         (c->flags & CLIENT_MULTI && c->cmd->proc != execCommand))) {
      flagTransaction(c);
      addReply(c, shared.oomerr);
      return C_OK;
    }
  }

  // 执行命令
  if (c->flags & CLIENT_MULTI &&
      c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
      c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
  {
    queueMultiCommand(c);
    addReply(c,shared.queued);
  } else {
    call(c,CMD_CALL_FULL);
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
      handleClientsBlockedOnKeys();
  }
  return C_OK;
}
```

可以看到核心方法就是**freeMemoryIfNeededAndSafe**，

```c
int freeMemoryIfNeededAndSafe(void) {
  if (server.lua_timedout || server.loading) return C_OK;
  return freeMemoryIfNeeded();
}
```

核心实现就是**freeMemoryIfNeeded**，不过这个方法非常的长，因为这个方法里面通过if else实现了多种回收策略的代码。由于我们由于只分析LRU和LFU，所以只挑选有意义的部分来分析，其他的就暂时省略。

```c
// 上面省略了一些代码，就是只要目标释放的内存和已经释放的内存不一致，就执行这个循环
// 不过这里的计算的时候，并不是直接拿用了多少内存来计算的，有些Redis自己比需要使用的内存需要预先扣除，比如集群相关的、AOF Buffer等会在计算前就减掉
while (mem_freed < mem_tofree) {
  int j, k, i, keys_freed = 0;
  static unsigned int next_db = 0;
  sds bestkey = NULL;
  int bestdbid;
  redisDb *db;
  dict *dict;
  dictEntry *de;

  // LFU、LRU和VOLATILE_TTL共享一组实现
  if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
      server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
  {
    struct evictionPoolEntry *pool = EvictionPoolLRU;

    // bestKey的意思就是，因为LFU、LRU和TTL都有最不常使用、使用次数最少和距离过期最近的
    // 所以既然有一个极值，就要找到那个极值
    while(bestkey == NULL) {
      unsigned long total_keys = 0, keys;

			// 这个就是一个Redis中有多个DB，遍历每一个DB
      for (i = 0; i < server.dbnum; i++) {
        db = server.db+i;
        // 这个就是根据配置的是不是ALLKEYS，来判断抽样的范围
        // 如果是ALLKEYS就是所有的数据，所以用db->dict，如果是VOLATILE，就只抽样配置了过期时间的，所以就是db->expires
        dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
          db->dict : db->expires;
        // 如果挑选的这个dict里面有数据
        if ((keys = dictSize(dict)) != 0) {
          // 抽样获取一部分数据
          // dict是抽样的数据范围，db->dict是全量数据，pool就是抽样后的数据放在的地方
          evictionPoolPopulate(i, dict, db->dict, pool);
          total_keys += keys;
        }
      }
      // 一个key都没找到就结束
      if (!total_keys) break; /* No keys to evict. */

      // 遍历抽样出来的数据
      for (k = EVPOOL_SIZE-1; k >= 0; k--) {
        if (pool[k].key == NULL) continue;
        // 这个候选数据的DB id
        bestdbid = pool[k].dbid;

        // 如果是ALLKEYS就从db->dict找
        // 否则就从db->expires找
        // 找到dictEntry
        if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
          de = dictFind(server.db[pool[k].dbid].dict,
                        pool[k].key);
        } else {
          de = dictFind(server.db[pool[k].dbid].expires,
                        pool[k].key);
        }

        /* Remove the entry from the pool. */
        if (pool[k].key != pool[k].cached)
          sdsfree(pool[k].key);
        pool[k].key = NULL;
        pool[k].idle = 0;

        // 从pool中找到一个最有删除价值key，注意这里的break只是退出从pool中找bestkey的流程，当然这个break退出for循环之后，while循环也会因为找到bestkey自动退出
        if (de) {
          bestkey = dictGetKey(de);
          break;
        } else {
          /* Ghost... Iterate again. */
        }
      }
    }
  }
  // 省略其他回收策略

  // 找到了最佳候选key
  if (bestkey) {
    // 找到db
    db = server.db+bestdbid;
    // 创建一个类似temp RedisKey
    robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
    // 这个就是写AOF以及集群通知等
    propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
    // 计算释放了多少内存，后面会减一下
    delta = (long long) zmalloc_used_memory();
    // 抽样，不管
    latencyStartMonitor(eviction_latency);
    // 删除
    if (server.lazyfree_lazy_eviction)
      dbAsyncDelete(db,keyobj);
    else
      dbSyncDelete(db,keyobj);
    // 抽样，不管
    latencyEndMonitor(eviction_latency);
    latencyAddSampleIfNeeded("eviction-del",eviction_latency);
    latencyRemoveNestedEvent(latency,eviction_latency);
    // 释放内存之前算一下，释放完了再算一下，这样就知道释放了多少内存了
    delta -= (long long) zmalloc_used_memory();
    // 统计
    mem_freed += delta;
    server.stat_evictedkeys++;
    // 事件
    notifyKeyspaceEvent(NOTIFY_EVICTED, "evicted",
                        keyobj, db->id);
    // 引用计数-1
    decrRefCount(keyobj);
    统计
    keys_freed++;

    // 这里是集群处理，之所以要处理集群是因为之前不是内存太大，所以很可能导致集群之间的通讯也被阻塞了，所以这里要尽快去尝试恢复集群之间的通信，具体方法到后面分析Redis集群再分析
    if (slaves) flushSlavesOutputBuffers();

    // 统计，以及判断是不是需要退出大循环
    if (server.lazyfree_lazy_eviction && !(keys_freed % 16)) {
      if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
        /* Let's satisfy our stop condition. */
        mem_freed = mem_tofree;
      }
    }
  }

  // 如果折腾了半天没有内存被释放，那么就直接去报错的地方
  if (!keys_freed) {
    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);
    goto cant_free; /* nothing to free... */
  }
}
```

上面这段代码比较长，但是主要的流程还是比较清楚的，从每个DB中抽样一些数据，然后找到最有价值的数据，然后释放。

不过这里要注意的一个点就是抽样数据的方法**evictionPoolPopulate**，Redis的LRU、LFU和Java的实现不同的就是Java是整个HashMap全局找到最最精确的那个数据，但是Redis没有采用最最精确的那个方式来删除那个极值，而是每次从所有数据中抽样一部分，找到一个相对最优的来删除。

所以这里就是一个权衡和取舍，如果要做到100%的精确就需维护更多的数据和花费更多的性能，因此Redis采用了一个不那么精确但是依然有效的方式来实现，这个的主要原因也是Redis本身就是一个高性能的DB，所以在精确度和性能上做了一个更偏向性能的取舍。当然采样的算法本身也是可以调节的，比如可以通过配置采样的数量来找提高相对的精确度。

下面就分析一下**evictionPoolPopulate**这个方法。

```c
void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
  int j, k, count;
  // 这个就是抽样数据的数组，maxmemory_samples = 5，默认抽样5个数据
  dictEntry *samples[server.maxmemory_samples];

  // 随机获取一些数据，从sampledict中随机抽样maxmemory_samples个数据，放到samples中
  count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
  for (j = 0; j < count; j++) {
    unsigned long long idle;
    sds key;
    robj *o;
    dictEntry *de;

    de = samples[j];
    key = dictGetKey(de);

    // 因为如果不是VOLATILE_TTL，就说嘛是LFU和LRU，那么就需要从RedisObject中读取lru字段，所以这里就要拿到o
    if (server.maxmemory_policy != MAXMEMORY_VOLATILE_TTL) {
      if (sampledict != keydict) de = dictFind(keydict, key);
      o = dictGetVal(de);
    }

    // 根据不同的配置类型，来决定这个数据是不是最需要被删除的
    
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
      // 这个就是直接拿时间戳
      idle = estimateObjectIdleTime(o);
    } else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
      // LFU是最少使用的，又因为用了8个bit，所以一定小于255
      idle = 255-LFUDecrAndReturn(o);
    } else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
      // VOLATILE_TTL这里直接转成long有点没看懂，不过意思能明白
      idle = ULLONG_MAX - (long)dictGetVal(de);
    } else {
      serverPanic("Unknown eviction policy in evictionPoolPopulate()");
    }

    // 把找到的这个数据插入pool中
    
    // 这个循环就是看看我当前找到的这个是不是比原有的pool里面的数据更符合删除的标准，如果是的话就替换，否则就拉倒
    k = 0;
    while (k < EVPOOL_SIZE &&
           pool[k].key &&
           pool[k].idle < idle) k++;
    // 下面这段if判断就是在真正执行插入逻辑之前，先做一些预处理
    if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
      continue;
    } else if (k < EVPOOL_SIZE && pool[k].key == NULL) {
      /* Inserting into empty position. No setup needed before insert. */
    } else {
      // 这段就是要往数组中间插入元素，所以k之后的元素向后挪一格
      if (pool[EVPOOL_SIZE-1].key == NULL) {
        // 这个cached是复用分配的内存的，后面有分析
        sds cached = pool[EVPOOL_SIZE-1].cached;
        memmove(pool+k+1,pool+k,
                sizeof(pool[0])*(EVPOOL_SIZE-k-1));
        pool[k].cached = cached;
      } else {
        // 这段就是后面塞不下了，就把最左边的给踢掉一个，因为这段代码是要找到bestkey，顺序是从左向右递增，所以最右边的数据是最需要删除的，最左边的就不要了
        k--;
        sds cached = pool[0].cached; /* Save SDS before overwriting. */
        if (pool[0].key != pool[0].cached) sdsfree(pool[0].key);
        memmove(pool,pool+1,sizeof(pool[0])*k);
        pool[k].cached = cached;
      }
    }

    int klen = sdslen(key);
    // emmmmmmm.....这里是复用sds
    if (klen > EVPOOL_CACHED_SDS_SIZE) {
      pool[k].key = sdsdup(key);
    } else {
      // cached就是为了复用sds这块内存，否则每一个key都要分配一个sds也是有一定的开销的，所以只要不是很大就缓存起来，下次复用
      // 感觉理解这段代码的时候不要管cached，就当没看见其实还挺好理解的，要是过分纠结cached就不好了
      memcpy(pool[k].cached,key,klen+1);
      sdssetlen(pool[k].cached,klen);
      pool[k].key = pool[k].cached;
    }
    pool[k].idle = idle;
    pool[k].dbid = dbid;
  }
}
```

所以整个流程就非常清晰了，通过抽样找出一些候选，然后找到一个最佳候选，然后删除，删除之后记录AOF日志等后续操作，然后再根据清理的内存空间来判断是不是要继续删除下一个。