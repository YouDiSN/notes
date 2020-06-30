#REDIS源码18-AOF(2)

上一个章节分析了AOF的写和fsync的基本原理，那么这章节要分析的是AOF Rewrite。

因为AOF文件不断的添加，会导致文件非常非常大，aof文件非常大的坏处比较典型的就是占用磁盘空间，其次就是根据aof文件来恢复数据的时候，会非常冗长。

因此针对aof文件，Redis有自己的瘦身逻辑，就是AOF Rewrite。

那么这个瘦身的方式其实很简单，就是做指令的合并，比如针对一个key有100个操作，那么rewrite之后其实只要一个操作就可以了，就直接把最后的数据状态插入就可以了，并不需要知道之前的100个操作到底是什么。这就是为什么AOF在文件操作上的基本原理。

另外插一句就是，AOF不需要读取之前的aof文件，因为只要最终状态的话当前内存里面的数据就是最好的材料，没有必要再去读文件来处理什么。

##什么时候会Rewrite

首先判断是否需要Rewrite的地方出发是在EventLoop的serverCron方法中，这个方法中有两个地方判断是否需要执行rewrite，分别如下：

```c
if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 && server.aof_rewrite_scheduled) {
  rewriteAppendOnlyFileBackground();
}
```

这个就是如果没有rdb子进程和aof子进程，以及配置了**aof_rewrite_scheduled**，就执行对应的方法。这个地方默认是0，也就是不会进来。但是**rewriteAppendOnlyFileBackground**里面是有逻辑会把这个配置改成1的，这个具体分析到这个方法内部的时候再深入分析。

第二个地方执行rewrite还是在serverCron中，就接在上面一种情况的下面

```c
if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
    ldbPendingChildren())
{
  int statloc;
  pid_t pid;

  if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
    // 省略其他各种情况
		if (pid == server.aof_child_pid) {
      // 这个就是rewrite结束之后要做的事情
      backgroundRewriteDoneHandler(exitcode,bysignal);
      if (!bysignal && exitcode == 0) receiveChildInfo();
    }
    // 省略其他各种情况
  }
} else {
  //省略rdb情况

  // 如果可以aof，同时没有rdb子进程、没有aof子进程，同时当前的aof文件大小 > 64*1024*1024 = 64mb
  if (server.aof_state == AOF_ON &&
      server.rdb_child_pid == -1 &&
      server.aof_child_pid == -1 &&
      server.aof_rewrite_perc &&
      server.aof_current_size > server.aof_rewrite_min_size)
  {
    // aof_rewrite_base_size默认是0，但是每次rewrite之后会不断的更新，越来越大
    long long base = server.aof_rewrite_base_size ?
      server.aof_rewrite_base_size : 1;
    long long growth = (server.aof_current_size*100/base) - 100;
    // aof_rewrite_perc = 100
    // 这个的意思就是和上一次rewrite比，growth了多少，是不是超过aof_rewrite_perc，否则就不rewrite
    if (growth >= server.aof_rewrite_perc) {
      rewriteAppendOnlyFileBackground();
    }
  }
}
```

通常情况下，出发rewrite的是这段代码中触发的。

##rewriteAppendOnlyFileBackground

下面就具体分析一下这个方法都干了些什么事情。

