#REDIS源码15-Redis EventLoop(3)

##Redis的定时任务

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
  // 省略

  // 处理定时任务
  if (flags & AE_TIME_EVENTS)
    processed += processTimeEvents(eventLoop);

  return processed;
}

static int processTimeEvents(aeEventLoop *eventLoop) {
  int processed = 0;
  aeTimeEvent *te;
  long long maxId;
  time_t now = time(NULL);

  // 这个就是校准系统时钟
  // lastTime就是上一次执行的时间戳，如果当前时间戳比上一次执行的时间戳要更小，就说明时钟被回拨了，所以就让所有的定时任务都执行一下，因为实践证明，定时任务提前执行比延迟执行要更好。
  // 不过通常来说这块逻辑应该不会执行到才对，生产的时钟讲道理不可以随便乱动的
  if (now < eventLoop->lastTime) {
    te = eventLoop->timeEventHead;
    while(te) {
      te->when_sec = 0;
      te = te->next;
    }
  }
  // 更新时间戳
  eventLoop->lastTime = now;

  // 定时任务链表
  te = eventLoop->timeEventHead;
  maxId = eventLoop->timeEventNextId-1;
  // 只要链表上还有任务可以执行
  while(te) {
    long now_sec, now_ms;
    long long id;

    // 如果是DELETED类型时间，就把这个事件从链表上移除，然后调用finalizerProc方法，这个就是一个函数指针，具体可以指向不同的函数。
    // 目前从源码中看，这个类型的事件直接设置只有在module部分才会做这件事情，所以对我们来说可以直接忽略
    // 还有一种情况就是执行完的事件会被标记成AE_DELETED_EVENT_ID
    if (te->id == AE_DELETED_EVENT_ID) {
      aeTimeEvent *next = te->next;
      if (te->prev)
        te->prev->next = te->next;
      else
        eventLoop->timeEventHead = te->next;
      if (te->next)
        te->next->prev = te->prev;
      if (te->finalizerProc)
        te->finalizerProc(eventLoop, te->clientData);
      zfree(te);
      te = next;
      continue;
    }

    // 这个if判断的目的是，在定时任务的执行过程中，可能本身还会创建新的定时任务，所以本次执行要跳过这些任务
    // 不过这个是一个防御性代码，目前是没有用的，因为新的任务都是放在head的，而这个链表是从head开始遍历的，所以这个te指针是不可能指向新创建的TimeEvent的。
    if (te->id > maxId) {
      te = te->next;
      continue;
    }
    // 计算时间戳，一个是当前的秒数，一个是当前的毫秒数
    aeGetTime(&now_sec, &now_ms);
    // 这个就是看是不是到执行时间了
    if (now_sec > te->when_sec ||
        (now_sec == te->when_sec && now_ms >= te->when_ms))
    {
      int retval;

      id = te->id;
      // 执行任务
      retval = te->timeProc(eventLoop, id, te->clientData);
      processed++;
      // 这个的意思是这个任务下次还要继续执行，间隔的时间为retval
      if (retval != AE_NOMORE) {
        // 所以既然下次还要执行，就在时间戳上增加
        aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
      } else {
        // 标记删除
        te->id = AE_DELETED_EVENT_ID;
      }
    }
    // 下一个任务
    te = te->next;
  }
  // 处理了多少任务
  return processed;
}
```

所以可以看到Redis的定时处理其实就是一个链表不停的循环，然后和当前时间比对，如果可以执行就执行。

> 因为是在EventLoop中执行的，如何确保能尽可能在定时任务需要执行的瞬间来执行？
>
> - 第一个就是在**aeApiPoll**方法阻塞的时候计算超时时间，这个超时时间就是尽可能到下一个定时任务的执行瞬间。
> - 其次在一些可能耗时比较长的任务中添加时间限制，比如expire就有添加时间限制，防止执行的时间过长
> - 还有一个就是Redis自己的命令需要轻量化执行，这也是为什么Redis往往要求不可以执行o(n)复杂度的命令，原因也在于此。

##典型的定时任务

ServerCron，Redis中最最典型的定时任务就是ServerCron了。这个方法又是一个又长又臭的方法，里面做了好多好多事情，有些需要认真研究一下，有些可以和其他主题合并，有些则直接跳过即可。

这个方法250+行，所以仅选取重点来分析。

首先要说明**run_with_period**这个方法，因为这个**serverCron**是在定时任务中触发的，所以里面的所有任务执行的时间都不能太频繁，否则对资源的消耗是比较大的。所以就有了这个**run_with_period**方法的目的就是为了限制在一定时间内的执行频率。

```c
// run_with_period是一个macro，还是根据server.hz来计算的。
#define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
```

**serverCron**方法实在太长了，所以就分段并且挑选一些好理解的内容的分析。

 ```c
