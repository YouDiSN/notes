#REDIS源码20-RDB(1)

前面分析的AOF是一种增量式的备份方式，虽然也有rewrite这样的重写，但是总的来说AOF的设计目的就是增量备份数据。那么作为一个数据库，Redis自然需要全量备份数据，这就是Redis的RDB机制。

## 触发RDB

触发Redis的RDB主要有两种方式，

- 配置文件中配置的每过多少时间触发一次；
- 还有一种就是手动触发，通过bgsave、save等命令主动触发。

##定时触发

定时触发的入口就是在**serverCron**中，

```c
for (j = 0; j < server.saveparamslen; j++) {
  // saveparams就是在配置文件中配置save的配置
  struct saveparam *sp = server.saveparams+j;

  /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * CONFIG_BGSAVE_RETRY_DELAY seconds already elapsed. */
  // server.dirty >= sp->changes：dirty就是距离上一次bgsave修改的数量，这个的含义就是修改的数据量足够多
  // server.unixtime-server.lastsave > sp->seconds：这个的含义是过去的时间足够多
  // (server.unixtime-server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY || server.lastbgsave_status == C_OK)：这个的含义是bgsave的时间过去的足够长并且上一次bgsave成功了
  if (server.dirty >= sp->changes &&
      server.unixtime-server.lastsave > sp->seconds &&
      (server.unixtime-server.lastbgsave_try >
       CONFIG_BGSAVE_RETRY_DELAY ||
       server.lastbgsave_status == C_OK))
  {
    serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
              sp->changes, (int)sp->seconds);
    rdbSaveInfo rsi, *rsiptr;
    rsiptr = rdbPopulateSaveInfo(&rsi);
    // 执行具体的rdbSave的逻辑
    rdbSaveBackground(server.rdb_filename,rsiptr);
    break;
  }
}
```

上面这段代码就是节选自**serverCron**，就是在TimedEvent执行的时候都会去尝试判断一下是否需要执行rdb，那么具体看一下**rdbPopulateSaveInfo**的实现。

###rdbPopulateSaveInfo

这个方法和AOF的起进程实现非常类似，所以可以快速类比。

```c
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {
  pid_t childpid;
  long long start;

  // 如果已经有子进程了就直接返回
  if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;

  server.dirty_before_bgsave = server.dirty;
  server.lastbgsave_try = time(NULL);
  // 打开传输数据的pipe
  openChildInfoPipe();

  start = ustime();
  // 这里就是fork出一个子进程
  if ((childpid = fork()) == 0) {
    int retval;

    // 子进程基本和AOF一致
    closeListeningSockets(0);
    redisSetProcTitle("redis-rdb-bgsave");
    // 核心rdbSave
    retval = rdbSave(filename,rsi);
    if (retval == C_OK) {
      size_t private_dirty = zmalloc_get_private_dirty(-1);

      if (private_dirty) {
        serverLog(LL_NOTICE,
                  "RDB: %zu MB of memory used by copy-on-write",
                  private_dirty/(1024*1024));
      }

      server.child_info_data.cow_size = private_dirty;
      sendChildInfo(CHILD_INFO_TYPE_RDB);
    }
    exitFromChild((retval == C_OK) ? 0 : 1);
  } else {
    // 父进程和AOF中的基本类似，做一些统计和信息收集的工作，然后就直接退出继续做其他的事情，因为这个父进程就是Redis的主进程，需要继续处理EventLoop的内容
    server.stat_fork_time = ustime()-start;
    server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
    latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
    if (childpid == -1) {
      closeChildInfoPipe();
      server.lastbgsave_status = C_ERR;
      serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
      return C_ERR;
    }
    serverLog(LL_NOTICE,"Background saving started by pid %d",childpid);
    server.rdb_save_time_start = time(NULL);
    server.rdb_child_pid = childpid;
    server.rdb_child_type = RDB_CHILD_TYPE_DISK;
    updateDictResizePolicy();
    return C_OK;
  }
  return C_OK; /* unreached */
}
```

**rdbSave**的具体实现：