```c
int rewriteAppendOnlyFileBackground(void) {
  pid_t childpid;
  long long start;

  // 如果已经rdb子线程和AOF子线程，就直接返回
  if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;
  // 调用pipe创建管道
  // 这个地方会创建3个pipe，也就是有6个文件描述符（一个pipe两个文件描述符是因为管道一端读，另一端写）
  // 创建的3个文件描述符的作用分别是：
  // 1. parent -> children data
  // 2. children -> parent ack
  // 3. parent -> children ack
  // 创建出来的6个文件描述符再放到server对象上，主要是用来父子进程之间通信
  // server.aof_pipe_write_data_to_child = fds[1];
  // server.aof_pipe_read_data_from_parent = fds[0];
  // server.aof_pipe_write_ack_to_parent = fds[3];
  // server.aof_pipe_read_ack_from_child = fds[2];
  // server.aof_pipe_write_ack_to_child = fds[5];
  // server.aof_pipe_read_ack_from_parent = fds[4];
  if (aofCreatePipes() != C_OK) return C_ERR;
  // 这是再打开一个pipe，叫child_info_pipe
  // 这个pipe的作用主要是用来汇报进度的
  openChildInfoPipe();
  start = ustime();
  // 调用fork
  // fork是一个神奇的函数，一次调用，两次返回，分别是子线程和父线程
  // 所以执行fork前只有一个进程，但是执行fork之后，后面的代码就是父子线程两个线程都会跑
  // 这也是为什么一个if和else了
  if ((childpid = fork()) == 0) {
    char tmpfile[256];

    /* Child */
    // 这个地方应该是不要监听父线程在用的tcp端口
    closeListeningSockets(0);
    // 设置进程的title
    redisSetProcTitle("redis-aof-rewrite");
    snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
    // 真正的rewrite逻辑
    if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
      size_t private_dirty = zmalloc_get_private_dirty(-1);

      if (private_dirty) {
        serverLog(LL_NOTICE,
                  "AOF rewrite: %zu MB of memory used by copy-on-write",
                  private_dirty/(1024*1024));
      }

      // 做一些子线程退出前的处理
      server.child_info_data.cow_size = private_dirty;
      sendChildInfo(CHILD_INFO_TYPE_AOF);
      exitFromChild(0);
    } else {
      exitFromChild(1);
    }
  } else {
    /* Parent */
    server.stat_fork_time = ustime()-start;
    server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
    // 是否采样
    latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
    // 如果子线程没有打开成功
    if (childpid == -1) {
      closeChildInfoPipe();
      serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
      aofClosePipes();
      return C_ERR;
    }
    serverLog(LL_NOTICE,
              "Background append only file rewriting started by pid %d",childpid);
    // 这个设置为0因为子进程已经在跑了，所以就不用定时了
    server.aof_rewrite_scheduled = 0;
    // 记录一些数据
    server.aof_rewrite_time_start = time(NULL);
    server.aof_child_pid = childpid;
    // 这个是如果子进程运行失败，就允许dict resize，否则就禁止dict resize
    // 因为子进程的原理是copy on write，所以一旦resize会导致大量数据迁移，得不偿失
    updateDictResizePolicy();
    // 这俩不知道干嘛的，貌似和集群有关，之后看到了才来补充
    server.aof_selected_db = -1;
    replicationScriptCacheFlush();
    return C_OK;
  }
  return C_OK; /* unreached */
}
```

这里就是父进程触发AOF子进程的代码，就是在触发子进程前，先准备好pipe，然后通过fork调用创建子进程，然后让子进程单独去跑自己的逻辑，父进程继续执行自己的事情。

一旦子进程执行完毕，自己就会主动退出。

下面分析一下**rewriteAppendOnlyFile**方法具体做了什么事情。

### rewriteAppendOnlyFile

