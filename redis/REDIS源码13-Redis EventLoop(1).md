#REDIS源码13-Redis EventLoop(1)

分析完了Redis的基本数据结构，接下去的重点是分析Redis是如何接受并处理客户端的网络请求的，它的IO模型是什么样子的，同时根据Redis的处理进一步分析Redis是如何把过期的数据清理掉的，同时又是如何触发备份（AOF和RDB的）。同时如果可以的话还可以分析一下Redis的集群模式和Sentinel模式又是如何工作的。

## 事件循环

Redis的事件循环和Netty的时间循环有一些类似，但是没有Netty这么复杂。

Redis的时间循环就是有一个EventLoop，然后EventLoop上有多个数组或链表，然后在主循环上不停的去处理被注册上来的这些事件。

> 相对的Netty要做的事情就更多一些，又因为是多线程的模型，不像Redis是单线程的模型。

在C语言的main方法中，

```c
int main(int argc, char **argv) {
  // 省略其他
	aeMain(server.el);
  // 省略其他
}
```

这个aeMain方法就是正式使用eventLoop来接受网络请求。

```c
void aeMain(aeEventLoop *eventLoop) {
  eventLoop->stop = 0;
  while (!eventLoop->stop) {
    if (eventLoop->beforesleep != NULL)
      eventLoop->beforesleep(eventLoop);
    aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
  }
}
```

可以看到在整个循环的过程中，eventLoop在一个死循环中不停的调用**beforeSleep**和**aeProcessEvents**这两个方法。

先看**beforesleep**到底做了些什么事情

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
  // 这个方法就是更新Redis集群的状态，比如个集群内的其他节点发pong的心跳等
  if (server.cluster_enabled) clusterBeforeSleep();

  // FAST类型的expire cycle，这个cycle里面会过期一些老的数据，腾出一些内存空间
  // 具体如何expire的，后面再单独分析，现在只要知道是在一个EventLoop里面做的这件事情
  // active_expire_enabled默认是1，也就是打开。除非是自己测试debug，一般不要关闭
  if (server.active_expire_enabled && server.masterhost == NULL)
    activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

	// 给slave机器发送ACK
  if (server.get_ack_from_slaves) {
    robj *argv[3];

    argv[0] = createStringObject("REPLCONF",8);
    argv[1] = createStringObject("GETACK",6);
    argv[2] = createStringObject("*",1); /* Not used argument. */
    replicationFeedSlaves(server.slaves, server.slaveseldb, argv, 3);
    decrRefCount(argv[0]);
    decrRefCount(argv[1]);
    decrRefCount(argv[2]);
    server.get_ack_from_slaves = 0;
  }

	// 解锁所有等在wait命令上的client，可以先忽略
  if (listLength(server.clients_waiting_acks))
    processClientsWaitingReplicas();

  // Redis module，直接忽略
  moduleHandleBlockedClients();

  // 处理之前因为Block指令被阻塞，现在需要unblock的那些clients
  if (listLength(server.unblocked_clients))
    processUnblockedClients();

  // 写AOF日志
  flushAppendOnlyFile(0);

  // 没写完的接着写？
  handleClientsWithPendingWrites();

  // Redis module，可以直接忽略
  if (moduleCount()) moduleReleaseGIL();
}

