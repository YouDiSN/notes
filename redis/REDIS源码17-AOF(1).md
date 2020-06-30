#REDIS源码17-AOF(1)

Redis的备份有两种方式，分别是快照和AOF，快照就是一次全量复制，AOF则是一种增量备份。

快照就是RDB，是一种二进制的文件形式，而AOF则是一个记录了修改指令的文本。

本节分析AOF，RDB之后再详细分析。

##AOF原理

AOF其实就是一个指令的序列，可以根据这个指令序列进行指令的重放，从而达到恢复数据的目的。

不过需要注意的是，和一般的数据库不同的是往往先记录日志才做处理，但是Redis是先处理再记录日志的。

> **为什么Redis要先执行命令再写日志。**
>
> 一般而言有两种说法，一个是为了防止AOF文件过大，还有一个是Redis本身不支持回滚。
>
> - 防止AOF文件过大。首先如果先记录日志的话，那么Redis只能针对一些语法进行检查，但是这个命令执行完是不是有效果其实并不知道，比如一个key不存在，我现在要删除这个不存在的key，如果是先记录日志，肯定会把这个操作记录进去，但是如果是执行完之后操作的话就知道这个命令其实没有做任何修改，也就是AOF日志中就可以不用记录了。
> - Redis不支持回滚。Mysql等通过binlog和redolog这样的日志来确保Crash Safe。但是Redis本身也没有这样的保证，所以即便Crash之后又数据丢失也是提前预知的行为。
>
> 所以既然没有确保数据安全的必要，又可以降低AOF文件的大小，所以就没有先写日志再操作数据了。
>
> 此外，作者在描述AOF的时候有提到：AOF是一个记录Redis State的东西。
>
> 那么既然是记录State，肯定是只有操作完了再记录，而不是再操作前记录的，所以应该是这三点的一个综合。
>
> 1. 防止AOF文件过大
> 2. 不支持回滚，所以没有必要
> 3. 设计上就是希望用来记录Redis当前数据状态的，那肯定是State（可以理解成数据）被修改了才有记录的必要

## 写AOF

在之前分析的EventLoop中有一个call方法是用来执行具体的Redis指令的，在整个指令执行完之后，就会尝试记录AOF日志，源码如下：

```c
void call(client *c, int flags) {
  // 省略

  /* Call the command. */
  dirty = server.dirty;
  updateCachedTime(0);
  start = server.ustime;
  // 执行命令
  c->cmd->proc(c);
  duration = ustime()-start;
  dirty = server.dirty-dirty;
  if (dirty < 0) dirty = 0;

	// 省略

  // 判断是不是需要AOF或者给slave节点发通知
  if (flags & CMD_CALL_PROPAGATE &&
      (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP)
  {
    int propagate_flags = PROPAGATE_NONE;

    // dirty就是看是不是有数据被修改了，如果有数据被修改就要AOF，言外之意没有数据修改就不管
    if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);
    // 看看是不是配置了强制Replication同步或强制AOF
    if (c->flags & CLIENT_FORCE_REPL) propagate_flags |= PROPAGATE_REPL;
    if (c->flags & CLIENT_FORCE_AOF) propagate_flags |= PROPAGATE_AOF;

    //看看是不是有避免做Replication同步或避免AOF
    if (c->flags & CLIENT_PREVENT_REPL_PROP ||
        !(flags & CMD_CALL_PROPAGATE_REPL))
      propagate_flags &= ~PROPAGATE_REPL;
    if (c->flags & CLIENT_PREVENT_AOF_PROP ||
        !(flags & CMD_CALL_PROPAGATE_AOF))
      propagate_flags &= ~PROPAGATE_AOF;

    if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
      // 这个地方就会尝试去做AOF
      propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
  }
  
	// 省略
}
```

所以可以看到，只要有数据修改或着服务器配置了强制AOF，那么Redis就会调用propagate方法来做处理。

