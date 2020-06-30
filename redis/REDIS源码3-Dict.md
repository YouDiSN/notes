#REDIS源码3-Dict

Redis本身其实就一个巨大的字典结构，所以dict这个数据结构对于Redis的重要程度不言而喻，而且redis内部使用Dict的地方非常多。

## Dict基本设计

一个Dict有两个hashtable，一个是原始的存储数据的hashtable，还有一个是扩容的过程中使用的hashtable，之所以需要两个的原因是，redis的扩容时一个渐进的过程，所以需要同时存在两个hashtable。

hashtable的基本结构和Java的HashMap基本类似，也是桶+链表的结构。

```c
typedef struct dict {
    dictType *type; // 针对这个dict数据结构操作的一个结构体，这个和结构体里面都是一堆的函数指针，可以通过这些函数指针对key和value进行操作
    void *privdata; // 私有数据，不这干嘛的
    dictht ht[2];   // 两个dicthashtable，一个用来存储是数据，另一个用来扩容
    long rehashidx; // hashtable扩容，rehash的进度
    unsigned long iterators; // iterator遍历
} dict;

typedef struct dictht {
    dictEntry **table; // 数组
    unsigned long size; // 数组的长度
    unsigned long sizemask; // 计算的一个mask
    unsigned long used; // 元素个数
} dictht;

typedef struct dictEntry {
    void *key; // key的指针
    union {
        void *val; // value指针
        uint64_t u64;
        int64_t s64;
        double d;
    } v; // value
    struct dictEntry *next; // 链表，指向下一个
} dictEntry;
```

## 添加和删除元素

### 添加

```c
int dictAdd(dict *d, void *key, void *val) {
  // dictAddRaw方法下面分析
  dictEntry *entry = dictAddRaw(d,key,NULL);

  // 如果无法添加，就返回错误
  if (!entry) return DICT_ERR;
  // 如果成功添加了，就设置一下value
  dictSetVal(d, entry, val);
  return DICT_OK;
}
```

### 删除

```c
int dictDelete(dict *ht, const void *key) {
  // 调用dictGenericDelete
  return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}

static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
  uint64_t h, idx;
  dictEntry *he, *prevHe;
  int table;

	// 如果dict没有元素，就直接返回null
  if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

  // 如果正在rehash，就帮忙rehash一个元素
  if (dictIsRehashing(d)) _dictRehashStep(d);
  // 获取hash
  h = dictHashKey(d, key);

  // 因为Dict有两个table，所以逐个遍历
  for (table = 0; table <= 1; table++) {
    // 获取table的下标
    idx = h & d->ht[table].sizemask;
    // 找到对应的entry
    he = d->ht[table].table[idx];
    prevHe = NULL;
    // 因为这是一个链表，所以如果he存在就要沿着链表一个个往后
    while(he) {
      // 如果key相同
      if (key==he->key || dictCompareKeys(d, key, he->key)) {
        // 如果链表的前一个元素存在，就前一个元素的next连到被删除元素的next
        if (prevHe)
          prevHe->next = he->next;
        else
          // 否则桶指向的链表头就指向被删除元素的next
          d->ht[table].table[idx] = he->next;
        // 是否需要释放内存，释放了内存说明只是删除，如果不释放内存就说明需要返回这块内存继续使用
        if (!nofree) {
          dictFreeKey(d, he);
          dictFreeVal(d, he);
          zfree(he);
        }
        // 因为删除一个元素，所以used减一
        d->ht[table].used--;
        return he;
      }
      // 沿着链表继续往后查
      prevHe = he;
      he = he->next;
    }
    // 如果没有rehash就停止寻找，因为第二个hashtable一定为空
    if (!dictIsRehashing(d)) break;
  }
  return NULL; /* not found */
}
```

## 渐进式rehash

如果一个Dict比较大，那么一次性的全量扩容是比较费时费力的，所以需要重新申请一个新的数组，然后逐步的把之前的元素都放到新的数组下，这是一个O(n)的操作，所以对于Redis这样的单线程模型而言，O(n)的开销是比较难以接受的。