int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
  int processed = 0, numevents;

  // 有fd，或者flag指明了要处理哪种类型的时间
  // TIME_EVENT就是一些定时任务，注册到一个一个链表上，然后每次在调用这个方法的时候看那个EventLoop上是不是有需要执行的任务。
  if (eventLoop->maxfd != -1 ||
      ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
    int j;
    aeTimeEvent *shortest = NULL;
    struct timeval tv, *tvp;

    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
      // 这个shortest就是看注册的TIME_EVENT
      shortest = aeSearchNearestTimer(eventLoop);
    // 这个地方目的就是算出来我阻塞的获取IO时间，这个超时时间是多少。
    // 因为如果有一些定时任务，我如果block的时间很长，这个定时任务执行的时候肯定已经滞后了。
    // 所以这里其实就是为了算一下这个时间。
    if (shortest) {
      long now_sec, now_ms;

      aeGetTime(&now_sec, &now_ms);
      tvp = &tv;

      /* How many milliseconds we need to wait for the next
             * time event to fire? */
      long long ms =
        (shortest->when_sec - now_sec)*1000 +
        shortest->when_ms - now_ms;

      if (ms > 0) {
        tvp->tv_sec = ms/1000;
        tvp->tv_usec = (ms % 1000)*1000;
      } else {
        tvp->tv_sec = 0;
        tvp->tv_usec = 0;
      }
    } else {
      /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
      if (flags & AE_DONT_WAIT) {
        tv.tv_sec = tv.tv_usec = 0;
        tvp = &tv;
      } else {
        /* Otherwise we can block */
        tvp = NULL; /* wait forever */
      }
    }

    // 上面算出来tvp，就是等待的时间有多少。
    // 这个地方就是把IO事件读取出来，然后放到eventLoop的fired里面。
    // fired就是一个时间循环等待处理的事件。
    // 这个方法也是一个不同的平台和操作系统有不同的实现，比如linux下可以用select或者epoll来实现，iOS下可以用kqueue来实现等。
    numevents = aeApiPoll(eventLoop, tvp);

    // 这个aftersleep就是上面已经获取了待处理的时间，不过具体的实现里面这个aftersleep没有干什么事情，可以忽略
    if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
      eventLoop->aftersleep(eventLoop);

    for (j = 0; j < numevents; j++) {
      // 这个fired就是前面获取到的IO事件
      aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
      int mask = eventLoop->fired[j].mask;
      int fd = eventLoop->fired[j].fd;
      int fired = 0; /* Number of events fired for current fd. */

      // 这个invert的意思是，通常Redis都是先处理读，再处理写。
      // 但是万一就是要反过来，先处理写再处理读，那就用这个invert来表示就是要反过来。
      int invert = fe->mask & AE_BARRIER;

      if (!invert && fe->mask & mask & AE_READABLE) {
        // 注册的读事件处理方法来处理
        // rfileProc是一个函数指针
        fe->rfileProc(eventLoop,fd,fe->clientData,mask);
        fired++;
      }

      /* Fire the writable event. */
      if (fe->mask & mask & AE_WRITABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
          // 注册的写事件处理方法来处理
          // wfileProc也是一个函数指针
          fe->wfileProc(eventLoop,fd,fe->clientData,mask);
          fired++;
        }
      }

      // 这个是为了invert的时候才执行的，通常情况下应该是忽略就可以了
      if (invert && fe->mask & mask & AE_READABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
          // 同上
          fe->rfileProc(eventLoop,fd,fe->clientData,mask);
          fired++;
        }
      }

      processed++;
    }
  }
  // 这里就是尝试去处理定时任务
  if (flags & AE_TIME_EVENTS)
    processed += processTimeEvents(eventLoop);

  return processed; /* return the number of processed file/time events */
}
```

其实**beforesleep**就是在每次EventLoop的过程中，处理集群相关的内容、处理客户端的block和block请求有结果需要return时的unblock处理、以及写AOF日志等相关事情。

而**aeProcessEvents**则是真正的通过操作系统的IO方法读取具体的IO事件，并且对应的处理这些IO事件。

那么真正处理事件的两个函数指针，一个读**rfileProc**以及一个写**wfileProc**，下面重点分析一下，这两个函数指针到底是指向哪个函数的，那个函数就是具体处理IO事件的。

##读事件处理

Redis中读事件的类型有很多，比如处理集群逻辑的**clusterReadHandler**，还有比如试图去同步Cluster Master的**readSyncBulkPayload**(这个地方就是先用write从master那里读取文件写下来，然后再用read去读取写下来的master记录来同步master的数据)，比如处理新的客户端链接的**acceptTcpHandler**，以及处理客户端发送的Redis操作的**readQueryFromClient**。

所以我们目前现在主要是分析Redis是如何处理命令的，所以我们重点分析**readQueryFromClient**这个方法。

```c
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
  client *c = (client*) privdata;
  int nread, readlen;
  size_t qblen;
  UNUSED(el);
  UNUSED(mask);

  readlen = PROTO_IOBUF_LEN;
  if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
      && c->bulklen >= PROTO_MBULK_BIG_ARG)
  {
    ssize_t remaining = (size_t)(c->bulklen+2)-sdslen(c->querybuf);
    if (remaining > 0 && remaining < readlen) readlen = remaining;
  }

  qblen = sdslen(c->querybuf);
  if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
  c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
  // 一直到这里就是把数据读取到querybuf中
  nread = read(fd, c->querybuf+qblen, readlen);
  if (nread == -1) {
    if (errno == EAGAIN) {
      return;
    } else {
      serverLog(LL_VERBOSE, "Reading from client: %s",strerror(errno));
      freeClient(c);
      return;
    }
  } else if (nread == 0) {
    // 这个就是关闭一个client
    serverLog(LL_VERBOSE, "Client closed connection");
    freeClient(c);
    return;
  } else if (c->flags & CLIENT_MASTER) {
    /* Append the query buffer to the pending (not applied) buffer
         * of the master. We'll use this buffer later in order to have a
         * copy of the string applied by the last command executed. */
    c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                    c->querybuf+qblen,nread);
  }

  sdsIncrLen(c->querybuf,nread);
  c->lastinteraction = server.unixtime;
  if (c->flags & CLIENT_MASTER) c->read_reploff += nread;
  server.stat_net_input_bytes += nread;
  // 如果待处理的数据大于client_max_querybuf_len = 1024 * 1024 * 1024 = 1G
  if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
    sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();

    bytes = sdscatrepr(bytes,c->querybuf,64);
    serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
    sdsfree(ci);
    sdsfree(bytes);
    freeClient(c);
    return;
  }

	// 真正的处理数据
  processInputBufferAndReplicate(c);
}

