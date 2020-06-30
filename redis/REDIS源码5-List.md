#REDIS源码5-List

以前Redis的List的基本实现是当数据比较少的时候，就用ziplist，当数据比较多的时候就用双向链表。但是双向链表是一个比较浪费空间的数据结构，因为前指针和后指针就要占据16个字节，另外每一个节点的内存都是单独分配，大量的链表节点单独分配就会造成内存的碎片化。同时针对链表的遍历不像一整块连续内存那样局部性比较好，所以Redis现在（3.2之后）使用了quicklist来代替ziplist和双向链表这样的数据结构。

其实所谓的quicklist也很无赖，还是一个双向链表。。。是的，还是双向链表，只不过之前双向链表每个节点都是存储数据，现在每个节点存储的是一个ziplist。。。

是的，这就是quicklist。。。

<img src="/Users/xiwang/Desktop/分享/redis/imgs/quicklist数据结构吐槽.png" alt="image-20200220222121236" style="zoom:33%;" />

不过既然要做Redis源码分析，还是应该要从源码入手，细致的分析一些实现的思路。

## QuickList

```c
typedef struct quicklist {
    quicklistNode *head; // 头指针，顾名思义
    quicklistNode *tail; // 尾指针，顾名思义
    unsigned long count; // 由于每一个node都是ziplist，这个count就是左右ziplist里面entry的综合
    unsigned long len; // quicklistNode的总数
    int fill : 16; // 填充因子，作用就是控制每一个ziplist的entry个数或大小
    unsigned int compress : 16; // LZF压缩算法的压缩深度，比如如果是1，那就是头尾1个节点不压缩，中间的都压缩，如果是2，就是头尾各2个共4个不压缩，中间的压缩
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev; // 前驱节点
    struct quicklistNode *next; // 后记节点
    unsigned char *zl; // 压缩列表指针
    unsigned int sz; // ziplist的字节大小
    unsigned int count : 16; // ziplist的entry个数
    unsigned int encoding : 2; /* RAW==1 or LZF==2 */
    unsigned int container : 2; /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; // 是否因为要使用所以临时解压缩
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; // 暂时没用，凑足16个字节用的
} quicklistNode;
```

在了解了基本的结构之后，通过几个Redis经典的对于list的操作命令，来具体看一下Redis是如何使用quicklist实现功能的。

## 常见命令原理分析

###push命令

```c
void lpushCommand(client *c) {
  pushGenericCommand(c,LIST_HEAD);
}

void rpushCommand(client *c) {
  pushGenericCommand(c,LIST_TAIL);
}
```

lpush和rpush就是一个从左边push，一个从右边push，其实现是一样的，只不过传入的参数不同，那么重点分析一下**pushGenericCommand**方法

```c
void pushGenericCommand(client *c, int where) {
  int j, pushed = 0;
  // 这个就是从全局的字典中找到RedisObject
  robj *lobj = lookupKeyWrite(c->db,c->argv[1]);

  // 如果RedisObject存在且不是List类型
  if (lobj && lobj->type != OBJ_LIST) {
    addReply(c,shared.wrongtypeerr);
    return;
  }

  for (j = 2; j < c->argc; j++) {
    // 如果RedisObject不存在，就创建一个放到db的dict里面
    if (!lobj) {
      // 创建一个RedisObject，然后指针指向quicklist
      lobj = createQuicklistObject();
      // 配置参数
      // list_max_ziplist_size默认为-2，就是一个ziplist最大不超过8k
      // list_compress_depth默认为0，就是不压缩；这个压缩主要是list很长，而且主要访问两端的数据，所以中间压缩会有一定的性能提升和内存节省，默认是不压缩的
      quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                          server.list_compress_depth);
      // 新建出来的这个东西放到dict中
      dbAdd(c->db,c->argv[1],lobj);
    }
    // 调用push
    listTypePush(lobj,c->argv[j],where);
    pushed++;
  }
  // 添加返回
  addReplyLongLong(c, (lobj ? listTypeLength(lobj) : 0));
  // 如果有push内容，说明有修改
  if (pushed) {
    char *event = (where == LIST_HEAD) ? "lpush" : "rpush";

    // 发布修改的事件
    signalModifiedKey(c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id);
  }
  server.dirty += pushed;
}
```

主要需要着重分析的是**listTypePush**方法

```c
void listTypePush(robj *subject, robj *value, int where) {
  if (subject->encoding == OBJ_ENCODING_QUICKLIST) {
    int pos = (where == LIST_HEAD) ? QUICKLIST_HEAD : QUICKLIST_TAIL;
    value = getDecodedObject(value);
    size_t len = sdslen(value->ptr);
    // 重点实现
    quicklistPush(subject->ptr, value->ptr, len, pos);
    decrRefCount(value);
  } else {
    serverPanic("Unknown list encoding");
  }
}

void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
  // 虽然分头尾，但是实现基本一致
  if (where == QUICKLIST_HEAD) {
    quicklistPushHead(quicklist, value, sz);
  } else if (where == QUICKLIST_TAIL) {
    quicklistPushTail(quicklist, value, sz);
  }
}
```