```c
int rewriteAppendOnlyFile(char *filename) {
  rio aof;
  FILE *fp;
  char tmpfile[256];
  char byte;

  // 临时文件的文件名
  snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
  // 创建这个临时文件
  fp = fopen(tmpfile,"w");
  if (!fp) {
    serverLog(LL_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
    return C_ERR;
  }

  // 这个diff初始化
  server.aof_child_diff = sdsempty();
  // rio就是一个io的包，专门用来处理IO操作的，把他当作一个第三方jar包来对待即可
  rioInitWithFile(&aof,fp);

  // 做一些配置
  if (server.aof_rewrite_incremental_fsync)
    rioSetAutoSync(&aof,REDIS_AUTOSYNC_BYTES);

  // 这个东西默认是1，这个的作用就是aof和rdb混合持久化
  if (server.aof_use_rdb_preamble) {
    int error;
    // 这里就是在做AOF之前，先用RDB格式保存全量数据
    // 这个方法具体的原理后面分析RDB的时候再来分析，现在只要知道这里用到了这个功能和特性即可
    if (rdbSaveRio(&aof,&error,RDB_SAVE_AOF_PREAMBLE,NULL) == C_ERR) {
      errno = error;
      goto werr;
    }
  } else {
    // 如果不要混合持久化，那就直接rewrite
    if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
  }

  // parent还在发送数据，这里先fsync一下，主要是为了之后速度能更快
  if (fflush(fp) == EOF) goto werr;
  if (fsync(fileno(fp)) == -1) goto werr;

  int nodata = 0;
  mstime_t start = mstime();
  // 这个就是从父进程的pipe中读取数据
  // 这个是防止时间太长以及没有数据的次数太多
  // 这里要加这个机制的原因是总有人给Redis服务器发送指令，所以父进程永远都会有数据来更新
  // 所以如果这里不采用一些方式停止而是仅仅看是不是有新的数据，那么这个while循环永远都不会停下来
  while(mstime()-start < 1000 && nodata < 20) {
    if (aeWait(server.aof_pipe_read_data_from_parent, AE_READABLE, 1) <= 0)
    {
      nodata++;
      continue;
    }
    // 读到数据了就清空nodata
    nodata = 0;
    // 读取到数据了就处理从父进程度过来的数据
    aofReadDiffFromParent();
  }

  // 让父进程不要再发送diff了
  if (write(server.aof_pipe_write_ack_to_parent,"!",1) != 1) goto werr;
  // 等父进程发送ACK
  if (anetNonBlock(NULL,server.aof_pipe_read_ack_from_parent) != ANET_OK)
    goto werr;
 	// 阻塞，防止子进程无法关闭
  if (syncRead(server.aof_pipe_read_ack_from_parent,&byte,1,5000) != 1 ||
      byte != '!') goto werr;
  serverLog(LL_NOTICE,"Parent agreed to stop sending diffs. Finalizing AOF...");

  // 因为之前告诉父进程不要diff了，但是在这通知一来一回过程中可能还有一些多余的部分，这里最后处理一下
  aofReadDiffFromParent();

  // 日志
  serverLog(LL_NOTICE,
            "Concatenating %.2f MB of AOF diff received from parent.",
            (double) sdslen(server.aof_child_diff) / (1024*1024));
  // 写diff
  if (rioWrite(&aof,server.aof_child_diff,sdslen(server.aof_child_diff)) == 0)
    goto werr;

  // flush和fsync文件，并且最后关闭文件描述符
  if (fflush(fp) == EOF) goto werr;
  if (fsync(fileno(fp)) == -1) goto werr;
  if (fclose(fp) == EOF) goto werr;

  // 前面写的时候用的是temp的文件名，现在用新的文件名
  if (rename(tmpfile,filename) == -1) {
    serverLog(LL_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
    unlink(tmpfile);
    return C_ERR;
  }
  // 日志
  serverLog(LL_NOTICE,"SYNC append only file rewrite performed");
  return C_OK;

  // 错误处理
  werr:
  serverLog(LL_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
  fclose(fp);
  unlink(tmpfile);
  return C_ERR;
}
```

上面这段代码虽然很长，但是总体氛围几个部分。

- 首先初始化文件相关的内容
- 其次根据配置是混合持久化还是单纯的AOF
- 然后一小段时间内读取服务端的diff
- 给父进程发送结束的ACK，然后等待父进程的确认ACK
- 把这个时间差内的diff再写到文件里
- 最后关闭文件等收尾工作

首先分析其中第一个关键部分，原始的AOF。

## rewriteAppendOnlyFileRio

首先我们要分析清楚最初的AOF都做了些什么事情，然后再看混合持久化相比之下有什么样的优势。