void processInputBufferAndReplicate(client *c) {
  // 如果不是master
  if (!(c->flags & CLIENT_MASTER)) {
    // 处理数据
    processInputBuffer(c);
  } else {
    // 之前的偏移
    size_t prev_offset = c->reploff;
    // 处理命令
    processInputBuffer(c);
    // 看看偏移是不是变化
    size_t applied = c->reploff - prev_offset;
    // 如果有变化，就去同步master的数据
    if (applied) {
      replicationFeedSlavesFromMasterStream(server.slaves,
                                            c->pending_querybuf, applied);
      sdsrange(c->pending_querybuf,applied,-1);
    }
  }
}
```

下面是**processInputBuffer**方法，去处理client发送来的命令。

```c
void processInputBuffer(client *c) {
  server.current_client = c;

  // 只要querybuf中还有数据可以读取，就不停的处理
  while(c->qb_pos < sdslen(c->querybuf)) {
    // 一些特殊情况处理
    if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;
    if (c->flags & CLIENT_BLOCKED) break;
    if (server.lua_timedout && c->flags & CLIENT_MASTER) break;
    if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

    // 看命令是一行还是多行
    if (!c->reqtype) {
      if (c->querybuf[c->qb_pos] == '*') {
        c->reqtype = PROTO_REQ_MULTIBULK;
      } else {
        c->reqtype = PROTO_REQ_INLINE;
      }
    }

    // 这里就是去解析client里面的querybuf
    // 然后把querybuf里面的数据写到client->argv里面
    if (c->reqtype == PROTO_REQ_INLINE) {
      if (processInlineBuffer(c) != C_OK) break;
    } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
      if (processMultibulkBuffer(c) != C_OK) break;
    } else {
      serverPanic("Unknown request type");
    }

    // 根据看客户端是不是传了参数
    if (c->argc == 0) {
      resetClient(c);
    } else {
      // processCommand就是处理client的命令
      if (processCommand(c) == C_OK) {
        if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
          /* Update the applied replication offset of our master. */
          c->reploff = c->read_reploff - sdslen(c->querybuf) + c->qb_pos;
        }

        if (!(c->flags & CLIENT_BLOCKED) || c->btype != BLOCKED_MODULE)
          resetClient(c);
      }
      if (server.current_client == NULL) break;
    }
  }

  /* Trim to pos */
  if (server.current_client != NULL && c->qb_pos) {
    sdsrange(c->querybuf,c->qb_pos,-1);
    c->qb_pos = 0;
  }

  server.current_client = NULL;
}
```

处理Redis指令**processCommand**，这个方法接近两百行，所以就抽取一些重要的内容分析

```c
int processCommand(client *c) {

  // 如果是quit命令，就直接退出

  // 从命令table中找到命令
  c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
  // 如果找不到命令就直接退出或参数的个数不正确，就直接退出
  // 检查client是不是已经auth授权通过

  // 集群模式处理省略

  // 省略各种校验检查

  /* Exec the command */
  if (c->flags & CLIENT_MULTI &&
      c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
      c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
  {
    // 处理多个命令
    queueMultiCommand(c);
    addReply(c,shared.queued);
  } else {
    // 处理单个命令
    call(c,CMD_CALL_FULL);
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
      handleClientsBlockedOnKeys();
  }
  return C_OK;
}
```

下面看call命令

```c
void call(client *c, int flags) {
  long long dirty;
  ustime_t start, duration;
  int client_old_flags = c->flags;
  struct redisCommand *real_cmd = c->cmd;

  server.fixed_time_expire++;

  // monitor，就是连上来专门看redis都干了些什么，算是一个调试的功能，这里可以不用管
  if (listLength(server.monitors) &&
      !server.loading &&
      !(c->cmd->flags & (CMD_SKIP_MONITOR|CMD_ADMIN)))
  {
    replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
  }

  /* Initialization: clear the flags that must be set by the command on
     * demand, and initialize the array for additional commands propagation. */
  c->flags &= ~(CLIENT_FORCE_AOF|CLIENT_FORCE_REPL|CLIENT_PREVENT_PROP);
  redisOpArray prev_also_propagate = server.also_propagate;
  redisOpArrayInit(&server.also_propagate);

  // 这个就是看有多少数据被修改了
  dirty = server.dirty;
  updateCachedTime(0);
  start = server.ustime;
  // 执行redis命令
  c->cmd->proc(c);
  // 耗时
  duration = ustime()-start;
  // 修改的数据条数
  dirty = server.dirty-dirty;
  if (dirty < 0) dirty = 0;

  /* When EVAL is called loading the AOF we don't want commands called
     * from Lua to go into the slowlog or to populate statistics. */
  if (server.loading && c->flags & CLIENT_LUA)
    flags &= ~(CMD_CALL_SLOWLOG | CMD_CALL_STATS);

  // lua脚本处理
  if (c->flags & CLIENT_LUA && server.lua_caller) {
    if (c->flags & CLIENT_FORCE_REPL)
      server.lua_caller->flags |= CLIENT_FORCE_REPL;
    if (c->flags & CLIENT_FORCE_AOF)
      server.lua_caller->flags |= CLIENT_FORCE_AOF;
  }

  // 记录日志和统计信息
  // slowlog就是慢操作，大于10000微妙 = 10毫秒，超过10毫秒的都是慢操作
  if (flags & CMD_CALL_SLOWLOG && c->cmd->proc != execCommand) {
    char *latency_event = (c->cmd->flags & CMD_FAST) ?
      "fast-command" : "command";
    latencyAddSampleIfNeeded(latency_event,duration/1000);
    slowlogPushEntryIfNeeded(c,c->argv,c->argc,duration);
  }
  if (flags & CMD_CALL_STATS) {
    /* use the real command that was executed (cmd and lastamc) may be
         * different, in case of MULTI-EXEC or re-written commands such as
         * EXPIRE, GEOADD, etc. */
    real_cmd->microseconds += duration;
    real_cmd->calls++;
  }

  // 这段代码就是看是否记录AOF日志或给所有的备份机器同步数据
  if (flags & CMD_CALL_PROPAGATE &&
      (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP)
  {
    int propagate_flags = PROPAGATE_NONE;

    /* Check if the command operated changes in the data set. If so
         * set for replication / AOF propagation. */
    if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);

    /* If the client forced AOF / replication of the command, set
         * the flags regardless of the command effects on the data set. */
    if (c->flags & CLIENT_FORCE_REPL) propagate_flags |= PROPAGATE_REPL;
    if (c->flags & CLIENT_FORCE_AOF) propagate_flags |= PROPAGATE_AOF;

    /* However prevent AOF / replication propagation if the command
         * implementations called preventCommandPropagation() or similar,
         * or if we don't have the call() flags to do so. */
    if (c->flags & CLIENT_PREVENT_REPL_PROP ||
        !(flags & CMD_CALL_PROPAGATE_REPL))
      propagate_flags &= ~PROPAGATE_REPL;
    if (c->flags & CLIENT_PREVENT_AOF_PROP ||
        !(flags & CMD_CALL_PROPAGATE_AOF))
      propagate_flags &= ~PROPAGATE_AOF;

    /* Call propagate() only if at least one of AOF / replication
         * propagation is needed. Note that modules commands handle replication
         * in an explicit way, so we never replicate them automatically. */
    if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
      propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
  }

  /* Restore the old replication flags, since call() can be executed
     * recursively. */
  c->flags &= ~(CLIENT_FORCE_AOF|CLIENT_FORCE_REPL|CLIENT_PREVENT_PROP);
  c->flags |= client_old_flags &
    (CLIENT_FORCE_AOF|CLIENT_FORCE_REPL|CLIENT_PREVENT_PROP);

  // 这个就是看是不是要求数据至少同步多个节点
  if (server.also_propagate.numops) {
    int j;
    redisOp *rop;

    if (flags & CMD_CALL_PROPAGATE) {
      for (j = 0; j < server.also_propagate.numops; j++) {
        rop = &server.also_propagate.ops[j];
        int target = rop->target;
        /* Whatever the command wish is, we honor the call() flags. */
        if (!(flags&CMD_CALL_PROPAGATE_AOF)) target &= ~PROPAGATE_AOF;
        if (!(flags&CMD_CALL_PROPAGATE_REPL)) target &= ~PROPAGATE_REPL;
        if (target)
          propagate(rop->cmd,rop->dbid,rop->argv,rop->argc,target);
      }
    }
    redisOpArrayFree(&server.also_propagate);
  }
  server.also_propagate = prev_also_propagate;
  server.fixed_time_expire--;
  server.stat_numcommands++;
}
```

call命令大体分为几个部分：

- 执行命令
- 更新统计信息
- 记录日志或同步数据