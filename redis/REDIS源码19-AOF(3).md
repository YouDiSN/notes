#REDIS源码19-AOF(3)

## 数据恢复

前面分析都是Redis是如何写AOF日志的，那么写完之后肯定是要用这个日志做一点事情的，比如DB启动的时候就会去读取AOF日志来回复自己的数据。

## 通过AOF文件恢复

在Redis的main函数中，initServer结束之后，就会调用**loadDataFromDisk**方法， 这个方法的具体实现如下：

```c
void loadDataFromDisk(void) {
  long long start = ustime();
  // AOF恢复数据
  if (server.aof_state == AOF_ON) {
    // loadAppendOnlyFile就是具体实现
    if (loadAppendOnlyFile(server.aof_filename) == C_OK)
      // 回复成功打一个日志
      serverLog(LL_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
  } else {
    // RDB恢复数据，省略
  }
}
```

**loadAppendOnlyFile**这个方法非常长，分段来分析，第一个段是尝试从AOF文件中恢复数据：

```c
int loadAppendOnlyFile(char *filename) {
  struct client *fakeClient;
  FILE *fp = fopen(filename,"r");
  struct redis_stat sb;
  int old_aof_state = server.aof_state;
  long loops = 0;
  off_t valid_up_to = 0; /* Offset of latest well-formed command loaded. */
  off_t valid_before_multi = 0; /* Offset before MULTI command loaded. */

  // 如果AOF文件为空，就直接返回
  if (fp && redis_fstat(fileno(fp),&sb) != -1 && sb.st_size == 0) {
    server.aof_current_size = 0;
    server.aof_fsync_offset = server.aof_current_size;
    fclose(fp);
    return C_ERR;
  }

  /* Temporarily disable AOF, to prevent EXEC from feeding a MULTI
     * to the same file we're about to read. */
  // 临时关闭一下AOF，因为避免Redis Client发送指令再写AOF文件
  server.aof_state = AOF_OFF;

  // 创建一个假装的Client
  fakeClient = createFakeClient();
  startLoading(fp);

  char sig[5]; /* "REDIS" */
  // 这个看是不是混合持久化
  if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
    /* No RDB preamble, seek back at 0 offset. */
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
  } else {
    // 这里是发现是混合持久化，就先读取RDB文件
    rio rdb;

    serverLog(LL_NOTICE,"Reading RDB preamble from AOF file...");
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    rioInitWithFile(&rdb,fp);
    // RDB读取
    if (rdbLoadRio(&rdb,NULL,1) != C_OK) {
      serverLog(LL_WARNING,"Error reading the RDB preamble of the AOF file, AOF loading aborted");
      goto readerr;
    } else {
      serverLog(LL_NOTICE,"Reading the remaining AOF tail...");
    }
  }

  // 不断的读取AOF文件
  while(1) {
    int argc, j;
    unsigned long len;
    robj **argv;
    char buf[128];
    sds argsds;
    struct redisCommand *cmd;

    /* Serve the clients from time to time */
    if (!(loops++ % 1000)) {
      // 每过一段上报一下进度
      loadingProgress(ftello(fp));
      // 这个是因为Redis是单线程的，防止一直在读取AOF文件导致Server启动起来但是EventLoop无法处理事件，不过根据源码中的貌似这里只会处理一些定时任务，客户端的请求不会去执行（当然也不能执行，因为数据都没恢复现在去执行客户端请求是会有错误的）
      processEventsWhileBlocked();
    }

    // 把文件读取到buf中
    if (fgets(buf,sizeof(buf),fp) == NULL) {
      // 如果是末尾，就终止
      if (feof(fp))
        break;
      else
        goto readerr;
    }
    // 根据AOF文件格式不停的读取内容
    if (buf[0] != '*') goto fmterr;
    if (buf[1] == '\0') goto readerr;
    argc = atoi(buf+1);
    if (argc < 1) goto fmterr;

    argv = zmalloc(sizeof(robj*)*argc);
    fakeClient->argc = argc;
    fakeClient->argv = argv;

    for (j = 0; j < argc; j++) {
      // 继续读取指令的参数
      if (fgets(buf,sizeof(buf),fp) == NULL) {
        fakeClient->argc = j; /* Free up to j-1. */
        freeFakeClientArgv(fakeClient);
        goto readerr;
      }
      if (buf[0] != '$') goto fmterr;
      len = strtol(buf+1,NULL,10);
      argsds = sdsnewlen(SDS_NOINIT,len);
      if (len && fread(argsds,len,1,fp) == 0) {
        sdsfree(argsds);
        fakeClient->argc = j; /* Free up to j-1. */
        freeFakeClientArgv(fakeClient);
        goto readerr;
      }
      argv[j] = createObject(OBJ_STRING,argsds);
      if (fread(buf,2,1,fp) == 0) {
        fakeClient->argc = j+1; /* Free up to j. */
        freeFakeClientArgv(fakeClient);
        goto readerr; /* discard CRLF */
      }
    }

    // 上面就是读取文件内容，然后解析成一个完整的指令
    cmd = lookupCommand(argv[0]->ptr);
    if (!cmd) {
      serverLog(LL_WARNING,
                "Unknown command '%s' reading the append only file",
                (char*)argv[0]->ptr);
      exit(1);
    }

    if (cmd == server.multiCommand) valid_before_multi = valid_up_to;

    // 使用这个FakeClient当作Redis的客户端
    // 这里的目的是因为每一个Redis请求都需要对应的RedisClient才能执行，但是这里因为没有真正的RedisClient，所以就用了自己的FakeClient来处理
    fakeClient->cmd = cmd;
    if (fakeClient->flags & CLIENT_MULTI &&
        fakeClient->cmd->proc != execCommand)
    {
      queueMultiCommand(fakeClient);
    } else {
      // 执行指令
      cmd->proc(fakeClient);
    }

    // Clean up
    freeFakeClientArgv(fakeClient);
    fakeClient->cmd = NULL;
    if (server.aof_load_truncated) valid_up_to = ftello(fp);
  }

  /* This point can only be reached when EOF is reached without errors.
     * If the client is in the middle of a MULTI/EXEC, handle it as it was
     * a short read, even if technically the protocol is correct: we want
     * to remove the unprocessed tail and continue. */
  if (fakeClient->flags & CLIENT_MULTI) {
    serverLog(LL_WARNING,
              "Revert incomplete MULTI/EXEC transaction in AOF file");
    valid_up_to = valid_before_multi;
    goto uxeof;
  }
}
```

