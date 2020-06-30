#REDIS源码23-事务

Redis的事务非常简单，比起MySQL这样的DB，需要支持所有命令全部成功或全部失败，Redis并没有这样的承诺。

Redis的事务主要是由三个命令组成，分别是multi、exec和descard，其中multi是代表了事务的开始，exec代表了事务的执行，dicard表示事务的丢弃。

Redis事务的基本处理逻辑是multi就设置一个flag表示进入事务模式，然后之后发送过来的命令全部都换存在一个执行队列里。因为Redis是单线程的，所以自己执行的时候不用担心被其他Client的请求打断。不过Redis的事务并不具备原子性，全部成功或失败，还有回滚的作用。**Redis的事务之保证隔离性。**

同时Redis还提供了watch命令，watch命令的作用就是有些计算无法使用Redis提供的命令进行原子的计算而必须读取到本机内存中计算，比如乘法、除法甚至平方开方等复杂的计算，这个时候就必须要使用一个指令把数据读取回来，然后计算完毕之后再修改。这样就有一个问题，就是我不知道在我读取回本地计算的过程中是否有人修改过，我计算的值会不会变成一个脏数据。因此可以用watch命令来观察是否有被人修改过。

下面是源码分析，先从multi命令开始。

##MULTI

```c
void multiCommand(client *c) {
  if (c->flags & CLIENT_MULTI) {
    addReplyError(c,"MULTI calls can not be nested");
    return;
  }
  c->flags |= CLIENT_MULTI;
  addReply(c,shared.ok);
}
```

multi命令非常的简洁明了，就是在这个链接的客户端上打一个标，然后就直接返回了。不过这里判断是了是否事务内部嵌套了事务。

## 后续命令入队列

下面这段代码**processCommand**中截取的，

```c
/* Exec the command */
if (c->flags & CLIENT_MULTI &&
    c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
    c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
{
  // 如果已经有了CLIENT_MULTI这个标记，同时要执行的命令不是和事务相关的命令，那么就把这个指令入队
  queueMultiCommand(c);
  addReply(c,shared.queued);
} else {
  // 这里就是正常的执行命令
  call(c,CMD_CALL_FULL);
  c->woff = server.master_repl_offset;
  if (listLength(server.ready_keys))
    handleClientsBlockedOnKeys();
}
```

入队代码也非常简单

```c
/* Add a new command into the MULTI commands queue */
void queueMultiCommand(client *c) {
  multiCmd *mc;
  int j;

  // 多分配一块内存
  c->mstate.commands = zrealloc(c->mstate.commands,
                                sizeof(multiCmd)*(c->mstate.count+1));
  // 这里就是计算出指针的位置
  mc = c->mstate.commands+c->mstate.count;
  mc->cmd = c->cmd;
  mc->argc = c->argc;
  mc->argv = zmalloc(sizeof(robj*)*c->argc);
  memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
  for (j = 0; j < c->argc; j++)
    incrRefCount(mc->argv[j]);
  c->mstate.count++;
  c->mstate.cmd_flags |= c->cmd->flags;
}

// 入队的对象就是这个样子的，保留参数，并且保留要执行的命令
typedef struct multiCmd {
  robj **argv;
  int argc;
  struct redisCommand *cmd;
} multiCmd;
```

## EXEC

上面已经开启了一个事务，并且把事务所需要执行的指令全部入队成功，那么这里就是开始执行事务。

```c
void execCommand(client *c) {
  int j;
  robj **orig_argv;
  int orig_argc;
  struct redisCommand *orig_cmd;
  int must_propagate = 0; // 这个是否需要吧MULTI/EXEC这样的命令给集群中其他节点
  int was_master = server.masterhost == NULL;

  // 如果压根就没有执行过MULTI，就直接报错
  if (!(c->flags & CLIENT_MULTI)) {
    addReplyError(c,"EXEC without MULTI");
    return;
  }

  // 这里是如果被WATCH key已经被修改过了，那么我们就要阻止事务继续执行。
  if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
    addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
             shared.nullmultibulk);
    // 放弃事务
    discardTransaction(c);
    // 直接执行异常流程
    goto handle_monitor;
  }

  // 如果我是一个集群中的只读节点结果让我执行写相关的命令，就直接返回
  if (!server.loading && server.masterhost && server.repl_slave_ro &&
      !(c->flags & CLIENT_MASTER) && c->mstate.cmd_flags & CMD_WRITE)
  {
    addReplyError(c,
                  "Transaction contains write commands but instance "
                  "is now a read-only slave. EXEC aborted.");
    discardTransaction(c);
    goto handle_monitor;
  }

  // 已经符合执行事务的条件，就不用浪费资源去watch了
  unwatchAllKeys(c);
  orig_argv = c->argv;
  orig_argc = c->argc;
  orig_cmd = c->cmd;
  addReplyMultiBulkLen(c,c->mstate.count);
  // 通过FOR循环挨个执行命令
  for (j = 0; j < c->mstate.count; j++) {
    c->argc = c->mstate.commands[j].argc;
    c->argv = c->mstate.commands[j].argv;
    c->cmd = c->mstate.commands[j].cmd;

    // 这个就看是不是MULTI和EXEC命令都要propagete一下
    if (!must_propagate && !(c->cmd->flags & (CMD_READONLY|CMD_ADMIN))) {
      execCommandPropagateMulti(c);
      must_propagate = 1;
    }

    // 执行命令
    call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL);

    /* Commands may alter argc/argv, restore mstate. */
    c->mstate.commands[j].argc = c->argc;
    c->mstate.commands[j].argv = c->argv;
    c->mstate.commands[j].cmd = c->cmd;
  }
  c->argv = orig_argv;
  c->argc = orig_argc;
  c->cmd = orig_cmd;
  // 事务执行完毕，执行discard
  discardTransaction(c);

  // 这个if判断主要是集群的切换，比如之前一个master节点，突然被人切换成了slave，那么在添加一个EXEC命令，保证其他节点至少能把事务执行下去而不会卡住
  if (must_propagate) {
    int is_master = server.masterhost == NULL;
    server.dirty++;
    if (server.repl_backlog && was_master && !is_master) {
      char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
      feedReplicationBacklog(execcmd,strlen(execcmd));
    }
  }

  handle_monitor:
  if (listLength(server.monitors) && !server.loading)
    replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```