```c
int rewriteAppendOnlyFileRio(rio *aof) {
  dictIterator *di = NULL;
  dictEntry *de;
  size_t processed = 0;
  int j;

  // 遍历所有的db
  for (j = 0; j < server.dbnum; j++) {
    char selectcmd[] = "*2\r\n$6\r\nSELECT\r\n";
    // 获取到db
    redisDb *db = server.db+j;
    // 获取到db的dict
    dict *d = db->dict;
    if (dictSize(d) == 0) continue;
    // 获取Iterator迭代器
    di = dictGetSafeIterator(d);

    // 把selectcmd这个内容写到aof文件中
    if (rioWrite(aof,selectcmd,sizeof(selectcmd)-1) == 0) goto werr;
    // 把j这个数字也写下来
    if (rioWriteBulkLongLong(aof,j) == 0) goto werr;

    // 迭代Redis中每一个数据
    while((de = dictNext(di)) != NULL) {
      sds keystr;
      robj key, *o;
      long long expiretime;

      // 获取key
      keystr = dictGetKey(de);
      // 获取value
      o = dictGetVal(de);
      initStaticStringObject(key,keystr);

      // 获取过期时间
      expiretime = getExpire(db,&key);

      // 根据不同的数据类型，写不同类型的AOF日志文件
      if (o->type == OBJ_STRING) {
        /* Emit a SET command */
        char cmd[]="*3\r\n$3\r\nSET\r\n";
        if (rioWrite(aof,cmd,sizeof(cmd)-1) == 0) goto werr;
        /* Key and value */
        if (rioWriteBulkObject(aof,&key) == 0) goto werr;
        if (rioWriteBulkObject(aof,o) == 0) goto werr;
      } else if (o->type == OBJ_LIST) {
        if (rewriteListObject(aof,&key,o) == 0) goto werr;
      } else if (o->type == OBJ_SET) {
        if (rewriteSetObject(aof,&key,o) == 0) goto werr;
      } else if (o->type == OBJ_ZSET) {
        if (rewriteSortedSetObject(aof,&key,o) == 0) goto werr;
      } else if (o->type == OBJ_HASH) {
        if (rewriteHashObject(aof,&key,o) == 0) goto werr;
      } else if (o->type == OBJ_STREAM) {
        if (rewriteStreamObject(aof,&key,o) == 0) goto werr;
      } else if (o->type == OBJ_MODULE) {
        if (rewriteModuleObject(aof,&key,o) == 0) goto werr;
      } else {
        serverPanic("Unknown object type");
      }
      // 保存expire过期时间
      if (expiretime != -1) {
        char cmd[]="*3\r\n$9\r\nPEXPIREAT\r\n";
        if (rioWrite(aof,cmd,sizeof(cmd)-1) == 0) goto werr;
        if (rioWriteBulkObject(aof,&key) == 0) goto werr;
        if (rioWriteBulkLongLong(aof,expiretime) == 0) goto werr;
      }
      // 从父进程中看是不是有diff文件发过来，如果有的话就写一点
      if (aof->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES) {
        processed = aof->processed_bytes;
        aofReadDiffFromParent();
      }
    }
    // 释放内存
    dictReleaseIterator(di);
    di = NULL;
  }
  return C_OK;

  // 处理错误
  werr:
  if (di) dictReleaseIterator(di);
  return C_ERR;
}
```

从这个流程中可以看出来的是所谓AOF，就是一个迭代器迭代dict中存储的每一个数据，然后把这些数据的最终状态作为一个Redis的插入指令写到AOF文件中。

虽然代码比较长，处理的情况也比较多，但是非常好理解。

### 混合持久化

上面这段是Redis以前老的逻辑，目前默认是使用混合持久化。所谓混合持久化就是前面用AOF的格式写的Redis全量数据改为用RDB格式写，RDB是一个二进制格式，所以总体上会更紧凑。

### AOF和混合持久化的优缺点

#### AOF　

优点：

1. 数据更完整，秒级数据丢失(取决于设置fsync策略)。
2. 兼容性较高，由于是基于redis通讯协议而形成的命令追加方式，无论何种版本的redis都兼容，再者aof文件是明文的，可阅读性较好。

缺点：

1. 数据文件体积较大，即使有重写机制，但是在相同的数据集情况下，AOF文件通常比RDB文件大。
2. 相对RDB方式，AOF速度慢于RDB（因为要生成命令），并且在数据量大时候，恢复速度AOF速度也是慢于RDB（因为要通过执行命令）。

####混合持久化

优点：

1. 混合持久化结合了RDB持久化 和 AOF 持久化的优点， 由于绝大部分都是RDB格式，加载速度快，同时结合AOF，增量的数据以AOF方式保存了，数据更少的丢失。

缺点：

1. 老版本不支持（这也算缺点？哪个东西敢拍着胸脯说老版本永远支持？所以混合持久化没有明显的缺点，版本升级到4.0以上就可以使用）

## AOF diff追加何时出发

前面分析源码的时候又看到，子进程会从**aof_pipe_read_data_from_parent**这个pipe中读取父进程变更的数据来追加到AOF文件中，那么这个地方的数据是从哪里发过来的呢？

其实前面也有看到过，但是没有仔细分析，就是在propagate方法中会判断。

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
               int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        feedAppendOnlyFile(cmd,dbid,argv,argc);
    if (flags & PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc);
}

