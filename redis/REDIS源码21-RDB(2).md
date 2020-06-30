#REDIS源码21-RDB(2)

第一个章节分析了background RDB，也就是定时的自动RDB，这里分析一下使用命令的RDB，由于流程上基本和自动RDB是一致的，所以快速带过，在之后分析一下如何从RDB恢复数据。

## 手动RDB

直接看源码实现，非常简单

```c
void saveCommand(client *c) {
  if (server.rdb_child_pid != -1) {
    addReplyError(c,"Background save already in progress");
    return;
  }
  rdbSaveInfo rsi, *rsiptr;
  rsiptr = rdbPopulateSaveInfo(&rsi);
  // 直接调用rdbSave，原理和之前一致
  if (rdbSave(server.rdb_filename,rsiptr) == C_OK) {
    addReply(c,shared.ok);
  } else {
    addReply(c,shared.err);
  }
}

/* BGSAVE [SCHEDULE] */
void bgsaveCommand(client *c) {
  int schedule = 0;

  // 这个参数就是是否需要schedule
  if (c->argc > 1) {
    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"schedule")) {
      schedule = 1;
    } else {
      addReply(c,shared.syntaxerr);
      return;
    }
  }

  rdbSaveInfo rsi, *rsiptr;
  rsiptr = rdbPopulateSaveInfo(&rsi);

  if (server.rdb_child_pid != -1) {
    addReplyError(c,"Background save already in progress");
  } else if (server.aof_child_pid != -1) {
    if (schedule) {
      server.rdb_bgsave_scheduled = 1;
      addReplyStatus(c,"Background saving scheduled");
    } else {
      addReplyError(c,
                    "An AOF log rewriting in progress: can't BGSAVE right now. "
                    "Use BGSAVE SCHEDULE in order to schedule a BGSAVE whenever "
                    "possible.");
    }
  } else if (rdbSaveBackground(server.rdb_filename,rsiptr) == C_OK) { // 这里就是通过开启子进程的方式来做RDB，具体实现和服务端自动的save是完全一样的。
    addReplyStatus(c,"Background saving started");
  } else {
    addReply(c,shared.err);
  }
}
```

所以从上面的源码可以看出来，通过命令执行的RDB和之前的逻辑几乎没有区别。

## 数据恢复

和AOF一样，也是在**rdbLoad**方法中的**loadDataFromDisk**，

```c
void loadDataFromDisk(void) {
  long long start = ustime();
  if (server.aof_state == AOF_ON) {
    // AOF
    if (loadAppendOnlyFile(server.aof_filename) == C_OK)
      serverLog(LL_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
  } else {
    // RDB
    rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
    // rdbLoad就是真正的实现逻辑
    if (rdbLoad(server.rdb_filename,&rsi) == C_OK) {
      serverLog(LL_NOTICE,"DB loaded from disk: %.3f seconds",
                (float)(ustime()-start)/1000000);

      // 处理集群相关逻辑，之后分析集群相关逻辑时再来分析
      if ((server.masterhost ||
           (server.cluster_enabled &&
            nodeIsSlave(server.cluster->myself))) &&
          rsi.repl_id_is_set &&
          rsi.repl_offset != -1 &&
          /* Note that older implementations may save a repl_stream_db
                 * of -1 inside the RDB file in a wrong way, see more
                 * information in function rdbPopulateSaveInfo. */
          rsi.repl_stream_db != -1)
      {
        memcpy(server.replid,rsi.repl_id,sizeof(server.replid));
        server.master_repl_offset = rsi.repl_offset;
        /* If we are a slave, create a cached master from this
                 * information, in order to allow partial resynchronizations
                 * with masters. */
        replicationCacheMasterUsingMyself();
        selectDb(server.cached_master,rsi.repl_stream_db);
      }
    } else if (errno != ENOENT) {
      serverLog(LL_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
      exit(1);
    }
  }
}
```

**rdbLoad**具体实现如下：

基本上和AOF都是一样的，就是一个是AOF一个是RDB而已，其他的流程甚至是代码的结构几乎都是一样的。

```c
int rdbLoad(char *filename, rdbSaveInfo *rsi) {
  FILE *fp;
  rio rdb;
  int retval;

  if ((fp = fopen(filename,"r")) == NULL) return C_ERR;
  startLoading(fp);
  rioInitWithFile(&rdb,fp);
  // 具体实现
  retval = rdbLoadRio(&rdb,rsi,0);
  fclose(fp);
  stopLoading();
  return retval;
}
```

**rdbLoadRio**这个方法比较长，具体就不贴了，和AOF基本上也是一样的，就是不断的读取RDB的文件内容，然后调用dictAdd的方法把load到的数据插入DB中即可。就直接跳过了不再仔细的一行行去分析了。