上面整个流程非常好理解，开启事务、积攒命令然后一口气全部执行完毕。再看一下discard又做了哪些事情。

##discard

```c
void discardCommand(client *c) {
  if (!(c->flags & CLIENT_MULTI)) {
    addReplyError(c,"DISCARD without MULTI");
    return;
  }
  discardTransaction(c);
  addReply(c,shared.ok);
}

// 上面EXEC也会调用这个方法，就是一个事务的后置处理
void discardTransaction(client *c) {
  // 释放之前的队列
  freeClientMultiState(c);
  // 初始化一个新的
  initClientMultiState(c);
  // 和事务相关的标记清零
  c->flags &= ~(CLIENT_MULTI|CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC);
  // watch的key全部清空，不用浪费资源去watch了
  unwatchAllKeys(c);
}
```

这个释放也很好理解，就不多展开了。

##watch

###添加watch列表

```c
void watchCommand(client *c) {
  int j;

  // watch不能在事务内，因为你要把数据读回去再修改的，如果是事务内，做不到
  if (c->flags & CLIENT_MULTI) {
    addReplyError(c,"WATCH inside MULTI is not allowed");
    return;
  }
  for (j = 1; j < c->argc; j++)
    // 开始watch
    watchForKey(c,c->argv[j]);
  addReply(c,shared.ok);
}
```

看一下watch的实现。

```c
void watchForKey(client *c, robj *key) {
  list *clients = NULL;
  listIter li;
  listNode *ln;
  watchedKey *wk;

  // 每一个Client都有一个watched_keys的list
  listRewind(c->watched_keys,&li);
  // 不断的迭代整个list
  while((ln = listNext(&li))) {
    // 获取node
    wk = listNodeValue(ln);
    // 如果这个node已经被watch了，就不再重复watch了
    if (wk->db == c->db && equalStringObjects(key,wk->key))
      return;
  }

  // 先看这个DB中，要被watch的这个key是不是有一个watch list
  // 这个watch list是记录有哪些Client正在watch这个key的
  clients = dictFetchValue(c->db->watched_keys,key);
  // 没有就创建一个
  if (!clients) {
    clients = listCreate();
    // 创建出来的list和key关联，然后保存在DB上，注意这个和client的watch list不同，这个是一个key被那些client watch，client上的list是一个client watch了哪些key
    dictAdd(c->db->watched_keys,key,clients);
    incrRefCount(key);
  }
  // 把自己这个client连到db中watch特定key的watch列表里
  listAddNodeTail(clients,c);
  // 然后再把这个key给放到client的watched_keys里
  wk = zmalloc(sizeof(*wk));
  wk->key = key;
  wk->db = c->db;
  incrRefCount(key);
  listAddNodeTail(c->watched_keys,wk);
}
```

上面这段代码，尤其是后半部分略微有点绕，但是其实很好理解，是从两个维度来观察的。

1. 我希望知道一个DB中哪些key被哪些Client watch，那么就有一个key: [client1, client2]的映射关系
2. 我还希望知道一个clients watch了哪些key，所以要保留一个client: [key1, key2]的映射关系

### 通知watch列表

代码上非常简单，直接分析源码即可

```c
void signalModifiedKey(redisDb *db, robj *key) {
  touchWatchedKey(db,key);
}

void touchWatchedKey(redisDb *db, robj *key) {
  list *clients;
  listIter li;
  listNode *ln;

  // 如果这个key根本没有watch就直接不管
  if (dictSize(db->watched_keys) == 0) return;
  clients = dictFetchValue(db->watched_keys, key);
  if (!clients) return;

  // 遍历watch list中的每一个client，通过设置一个flag，告诉他们你们watch的key被修改了，不过这里知道被人改了，并不知道具体修改的是哪个key，不过也没关系，知道这点也足够了
  listRewind(clients,&li);
  while((ln = listNext(&li))) {
    client *c = listNodeValue(ln);

    c->flags |= CLIENT_DIRTY_CAS;
  }
}
```

这里的代码非常简单，但是要注意的是**signalModifiedKey**方法，这个方法在Redis源码中被调用的地方非常的多，举几个例子，比如集群的restore，比如expire过期数据，还比如hash、set、stream等之前分析过的数据结构数据被修改了，都会在确认修改成功之后主动调用这个方法。