所以需要重点分析这个方法。

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
               int flags)
{
  // AOF
  if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
    feedAppendOnlyFile(cmd,dbid,argv,argc);
  // Replication
  if (flags & PROPAGATE_REPL)
    replicationFeedSlaves(server.slaves,dbid,argv,argc);
}
```

Replication之后如果有时间分析Redis Cluster的话就分析一下，本次就分析AOF就好了。

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
  // 创建一个空的sds
  sds buf = sdsempty();
  robj *tmpargv[3];

  /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
  if (dictid != server.aof_selected_db) {
    char seldb[64];

    snprintf(seldb,sizeof(seldb),"%d",dictid);
    buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
                       (unsigned long)strlen(seldb),seldb);
    server.aof_selected_db = dictid;
  }

  if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
      cmd->proc == expireatCommand) {
		// 如果是expire、pexpire和expireat这三个命令，就用专门的方法来处理AOF
    // 这三个都是过期，只不过expire是过期接受秒为参数，pexpire接受毫秒，expireat接受时间戳
    buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
  } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
    // 处理setex和psetex两个命令
    // 带过期时间的set，分别是秒和毫秒
    tmpargv[0] = createStringObject("SET",3);
    tmpargv[1] = argv[1];
    tmpargv[2] = argv[3];
    // 这是针对set命令的，和下面一样
    buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
    decrRefCount(tmpargv[0]);
    // 这个是针对expire的，这个和上面的那个一样
    buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
  } else if (cmd->proc == setCommand && argc > 3) {
    // 处理set命令，并且参数数量大于3的情况，其实和上面的setex和psetex是一样的，只不过用了另一种传指令的方式
    int i;
    robj *exarg = NULL, *pxarg = NULL;
    /* Translate SET [EX seconds][PX milliseconds] to SET and PEXPIREAT */
    buf = catAppendOnlyGenericCommand(buf,3,argv);
    for (i = 3; i < argc; i ++) {
      if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i+1];
      if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i+1];
    }
    serverAssert(!(exarg && pxarg));
    if (exarg)
      buf = catAppendOnlyExpireAtCommand(buf,server.expireCommand,argv[1],
                                         exarg);
    if (pxarg)
      buf = catAppendOnlyExpireAtCommand(buf,server.pexpireCommand,argv[1],
                                         pxarg);
  } else {
    // 除了上面这些特殊情况下的所有其他指令都直接用这个方法就可以了
    buf = catAppendOnlyGenericCommand(buf,argc,argv);
  }

  // 把生成的buf追加到aof_buf中，aof_buf也是一个sds
  if (server.aof_state == AOF_ON)
    server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

  // 这个不为-1就代表现在有一个子进程正在rewrite AOF文件，所以我们这里的AOF就不用写在老文件了，直接放到新的AOF文件就可以了
  if (server.aof_child_pid != -1)
    aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

  sdsfree(buf);
}
```

从上面的这段代码可以看出来，其实洋洋洒洒这么多代码，真正的逻辑很简单，就是根据不同的命令生成对应要写到日志里的buf，然后把这个buf追加到全局**aof_buf**中，然后根据是不是有子进程在Rewrite AOF文件。

所谓不同的命令的处理，其实也就只有两个方法，一个是**catAppendOnlyExpireAtCommand**还有一个是**catAppendOnlyGenericCommand**。所以重点分析一下这两个方法就知道AOF的时候都在干什么。

###catAppendOnlyExpireAtCommand

这个方法其实是**catAppendOnlyGenericCommand**的一个封装，只不过因为expire有不同的时间长度等，所以在同一调用**catAppendOnlyGenericCommand**之前先做一些统一的处理，把各种expire都翻译成同一个。

```c
sds catAppendOnlyExpireAtCommand(sds buf, struct redisCommand *cmd, robj *key, robj *seconds) {
  long long when;
  robj *argv[3];

  // 这两个方法就是把字符串变成int
  seconds = getDecodedObject(seconds);
  when = strtoll(seconds->ptr,NULL,10);
  // 根据毫秒等变成具体的值
  if (cmd->proc == expireCommand || cmd->proc == setexCommand ||
      cmd->proc == expireatCommand)
  {
    when *= 1000;
  }
  /* Convert into absolute time for EXPIRE, PEXPIRE, SETEX, PSETEX */
  if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
      cmd->proc == setexCommand || cmd->proc == psetexCommand)
  {
    when += mstime();
  }
  decrRefCount(seconds);

  argv[0] = createStringObject("PEXPIREAT",9);
  argv[1] = key;
  argv[2] = createStringObjectFromLongLong(when);
  // 内部调用catAppendOnlyGenericCommand
  buf = catAppendOnlyGenericCommand(buf, 3, argv);
  decrRefCount(argv[0]);
  decrRefCount(argv[2]);
  return buf;
}
```

###catAppendOnlyGenericCommand