所以Redis通过小步迁移的方式逐步完成数据的搬迁。

出发渐进式rehash有两个入口，第一个是添加数据的时候主动搬迁，还有一个是Redis的定时任务。

### 主动迁移

```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing) {
    long index;
    dictEntry *entry;
    dictht *ht;

	  // 如果当前在rehash过程中，就rehash一下
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 如果这个key对应的entry已经存在，就直接返回NULL
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // 如果在rehash的过程中，就用新的hashtable，否则就用老的
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
  	// 分配内存
    entry = zmalloc(sizeof(*entry));
  	// 把自己放到链表的头节点
    entry->next = ht->table[index];
    ht->table[index] = entry;
  	// hashtable使用数量+1
    ht->used++;

    // 把entry的key赋值
    dictSetKey(d, entry, key);
    return entry;
}
```

看一下rehash的具体实现

```c
int dictRehash(dict *d, int n) {
  	// 因为rehash要找到数组中不为空的那个桶，然后迁移，那么有可能连续好多个桶都是NULL，所以这个字段的目的就是连续访问到空桶的上限
    int empty_visits = n*10;
  	// 如果没有rehash就直接结束。
    if (!dictIsRehashing(d)) return 0;

  	// d->ht[0].used != 0就是老的hashtable还有元素
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

      	// 从rehashidx（上一次的位置，因为是渐进式，所以要记录上一次迁移到哪里）开始找
        while(d->ht[0].table[d->rehashidx] == NULL) {
          	// 如果为空，就找下一个
            d->rehashidx++;
	          // 同时这个上限-1，防止当量时间浪费在空桶的遍历上
            if (--empty_visits == 0) return 1;
        }
      	// 获取那个不为空的entry
        de = d->ht[0].table[d->rehashidx];
        // 把这个entry下的整个链表，都迁移到新的table中
        while(de) {
            uint64_t h;

            nextde = de->next;
            // 从新的hashtable获取新的下标
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
          	// 老table使用数量-1
            d->ht[0].used--;
          	// 新table使用数量+1
            d->ht[1].used++;
            de = nextde;
        }
      	// 老的置空
        d->ht[0].table[d->rehashidx] = NULL;
      	// 下标+1
        d->rehashidx++;
    }

    // 如果迁移已经结束了，
    if (d->ht[0].used == 0) {
      	// free内存
        zfree(d->ht[0].table);
      	// table交换一下
        d->ht[0] = d->ht[1];
      	// 重置
        _dictReset(&d->ht[1]);
      	// rehash下标置为-1，表示没有rehash
        d->rehashidx = -1;
      	// 返回0表示rehash已经结束了
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

可以看到每次迁移，找到有数据的下一个桶，然后把整个桶迁移过去。这就是渐进式rehash。

### 被动迁移

```c
void databasesCron(void) {
  // 省略其他代码
  /* Rehash */
  if (server.activerehashing) {
    for (j = 0; j < dbs_per_call; j++) {
      // 每一个redisdb都执行渐进式rehash
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

int incrementallyRehash(int dbid) {
  /* Keys dictionary */
  if (dictIsRehashing(server.db[dbid].dict)) {
    // 分配一定的毫秒时间，在这段时间内执行，避免一个超大的dict，然后花费太多的时间用在reash上。
    dictRehashMilliseconds(server.db[dbid].dict,1);
    return 1; /* already used our millisecond for this loop... */
  }
  /* Expires */
  if (dictIsRehashing(server.db[dbid].expires)) {
    dictRehashMilliseconds(server.db[dbid].expires,1);
    return 1; /* already used our millisecond for this loop... */
  }
  return 0;
}
```

## hash算法

redis的hash算法是siphash。具体就不分析了。

## 扩缩容

###缩容

hashtable自身的缩容是在db定时任务里处理的，同时应用redis中的几个基本数据结构，每一次删除一个数据的时候都会尝试判断是否需要缩容，这些数据结构都是基于dict的数据结构（t_hash, t_set, t_zset）

```c
void databasesCron(void) {
  // 省略。。。
  /* Resize */
  for (j = 0; j < dbs_per_call; j++) {
    tryResizeHashTables(resize_db % server.dbnum);
    resize_db++;
  }
  // 省略。。。
}

void tryResizeHashTables(int dbid) {
  // htNeedsResize就是判断是否需要缩容
  // dictResize实际缩容
  if (htNeedsResize(server.db[dbid].dict))
    dictResize(server.db[dbid].dict);
  if (htNeedsResize(server.db[dbid].expires))
    dictResize(server.db[dbid].expires);
}
```

首先是看满足什么条件需要缩容

```c
int htNeedsResize(dict *dict) {
  long long size, used;

	// 这个就是dict的两个hashtable的size相加
  size = dictSlots(dict);
  // 这个就是两个hashtable的used相加
  used = dictSize(dict);
  // 如果size > 4（言外之意就是大小不能小于4） && 使用的个数不到数组长度的10%
  return (size > DICT_HT_INITIAL_SIZE &&
          (used*100/size < HASHTABLE_MIN_FILL));
}
```

那么具体缩容的操作如下，

```c
int dictResize(dict *d) {
  int minimal;
	
  // 如果正在resize过程中，或者配置dict_can_resize不可以缩容就放弃
  if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
  // 获取当前使用综述
  minimal = d->ht[0].used;
  // 如果小于4就让这个数字 = 4
  if (minimal < DICT_HT_INITIAL_SIZE)
    minimal = DICT_HT_INITIAL_SIZE;
  // 调用方法
  return dictExpand(d, minimal);
}

int dictExpand(dict *d, unsigned long size) {
  // 检查
  if (dictIsRehashing(d) || d->ht[0].used > size)
    return DICT_ERR;

  dictht n; /* the new hash table */
  // 找到比size大且最接近的2的次方
  unsigned long realsize = _dictNextPower(size);

	// 大小校验
  if (realsize == d->ht[0].size) return DICT_ERR;

	// 赋值
  n.size = realsize;
  n.sizemask = realsize-1;
  // 分配内存
  n.table = zcalloc(realsize*sizeof(dictEntry*));
  n.used = 0;

  // 判断是不是初始化的dict，算是一个防御性代码
  if (d->ht[0].table == NULL) {
    d->ht[0] = n;
    return DICT_OK;
  }

  // 把新建的hashtable放到数组中
  d->ht[1] = n;
  // rehashidx不为-1就是可以rehash，因为扩容缩容的目的就是要把老的hashtable干掉，所以一定要开始迁移，否则只是新建了空间没用
  d->rehashidx = 0;
  return DICT_OK;
}
```

### 扩容

扩容源码中dictExpand的实现和上面缩容是相同的，就是找到比期望大小大的2的n次方，然后分配一个新的hashtable。

```c
static int _dictExpandIfNeeded(dict *d) {
  // 如果在rehash的过程中，直接返回
  if (dictIsRehashing(d)) return DICT_OK;

  // 如果是初始化，就用初始化的大小4
  if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

  // 如果hashtable的元素数量已经 >= hashtable的size 同时（配置允许dict_can_resize || 使用的数量 / 数组的大小大于强制扩容的比例（5）
  // 也就是元素数量是数组长度的五倍时，强制扩容
  if (d->ht[0].used >= d->ht[0].size &&
      (dict_can_resize ||
       d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
  {
    return dictExpand(d, d->ht[0].used*2);
  }
  return DICT_OK;
}
```

什么情况下会尝试扩容？

这个方法的调用链路如下：**dictAdd -> _dictKeyIndex -> _dictExpandIfNeeded**，其实也就是每次给dict添加新内容的时候，都会去判断是否需要扩容，如果需要扩容就分配当前size的两倍的新table。