run_with_period(100) {
  // 这个方法就是采集样本做统计
  // 执行命令的样本、输入的bytes和输出的bytes进行统计
  trackInstantaneousMetric(STATS_METRIC_COMMAND,
                           server.stat_numcommands);
  trackInstantaneousMetric(STATS_METRIC_NET_INPUT,
                           server.stat_net_input_bytes);
  trackInstantaneousMetric(STATS_METRIC_NET_OUTPUT,
                           server.stat_net_output_bytes);
}
 ```

   除了这段代码，**serverCron**还有好多处处理统计信息的地方，比如下面这个地方

```c
// 这段代码也是统计信息的手机，主要是内存相关，包括lua所需要使用的内存
run_with_period(100) {
  server.cron_malloc_stats.process_rss = zmalloc_get_rss();
  server.cron_malloc_stats.zmalloc_used = zmalloc_used_memory();
  zmalloc_get_allocator_info(&server.cron_malloc_stats.allocator_allocated,
                             &server.cron_malloc_stats.allocator_active,
                             &server.cron_malloc_stats.allocator_resident);
  if (!server.cron_malloc_stats.allocator_resident) {
    size_t lua_memory = lua_gc(server.lua,LUA_GCCOUNT,0)*1024LL;
    server.cron_malloc_stats.allocator_resident = server.cron_malloc_stats.process_rss - lua_memory;
  }
  if (!server.cron_malloc_stats.allocator_active)
    server.cron_malloc_stats.allocator_active = server.cron_malloc_stats.allocator_resident;
  if (!server.cron_malloc_stats.allocator_allocated)
    server.cron_malloc_stats.allocator_allocated = server.cron_malloc_stats.zmalloc_used;
}
```

再往后会记录一些日志，比如下面：

```c
run_with_period(5000) {
  for (j = 0; j < server.dbnum; j++) {
    long long size, used, vkeys;

    size = dictSlots(server.db[j].dict);
    used = dictSize(server.db[j].dict);
    vkeys = dictSize(server.db[j].expires);
    // 只要db里面有内容，就打印一些日志相关信息
    if (used || vkeys) {
      serverLog(LL_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
      /* dictPrintStats(server.dict); */
    }
  }
}

// 不是sentinel模式下
if (!server.sentinel_mode) {
  run_with_period(5000) {
    // 这里会记录有多少redis cients链接了当前server，以及有多少个主备和使用了多少的内存等
    serverLog(LL_VERBOSE,
              "%lu clients connected (%lu replicas), %zu bytes in use",
              listLength(server.clients)-listLength(server.slaves),
              listLength(server.slaves),
              zmalloc_used_memory());
  }
}
```

再后面是针对所有Redis Clients的一个类似监控处理的方法：

```c
clientsCron();

void clientsCron(void) {
	// 找到所有的clients数量
  int numclients = listLength(server.clients);
  // 为了避免clients太多，一个个遍历的时间太长，所以还是需要对遍历的数量做一定的限制
  int iterations = numclients/server.hz;
  mstime_t now = mstime();

	// 虽然希望少一点，但是也不要太少，不然这个方法执行的意义就不大了
  if (iterations < CLIENTS_CRON_MIN_ITERATIONS)
    iterations = (numclients < CLIENTS_CRON_MIN_ITERATIONS) ?
    numclients : CLIENTS_CRON_MIN_ITERATIONS;

  while(listLength(server.clients) && iterations--) {
    client *c;
    listNode *head;

		// 把server.clients这个链表的尾节点移到头上来
    listRotate(server.clients);
    // 然后获取新的头节点
    // 通过这种每次迭代把尾巴节点的clients放到头上的方式来检查，其实就是挨个迭代。但是这样的方式好处就是考虑到Redis不希望每次遍历所有的clients，所以用这样的方式来减少遍历的数量。但是如果传统的方式还要记录下标，同时clients存在不停的动态增加和动态减少的过程，会导致有些clients没有被及时处理到。所以这里用链表的方式非常巧妙的处理了逐个迭代过程中不用记录下标但是又可以很好的完成迭代任务的
    // 虽然只是一个简单的链表结构，但是这样的使用非常的巧妙，值得好好体会
    head = listFirst(server.clients);
    c = listNodeValue(head);
    // 这个就是看当前clients是不是超时了，上一次的请求时间是不是超出了配置的maxidletime
    // 如果是的话就释放clients，注意这个方法里面freeClient，所以外面就不用再处理需要释放的场景了
    // 还有一些block的命令，比如bpop，看这样的阻塞命令是不是有超时，也是在里面做的
    if (clientsCronHandleTimeout(c,now)) continue;
    // client QueryBuffer就是用来接受客户端发送过来的数据的，其实就是一个sds，但是最最早学习sds的时候就知道sds后面有一块未使用部分的，所以这里尝试resize一下，防止大块的内存没有很好的利用造成浪费
    if (clientsCronResizeQueryBuffer(c)) continue;
    // 这个是追踪是不是这个clients在短时间内使用了大量的内存
    // 其实就是看InBuffer和OutBuffer，然后放到一个桶里
    // 然后短时间内其实就是最近的（Recently），采用的方式就是拿时间戳和桶的长度做&预算，然后和原来的比较，如果比原来大，就替换掉，否则就拉倒
    // 这个方法不会freeClient，只是一个追踪
    if (clientsCronTrackExpansiveClients(c)) continue;
  }
}
```

上面这个方法是针对所有链接到Redis的clients进行的一些定时处理，那么自然还有Redis针对自己的数据进行的处理，那就是**databasesCron**方法

```c
void databasesCron(void) {
  // 尝试开启activeExpireCycle
  if (server.active_expire_enabled) {
    // master为NULL的意思是，如果当前节点是master
    if (server.masterhost == NULL) {
      // 使用SLOW的方式来进行expire数据
      // expire的方式之前也已经分析过了，就是看expires这个dict里面的数据，然后随机抽取不超过20个，20%的数据过期了就继续下一批等，具体可以复习之前的内容
      activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
    } else {
      // 这个就是如果当前节点是slave
      expireSlaveKeys();
    }
  }

  // 没有rdb进程id和aof进程id的情况下，尝试rehash db
  // 这个也很好理解，否则一边在写文件，一边在改数据，就会出问题
  if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
    static unsigned int resize_db = 0;
    static unsigned int rehash_db = 0;
    int dbs_per_call = CRON_DBS_PER_CALL;
    int j;

    /* Don't test more DBs than we have. */
    if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;

    for (j = 0; j < dbs_per_call; j++) {
      // 尝试resize db
      // 就是dict太小或太大了，出发一次扩容或缩容
      tryResizeHashTables(resize_db % server.dbnum);
      resize_db++;
    }

    if (server.activerehashing) {
      for (j = 0; j < dbs_per_call; j++) {
        // 如果之前rehash做了一半，继续往下做
        int work_done = incrementallyRehash(rehash_db);
        if (work_done) {
          /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
          break;
        } else {
          /* If this db didn't need rehash, we'll try the next one. */
          rehash_db++;
          rehash_db %= server.dbnum;
        }
      }
    }
  }
}
```

到这里为止，clients的处理和自身的一些数据处理都已经结束了，那么后续就要做一些Redis自身架构上的事情了，那就是针对AOF和RDB的处理了，不过这里就先不展开相关的内容，具体的做法到专门分析AOF和RDB的时候再来补充这块内容，这里就先跳过一大段的AOF和RDB相关的内容。

最后还会处理一些clients和集群等相关的内容，就放到一起快速过一遍。

```c
/* Close clients that need to be closed asynchronous */
freeClientsInAsyncFreeQueue();

/* Clear the paused clients flag if needed. */
clientsArePaused(); /* Don't check return value, just use the side effect.*/

/* Replication cron function -- used to reconnect to master,
     * detect transfer failures, start background RDB transfers and so forth. */
run_with_period(1000) replicationCron();

/* Run the Redis Cluster cron. */
run_with_period(100) {
  if (server.cluster_enabled) clusterCron();
}

/* Run the Sentinel timer if we are in sentinel mode. */
if (server.sentinel_mode) sentinelTimer();

/* Cleanup expired MIGRATE cached sockets. */
run_with_period(1000) {
  migrateCloseTimedoutSockets();
}
```

这几个方法看名称都很好理解，具体的原理由于不是Redis的重点内容，将来有需要了再来细细研究，目前只要知道大概干了些什么事情即可。