```c
sds catAppendOnlyGenericCommand(sds dst, int argc, robj **argv) {
  char buf[32];
  int len, j;
  robj *o;

  // *开头
  buf[0] = '*';
  // 把argc的长度写到buf中
  len = 1+ll2string(buf+1,sizeof(buf)-1,argc);
  // 追加
  buf[len++] = '\r';
  buf[len++] = '\n';
  dst = sdscatlen(dst,buf,len);

  // 把输入的参数都一个个写到buf中
  for (j = 0; j < argc; j++) {
    o = getDecodedObject(argv[j]);
    buf[0] = '$';
    len = 1+ll2string(buf+1,sizeof(buf)-1,sdslen(o->ptr));
    buf[len++] = '\r';
    buf[len++] = '\n';
    dst = sdscatlen(dst,buf,len);
    dst = sdscatlen(dst,o->ptr,sdslen(o->ptr));
    dst = sdscatlen(dst,"\r\n",2);
    decrRefCount(o);
  }
  
  // *3 	这个就代表这个命令有3个参数
	// $3 	这个就代表第一个参数长度为3
	// set  这个就是长度为3的第一个参数值
	// $3 	这个代表第二个参数长度也是3
	// we3 	这个就是第二个长度为3的参数值
	// $4 	这个就是第三个参数长度为4
	// 1234 这个就是第三个参数的值
  return dst;
}
```

上面的分析可以看到已经生成了具体要写到日志里的sds，同时也已经放到全局**aof_buf**中，那么接下去要分析的问题还有两个，一个是什么时候把**aof_buf**的日志真正写到文件里，还有一个就是AOF子进程是如何开启的，同时开启之后又做了一些什么事情。

## 写文件

Redis中要把**aof_buf**写到磁盘的方法是**flushAppendOnlyFile**，这个方法有几个典型的地方在调用，首先是EventLoop中的**beforeSleep**；**prepareForShutdown**就是Redis关闭前（这个不只是关闭，比如restart指令也是先关闭在开启，反正就是关闭之前也会flush一下）；还有一个就是**serverCron**方法里面。

> 重新复习一下，beforeSleep是aeMain函数中的，每次时间循环都会调用一次，shutdown则是需要关闭的地方会涉及到，serverCron是通过aeTimedEvent触发，但是是根据server.hz来做判断是不是调用的太频繁等，所以虽然每次时间循环都去看是不是有TimedEvent需要执行，但是不是每次都会执行到**serverCron**方法的。
>
> 关于**serverCron**需要补充一点的就是，在这个方法里面的flush一秒内只会执行一次，而且必须上一次flush失败，才会执行。

所以除了关闭之前的操作，其实就是每次执行下一个命令前都会尝试去flush一下。

那么重点就是**flushAppendOnlyFile**是如何实现的，不过鉴于这个方法实在太长了（接近200行），所以就挑选重点分析。

###flush的执行时机

```c
#define AOF_FSYNC_NO 0
#define AOF_FSYNC_ALWAYS 1
#define AOF_FSYNC_EVERYSEC 2
```

首先这个方法虽然会被频繁调用，但是不代表每一次调用都会flush AOF文件，因为这里面会进行一个判断，也就是网上经常能看到的AOF的三种模式，分别是不Flush、每一个命令都Flush以及每一秒Flush。

```c
void flushAppendOnlyFile(int force) {
  // 省略无用代码
  
  // 一上来先判断一下aof_buf里面是不是有东西，
  if (sdslen(server.aof_buf) == 0) {
    // 进入到里面的判断就是aof_buf没有东西，
    // 根据Fsync的模式是不是AOF_FSYNC_EVERYSEC，是不是本次执行方法即便aof_buf为空也要执行一下Fsync，比如上一次没有Fsync没有完全把buf内容都写到文件里
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
        server.aof_fsync_offset != server.aof_current_size &&
        server.unixtime > server.aof_last_fsync &&
        !(sync_in_progress = aofFsyncInProgress())) {
      // 这个就是一个单独的代码块，去写文件的，后面分析
      goto try_fsync;
    } else {
      return;
    }
  }
}
```

上面的逻辑是如果没有东西需要AOF，那么做一些事情。后面的所有代码都是buf中有内容。