```c
int rdbSave(char *filename, rdbSaveInfo *rsi) {
  char tmpfile[256];
  char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */
  FILE *fp;
  rio rdb;
  int error = 0;

  // 同样也是先放到一个 temp文件中
  snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
  fp = fopen(tmpfile,"w");
  if (!fp) {
    char *cwdp = getcwd(cwd,MAXPATHLEN);
    serverLog(LL_WARNING,
              "Failed opening the RDB file %s (in server root dir %s) "
              "for saving: %s",
              filename,
              cwdp ? cwdp : "unknown",
              strerror(errno));
    return C_ERR;
  }

  // 和AOF一样，初始化rio
  rioInitWithFile(&rdb,fp);

  if (server.rdb_save_incremental_fsync)
    rioSetAutoSync(&rdb,REDIS_AUTOSYNC_BYTES);

  // 具体实现逻辑
  if (rdbSaveRio(&rdb,&error,RDB_SAVE_NONE,rsi) == C_ERR) {
    errno = error;
    goto werr;
  }

  // 把数据都刷到磁盘上
  if (fflush(fp) == EOF) goto werr;
  if (fsync(fileno(fp)) == -1) goto werr;
  if (fclose(fp) == EOF) goto werr;

  // 因为RDB结束了，所以就重命名文件
  if (rename(tmpfile,filename) == -1) {
    char *cwdp = getcwd(cwd,MAXPATHLEN);
    serverLog(LL_WARNING,
              "Error moving temp DB file %s on the final "
              "destination %s (in server root dir %s): %s",
              tmpfile,
              filename,
              cwdp ? cwdp : "unknown",
              strerror(errno));
    unlink(tmpfile);
    return C_ERR;
  }

  serverLog(LL_NOTICE,"DB saved on disk");
  // 由于数据已经全部写到RDB中，所以更新一些统计信息，dirty也清零
  server.dirty = 0;
  server.lastsave = time(NULL);
  server.lastbgsave_status = C_OK;
  return C_OK;

werr:
  serverLog(LL_WARNING,"Write error saving DB on disk: %s", strerror(errno));
  fclose(fp);
  unlink(tmpfile);
  return C_ERR;
}
```

RDB的代码到目前为止和AOF的结构基本上都是一致的，关键是**rdbSaveRio**的实现。

```c
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi) {
  dictIterator *di = NULL;
  dictEntry *de;
  char magic[10];
  int j;
  uint64_t cksum;
  size_t processed = 0;

  if (server.rdb_checksum)
    rdb->update_cksum = rioGenericUpdateChecksum;
  snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);
  // 写魔数
  if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
  // 写一些元数据，比如Redis版本、使用了多少内存等
  if (rdbSaveInfoAuxFields(rdb,flags,rsi) == -1) goto werr;
  // modules内容，跳过
  if (rdbSaveModulesAux(rdb, REDISMODULE_AUX_BEFORE_RDB) == -1) goto werr;

  // 这里和AOF基本上是一样的，遍历每一个Redis的DB，然后把每一个RedisDB内的数据都写到文件里
  for (j = 0; j < server.dbnum; j++) {
    redisDb *db = server.db+j;
    dict *d = db->dict;
    if (dictSize(d) == 0) continue;
    di = dictGetSafeIterator(d);

    /* Write the SELECT DB opcode */
    if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr;
    if (rdbSaveLen(rdb,j) == -1) goto werr;

    // 就是写一点统计信息
    uint64_t db_size, expires_size;
    db_size = dictSize(db->dict);
    expires_size = dictSize(db->expires);
    if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;
    if (rdbSaveLen(rdb,db_size) == -1) goto werr;
    if (rdbSaveLen(rdb,expires_size) == -1) goto werr;

    // 遍历每一个数据
    while((de = dictNext(di)) != NULL) {
      sds keystr = dictGetKey(de);
      robj key, *o = dictGetVal(de);
      long long expire;

      initStaticStringObject(key,keystr);
      expire = getExpire(db,&key);
      // 根据不同类型写文件
      if (rdbSaveKeyValuePair(rdb,&key,o,expire) == -1) goto werr;

      if (flags & RDB_SAVE_AOF_PREAMBLE &&
          rdb->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES)
      {
        // 汇报自己的进度，读取父进程的diff数据，就是把pipe里的数据写到aof_child_diff里，避免长时间不读不写一下子累计了很多数据
        processed = rdb->processed_bytes;
        aofReadDiffFromParent();
      }
    }
    // 这里就是一个dict处理完毕了，就释放对应的内存
    dictReleaseIterator(di);
    di = NULL; /* So that we don't release it again on error. */
  }

  // lua相关的处理
  if (rsi && dictSize(server.lua_scripts)) {
    di = dictGetIterator(server.lua_scripts);
    while((de = dictNext(di)) != NULL) {
      robj *body = dictGetVal(de);
      if (rdbSaveAuxField(rdb,"lua",3,body->ptr,sdslen(body->ptr)) == -1)
        goto werr;
    }
    dictReleaseIterator(di);
    di = NULL; /* So that we don't release it again on error. */
  }

  // 模块
  if (rdbSaveModulesAux(rdb, REDISMODULE_AUX_AFTER_RDB) == -1) goto werr;

  // 文件中终止
  if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;

  // CRC64 checksum
  cksum = rdb->cksum;
  memrev64ifbe(&cksum);
  if (rioWrite(rdb,&cksum,8) == 0) goto werr;
  return C_OK;

  // 异常处理
  werr:
  if (error) *error = errno;
  if (di) dictReleaseIterator(di);
  return C_ERR;
}
```

从上面的流程可以看到整体上RDB的流程和AOF几乎是一样的，从创建pipe，到fork进程，到父进程写一些统计数据、子进程不断的迭代Redis的数据写到RDB文件中。