void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
  // 省略其他内容

  // 这个地方就是给子进程发送的地方
  if (server.aof_child_pid != -1)
    aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

  sdsfree(buf);
}
```

上面这段代码就是每执行一个Redis命令，Redis都会判断是不是需要**propagate**（并不是每一个命令都需要，因为有的就是读取而已）。不过如果需要propagate的话，就会通过**aof_child_pid**是不是存在来判断是不是需要给子进程发送diff的数据。

下面具体分析一下**aofRewriteBufferAppend**的实现，这个方法并不是简单的放到的pipe里就完事的，还是值得分析的。

```c
void aofRewriteBufferAppend(unsigned char *s, unsigned long len) {
  // 获取aof_rewrite_buf_blocks最后一个节点
  listNode *ln = listLast(server.aof_rewrite_buf_blocks);
  // 获取block块
  aofrwblock *block = ln ? ln->value : NULL;

  // 只要还有数据可以写
  while(len) {
    // 如果已经有block了，就尝试向这个block中追加数据
    if (block) {
      unsigned long thislen = (block->free < len) ? block->free : len;
      // block还有多余的空间可以写
      if (thislen) {
        memcpy(block->buf+block->used, s, thislen);
        block->used += thislen;
        block->free -= thislen;
        s += thislen;
        len -= thislen;
      }
    }

    // 这个地方就是之前的block不存在，就是分配一个新的block
    if (len) {
      int numblocks;

      // 分配内存
      block = zmalloc(sizeof(*block));
      // 10mb一个block
      block->free = AOF_RW_BUF_BLOCK_SIZE;
      block->used = 0;
      // 把这个block放到aof_rewrite_buf_blocks队列中
      listAddNodeTail(server.aof_rewrite_buf_blocks,block);

      // 超过10个就打日志，超过100就warning
      numblocks = listLength(server.aof_rewrite_buf_blocks);
      if (((numblocks+1) % 10) == 0) {
        int level = ((numblocks+1) % 100) == 0 ? LL_WARNING :
        LL_NOTICE;
        serverLog(level,"Background AOF buffer size: %lu MB",
                  aofRewriteBufferSize()/(1024*1024));
      }
    }
  }

  // 观察EventLoop的FileEvent中是不是存在针对aof_pipe_write_data_to_child这个fd的事件aeFileEvent
  // 如果存在就不管，否则就往EventLoop中添加一个aeFileEvent，这个event具体执行的方法是aofChildWriteDiffData
  if (aeGetFileEvents(server.el,server.aof_pipe_write_data_to_child) == 0) {
    aeCreateFileEvent(server.el, server.aof_pipe_write_data_to_child,
                      AE_WRITABLE, aofChildWriteDiffData, NULL);
  }
}
```

上面这段的内容就是把要追加的东西放到一个block链表中，然后确保EventLoop中存在一个针对aof_pipe_write_data_to_child（这个就是一个fd）的aeFileEvent。

> 这里复习一下，EventLoop就是不停的处理各种FileEvent，主要分成读和写两种，比如处理Redis命令就是注册了一个读事件，这里就是注册一个写事件。
>
> 时间循环就是不停的处理各种事件的循环。

所以这里就是在EventLoop中注册一个事件，之后在EventLoop的过程中会执行这个事件。

###aofChildWriteDiffData

不过既然是事件，总有事件对应的方法。

**aofChildWriteDiffData**这个函数就是给子进程diff时所使用到的方法。

其实很简单，前面说到了添加的数据会放到**aof_rewrite_buf_blocks**这个链表上，这个函数的作用就是不停的读取这个链表上的内容，然后通过pipe发送个子进程就好了。

```c
void aofChildWriteDiffData(aeEventLoop *el, int fd, void *privdata, int mask) {
  listNode *ln;
  aofrwblock *block;
  ssize_t nwritten;
  UNUSED(el);
  UNUSED(fd);
  UNUSED(privdata);
  UNUSED(mask);

  while(1) {
    ln = listFirst(server.aof_rewrite_buf_blocks);
    block = ln ? ln->value : NULL;
    if (server.aof_stop_sending_diff || !block) {
      aeDeleteFileEvent(server.el,server.aof_pipe_write_data_to_child,
                        AE_WRITABLE);
      return;
    }
    if (block->used > 0) {
      // 往pipe里面写
      nwritten = write(server.aof_pipe_write_data_to_child,
                       block->buf,block->used);
      if (nwritten <= 0) return;
      memmove(block->buf,block->buf+nwritten,block->used-nwritten);
      block->used -= nwritten;
      block->free += nwritten;
    }
    // 写完了就把对应的节点删除
    if (block->used == 0) listDelNode(server.aof_rewrite_buf_blocks,ln);
  }
}
```