```c
if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
  // 看Fsync是不是正在执行中
  sync_in_progress = aofFsyncInProgress();
// 这个就是如果是每秒Fsync同时本次不是强制flush，出现强制flush是因为比如server关闭之前就没有办法等了，需要立即执行，这种就需要force，常规情况都不是force
if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
  // 如果fsync以及在执行了
  if (sync_in_progress) {
    // 这里就是如果Fsync已经在跑了，那么我就把本次fsync放到延迟fsync中，然后标记一下时间戳
    if (server.aof_flush_postponed_start == 0) {
			// 这个就是之前没有需要延时执行的fsync，把自己更新
      server.aof_flush_postponed_start = server.unixtime;
      return;
    } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
      // 差距2s内，不管
      return;
    }
    // 如果之前的Fsync 超过2s还没有被执行，就记录一些日志
    // 其实和上面2s内的逻辑上没有区别，就是多打一些日志告诉你有这个情况
    server.aof_delayed_fsync++;
    serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
  }
}
```

上面这段代码展示的是每秒刷磁盘的情况下需要做的判断，继续往下。

下面这段代码就是调用操作系统的write函数来写文件。

```c
// 这是write函数执行前的时间戳
latencyStartMonitor(latency);
nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
// 这是计算本次调用操作系统write方法到底花了多少时间
latencyEndMonitor(latency);
// 根据不同情况来记录统计信息
if (sync_in_progress) {
  latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
} else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
  latencyAddSampleIfNeeded("aof-write-active-child",latency);
} else {
  latencyAddSampleIfNeeded("aof-write-alone",latency);
}
latencyAddSampleIfNeeded("aof-write",latency);
```

> 这里要说明一点的是，这里只是调用write，并没有真正刷到磁盘上。
>
> 因为对于linux而言，write只是写到高速缓冲区内，并不是真正落到磁盘上，调用完write直接拔电源是有可能造成文件丢失的，所以真正的落盘必须是调用fsync之后。
>
> 不过这里既然分析到了再补充一点，既然write只是写到高速缓冲区内，没有写到磁盘上，调用完write之后马上read是不是能读取到刚才write的内容呢？会不会因为文件没有写到磁盘就读取不到呢？
>
> 答案不会，马上read也可以读取到刚才write的内容，因此操作系统针对高速缓冲区做了优化，如果操作系统判断这块缓冲没有更新，会直接从缓冲区返回对应的数据而不是从磁盘上刷。
>
> 所以这里的write只是写到高速缓冲区内，并不是真正的持久化。

再往后有很长的一段如果write失败的处理，我们假装write成功，跳过这段各种if判断的和输出日志的代码。

后面就是write文件都已经write完了，**aof_buf**已经没用了，针对这个进行一些处理

```c
// 如果aof_buf大小4000之内，就重用这块内存
// 否则就清空，重新生成一块新的
// 这个主要是保证不会太浪费
if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
  sdsclear(server.aof_buf);
} else {
  sdsfree(server.aof_buf);
  server.aof_buf = sdsempty();
}
```

最后尝试fsync了。

```c
try_fsync:
    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
		// 如果已经rewrite或执行rdb，就不尝试fsync了，因为已经有子线程在做了
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    // 如果是always就马上执行fsync
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
				// 这个很好理解就是调用fsync
        latencyStartMonitor(latency);
        redis_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_fsync_offset = server.aof_current_size;
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
	      // 这里是如果每秒执行一次fsync
	      // server.unixtime就是秒级时间戳，比上一次的大就说明超过1s了
	      // 并且当前没有fsync在执行
        if (!sync_in_progress) {
	          // 就生成一个background job
            aof_background_fsync(server.aof_fd);
            server.aof_fsync_offset = server.aof_current_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
}
```

再看看这个background job的创建：

```c
void aof_background_fsync(int fd) {
  bioCreateBackgroundJob(BIO_AOF_FSYNC,(void*)(long)fd,NULL,NULL);
}

void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
  struct bio_job *job = zmalloc(sizeof(*job));

  job->time = time(NULL);
  job->arg1 = arg1;
  job->arg2 = arg2;
  job->arg3 = arg3;
  // 把这个job放到bio_jobs的队列中
  pthread_mutex_lock(&bio_mutex[type]);
  listAddNodeTail(bio_jobs[type],job);
  bio_pending[type]++;
  pthread_cond_signal(&bio_newjob_cond[type]);
  pthread_mutex_unlock(&bio_mutex[type]);
}
```

这个就是之前分析多线程的时候有说过Redis创建了3个线程，其中有一个就是**BIO_AOF_FSYNC**，这个线程就是专门用来处理每秒flush一次的。

这个线程就会调用**redis_fsync**这个方法就是fsync方法。

到了这里写AOF的原理都分析清楚了，后面还有如何根据AOF来恢复数据以及rewrite的相关逻辑。