上面这段代码就是读取文件，然后解析文件的参数，然后假装自己是一个RedisClient来执行从AOF文件中读取到的命令然后插入的Redis中。

虽然流程很长，但是非常的清晰。其中不间断的再去处理一些EventLoop的event。

第二段就是处理数据被正常的加载到Redis中，

```c
loaded_ok:
	// 关闭fp
  fclose(fp);
	// 释放内存
  freeFakeClient(fakeClient);
  server.aof_state = old_aof_state;
  stopLoading();
	// 收集一些统计信息
  aofUpdateCurrentSize();
  server.aof_rewrite_base_size = server.aof_current_size;
  server.aof_fsync_offset = server.aof_current_size;
  return C_OK;
```

第三段是处理文件读取错误

```c
readerr:
	// 如果是文件终止符 
  if (!feof(fp)) {
    // 释放内存
    if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
    serverLog(LL_WARNING,"Unrecoverable error reading the append only file: %s", strerror(errno));
    // 退出进程
    exit(1);
  }
```

第四段是期望读取到AOF数据，结果读取到了文件终止符。

```c
uxeof: /* Unexpected AOF end of file. */
	// 这是一个配置，就是如果错误读取到了文件终止符是否还要继续，默认是true
  if (server.aof_load_truncated) {
    if (valid_up_to == -1 || truncate(filename,valid_up_to) == -1) {
      if (valid_up_to == -1) {
        serverLog(LL_WARNING,"Last valid command offset is invalid");
      } else {
        serverLog(LL_WARNING,"Error truncating the AOF file: %s",
                  strerror(errno));
      }
    } else {
      /* Make sure the AOF file descriptor points to the end of the
               * file after the truncate call. */
      if (server.aof_fd != -1 && lseek(server.aof_fd,0,SEEK_END) == -1) {
        serverLog(LL_WARNING,"Can't seek the end of the AOF file: %s",
                  strerror(errno));
      } else {
        serverLog(LL_WARNING,
                  "AOF loaded anyway because aof-load-truncated is enabled");
        // 这里就是当作成功，因为已经做了配置出错也继续正常执行
        goto loaded_ok;
      }
    }
  }
	// 这里就是出错要停止
  if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
  exit(1);
```

最后一个就是文件格式错误的处理：

```c
fmterr: /* Format error. */
	// 老规矩，释放内存
  if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
	// 记录日志
  serverLog(LL_WARNING,"Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>");
  exit(1);
```