头尾的push就以push头为例，push尾基本原理一致。

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
  quicklistNode *orig_head = quicklist->head;
  // 如果头节点的ziplist允许添加元素，就直接放到头节点的ziplist中
  if (likely(
    _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
    quicklist->head->zl =
      // push节点
      ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
    // quicklist更新长度
    quicklistNodeUpdateSz(quicklist->head);
  } else {
    // 否则就创建一个新的头节点
    quicklistNode *node = quicklistCreateNode();
    // 头节点新建一个新的ziplist，然后放到这个新的ziplist
    node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);

    quicklistNodeUpdateSz(node);
    // 然后更新头节点
    _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
  }
  
  // 更新一些计数信息
  quicklist->count++;
  quicklist->head->count++;
  return (orig_head != quicklist->head);
}
```

### pop命令

```c
void popGenericCommand(client *c, int where) {
  robj *o = lookupKeyWriteOrReply(c,c->argv[1],shared.nullbulk);
  if (o == NULL || checkType(c,o,OBJ_LIST)) return;

  // 弹出一个元素
  robj *value = listTypePop(o,where);
  if (value == NULL) {
    // 为空直接回复
    addReply(c,shared.nullbulk);
  } else {
    char *event = (where == LIST_HEAD) ? "lpop" : "rpop";

    // 回复
    addReplyBulk(c,value);
    // 引用-1，如果到0就释放内存
    decrRefCount(value);
    // 发布事件
    notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id);
    // 如果这个list没东西了，就从dict里面删除
    if (listTypeLength(o) == 0) {
      notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                          c->argv[1],c->db->id);
      dbDelete(c->db,c->argv[1]);
    }
    signalModifiedKey(c->db,c->argv[1]);
    server.dirty++;
  }
}
```

**listTypePop**具体实现：

```c
robj *listTypePop(robj *subject, int where) {
  long long vlong;
  robj *value = NULL;

  int ql_where = where == LIST_HEAD ? QUICKLIST_HEAD : QUICKLIST_TAIL;
  if (subject->encoding == OBJ_ENCODING_QUICKLIST) {
    // 弹出一个
    if (quicklistPopCustom(subject->ptr, ql_where, (unsigned char **)&value,
                           NULL, &vlong, listPopSaver)) {
			// 如果value为空
      if (!value)
        value = createStringObjectFromLongLong(vlong);
    }
  } else {
    serverPanic("Unknown list encoding");
  }
  return value;
}
```

具体弹出的实现如下：

其实整体逻辑非常好理解，看是从头还是从尾，找到头和尾的node，然后头的话就找到头节点的ziplist删除第一个，如果是尾，就从尾节点的node删除最后一个。

```c
int quicklistPopCustom(quicklist *quicklist, int where, unsigned char **data,
                       unsigned int *sz, long long *sval,
                       void *(*saver)(unsigned char *data, unsigned int sz)) {
  unsigned char *p;
  unsigned char *vstr;
  unsigned int vlen;
  long long vlong;
  // 看是从头开始还是从尾巴开始
  int pos = (where == QUICKLIST_HEAD) ? 0 : -1;

  // 没东西就直接返回
  if (quicklist->count == 0)
    return 0;

  // 数据初始化
  if (data)
    *data = NULL;
  if (sz)
    *sz = 0;
  if (sval)
    *sval = -123456789;

  // 找到对应的node节点，因为是从头弹出或者从尾弹出，所以要找到头或者尾
  quicklistNode *node;
  if (where == QUICKLIST_HEAD && quicklist->head) {
    node = quicklist->head;
  } else if (where == QUICKLIST_TAIL && quicklist->tail) {
    node = quicklist->tail;
  } else {
    return 0;
  }

  // 根据pos获取对应的指针元素，因为从头弹出是要拿ziplist最前面的，从尾拿是要拿ziplist最后一个，因此pos要单独计算一下
  p = ziplistIndex(node->zl, pos);
  // p指针对应的entry是否存在，同时会赋值vstr、vlen和vlong三个字段
  if (ziplistGet(p, &vstr, &vlen, &vlong)) {
    // 这里就是根据不同的类型，值会被放到不同的字段里面
    if (vstr) {
      if (data)
        // saver是一个函数指针
        *data = saver(vstr, vlen);
      if (sz)
        *sz = vlen;
    } else {
      if (data)
        *data = NULL;
      if (sval)
        *sval = vlong;
    }
    // 因为是pop，所以要把对应的元素给删掉
    quicklistDelIndex(quicklist, node, &p);
    return 1;
  }
  return 0;
}
```

###lrange

最后再分析一个lrange，其实大致的做法也很简单，找到quicklist，然后分配一个iterator，然后从头节点开始找到起始的node，因为比如第100-150个元素，那么就要找到第100个是在哪个quicklist node里的ziplist里（quicklist node有存储这个node的ziplist有多少元素，所以只要遍历node，不用遍历node里面的ziplist），然后找到定位到那个元素，逐个往下遍历，如果这个ziplist用完了，就找到下一个quicklist的node，一直到结束为止。