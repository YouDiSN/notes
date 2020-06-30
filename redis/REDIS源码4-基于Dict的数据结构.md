#REDIS源码4-基于Dict的数据结构

前面已经分析了Dict的基本原理，Dict本身在Redis内部的应用非常广泛，但是对于我们而言，我们通常使用的Redis数据结构无外乎string，list，zset，set和hash（还有其他的）。其中set和hash（hash比较特殊，不能一概而论）都是基于Dict实现的数据结构。

由于hash这个数据结构可能还会用ziplist来实现，所以可以先分析Redis是如何使用Dict来实现Set的。

##Set的实现原理

Redis的Set和Java的HashSet基本差不多。Java的HashSet是基于HashMap，然后value永远为null。Redis里面的实现基本也是类似的，没有太大的区别。

往Set中添加一个元素：

```c
// OBJ_ENCODING_HT是Set的一个Encoding类型
if (subject->encoding == OBJ_ENCODING_HT) {
  // 获取dict
  dict *ht = subject->ptr;
  // 找到dict中的dictEntry，如果已经存在，de就是NULL，否则de就是新分配到的一块内存
  // 注意dictAddRaw可能会引起dict的扩容
  dictEntry *de = dictAddRaw(ht,value,NULL);
  // 如果能够新分配一块de，说明之前不存在对应的值，所以新分配到的entry就设置一下
  if (de) {
    // key是用户传入的参数，类似HashSet；注意这里需要把用户传入的参数value拷贝一份放到dict中
    // 拷贝的原因应该是传入的这个value应该会被回收掉以便后续的请求再次利用，所以这里要拷贝一份。
    dictSetKey(ht,de,sdsdup(value));
    // value时null
    dictSetVal(ht,de,NULL);
    return 1;
  }
}
```

往Set中移除一个元素：

```c
if (setobj->encoding == OBJ_ENCODING_HT) {
  // 调用Dict的删除
  if (dictDelete(setobj->ptr,value) == DICT_OK) {
    // 判断是否需要缩容
    if (htNeedsResize(setobj->ptr)) dictResize(setobj->ptr);
    return 1;
  }
}
```

##Hash的实现原理

在分析Hash结构之前，先要认识到对于Hash而言，有两种数据结构，一种是前面分析的Dict，也就是类似Java的HashMap，还有一个就是ziplist结构。

### ziplist

ziplist是一个非常重要的结构，因为除了Hash，ZSET结构也用到了ziplist（在元素比较少的情况下）。ziplist其实类似embstr和sds这样的关系，是把通过指针寻找的一块分散的内存，进行整合，放到一块连续的内存空间，这样做可以提高程序的局部性。

所以要想完整的了解Hash的实现原理，就要先分析ziplist的基本原理。
$$
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
$$

- zlbytes，4个字节无符号整数，代表整个ziplist的长度
- zltail，4个字节无符号整数，代表到最后一个entry头的偏移，主要目的是为了从后往前遍历，所以要定位到最后一个entry的头部
- zllen，2个字节无符号整数，entry的个数
- zlend，1个字节无符号整数，代表ziplist的末尾，固定值永远都是255 = 0xFF
- 所有存储的字节序都是小端存储

ziplist的基本结构清楚了，那么看一下zlentry的基本结构：

```c
typedef struct zlentry {
  unsigned int prevrawlensize; // prevrawlen这个字段的长度，这个字段不会实际在内存中存储，都是要用的时候根据已有内容推算出来的
  unsigned int prevrawlen;     // 前一个entry的长度
  unsigned int lensize;        // len这个字段的长度，这个字段也不会实际在内存中存储，都是要用的时候根据已有内容推算出来的
  unsigned int len;            // 长度
  unsigned int headersize;     // prevrawlensize + lensize，这个字段也不会实际在内存中存储，都是要用的时候根据已有内容推算出来的
  unsigned char encoding;      // 编码
  unsigned char *p;            // entry内容的指针，这个指针往往是构建zlentry的时候才用到，实际存储的时候由于是一大块内存，所以其实没用
} zlentry;
```

其中encoding字段的目的是为了知道后面的content内容的编码格式，不同的encoding取值对应的含义如下：

```
|00pppppp| - 1 byte
     String value with length less than or equal to 63 bytes (6 bits).
     "pppppp" represents the unsigned 6 bit length.
|01pppppp|qqqqqqqq| - 2 bytes
     String value with length less than or equal to 16383 bytes (14 bits).
     IMPORTANT: The 14 bit number is stored in big endian.
|10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
     String value with length greater than or equal to 16384 bytes.
     Only the 4 bytes following the first byte represents the length
     up to 32^2-1. The 6 lower bits of the first byte are not used and
     are set to zero.
     IMPORTANT: The 32 bit number is stored in big endian.
|11000000| - 3 bytes
     Integer encoded as int16_t (2 bytes).
|11010000| - 5 bytes
     Integer encoded as int32_t (4 bytes).
|11100000| - 9 bytes
     Integer encoded as int64_t (8 bytes).
|11110000| - 4 bytes
     Integer encoded as 24 bit signed (3 bytes).
|11111110| - 2 bytes
     Integer encoded as 8 bit signed (1 byte).
|1111xxxx| - (with xxxx between 0000 and 1101) immediate 4 bit integer.
     Unsigned integer from 0 to 12. The encoded value is actually from
     1 to 13 because 0000 and 1111 can not be used, so 1 should be
     subtracted from the encoded 4 bit value to obtain the right value.
|11111111| - End of ziplist special entry.
```

其实有点类似UTF-8的编码，每个字节的开头几个字符代表了当前这个字母占用几个字节，这里其实也是一样的。

### 读取操作元素

ziplist其实和embstr类似，都是紧凑存储，所以每插入或删除一个元素都需要扩展内存，如果存储很大的内容或者存储的元素相当多，那么不停的重新分配内存或拷贝内存就会有很大的消耗。

下面从源码级别分析一下在ziplist中读取元素的原理，其实非常简单，获取指针之后，看是从头便利还是从尾遍历，然后根据len计算当前entry的长度，把指针移动到计算后的size，就是获取下一个entry了。

####ziplist根据下标查找：

```c
unsigned char *ziplistIndex(unsigned char *zl, int index) {
  unsigned char *p;
  unsigned int prevlensize, prevlen = 0;
  // 这个就是看从尾遍历还是从头便利，小于0就是从尾开始
  if (index < 0) {
    // 倒数的变成正数
    index = (-index)-1;
    // 找到tail entry的指针
    p = ZIPLIST_ENTRY_TAIL(zl);
    // 不是end就说明ziplistyou 元素
    if (p[0] != ZIP_END) {
      // 获取prevlen
      ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
      // 然后因为要获取倒数第index个，所以从最后一个开始一个个往前数
      while (prevlen > 0 && index--) {
        p -= prevlen;
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
      }
    }
  } else {
    // 这个就是正数
    p = ZIPLIST_ENTRY_HEAD(zl);
    // 从头开始一个个往后数
    while (p[0] != ZIP_END && index--) {
      // 这个就是当前entry的length + 下一个entry的prevlen，这样就空开不要的数据，直接获取下一个entry的头
      p += zipRawEntryLength(p);
    }
  }
  // 然后看是不是找到了，没找到返回NULL，找到了就返回对应的指针
  return (p[0] == ZIP_END || index > 0) ? NULL : p;
}
```

####ziplist根据值查找：

```c
unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip) {
  int skipcnt = 0;
  unsigned char vencoding = 0;
  long long vll = 0;

  // 只要没有遍历到头
  while (p[0] != ZIP_END) {
    unsigned int prevlensize, encoding, lensize, len;
    unsigned char *q;

    ZIP_DECODE_PREVLENSIZE(p, prevlensize);
    ZIP_DECODE_LENGTH(p + prevlensize, encoding, lensize, len);
    // 这个q就是content的指针位置
    // 因为p是头，加上prevlensize和lensize，就到了content的内容
    q = p + prevlensize + lensize;

    // 是否需要跳过
    if (skipcnt == 0) {
      // 如果编码是string
      if (ZIP_IS_STR(encoding)) {
        // 看长度是否相同，以及对应的内存是否相同
        if (len == vlen && memcmp(q, vstr, vlen) == 0) {
          // 如果相同直接返回
          return p;
        }
      } else {
        // 获取编码
        if (vencoding == 0) {
          if (!zipTryEncoding(vstr, vlen, &vll, &vencoding)) {
            /* If the entry can't be encoded we set it to
                         * UCHAR_MAX so that we don't retry again the next
                         * time. */
            vencoding = UCHAR_MAX;
          }
          /* Must be non-zero by now */
          assert(vencoding);
        }

        // 继续比较
        if (vencoding != UCHAR_MAX) {
          long long ll = zipLoadInteger(q, encoding);
          if (ll == vll) {
            // 一致就返回
            return p;
          }
        }
      }

      /* Reset skip count */
      skipcnt = skip;
    } else {
      /* Skip entry */
      skipcnt--;
    }

    /* Move to next entry */
    p = q + len;
  }

  return NULL;
}
```

这里的skip的作用一开始有点没看懂，因为作为一个ziplist查找元素难道不是应该一个个查找，为什么需要skip，比如在Hash结构当中，肯定有key和value，那么我要比对的肯定key，所以value都不用比对，直接跳过就可以了，所以在hash部分的源码中，调用ziplistFind方法的时候skip的传参都是1，就是因为key和value都是成对出现的这个原因。

####ziplist添加或删除元素：

```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
  // 小端获取当前的长度
  size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
  unsigned int prevlensize, prevlen = 0;
  size_t offset;
  int nextdiff = 0;
  unsigned char encoding = 0;
  long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
  zlentry tail;

  // 这里就是看p这个指针是否已经指向了末尾，如果已经在末尾了，就直接获取prevlen
  // 如果p没有指向末尾，就获取最后一个元素的prevlen
  if (p[0] != ZIP_END) {
    ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
  } else {
    unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
    if (ptail[0] != ZIP_END) {
      prevlen = zipRawEntryLength(ptail);
    }
  }

  // 这段reqlen的计算主要是因为要新插入一个entry，所以要计算这个新插入的entry需要多少的内存空间。

  // zipTryEncoding这个方法主要是看string类型的字段是否可以用注入int8、int16等来存储，这样可以节约内存
  if (zipTryEncoding(s,slen,&value,&encoding)) {
    // 如果可以话，entry content的size就是用这个格式存储之后的size，比如int8就是8
    reqlen = zipIntSize(encoding);
  } else {
    // 否则就直接用插入的数据长度slen来做content的长度
    reqlen = slen;
  }
  // 计算prevlen需要多少空间
  reqlen += zipStorePrevEntryLength(NULL,prevlen);
  // 计算encoding需要多少空间
  reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

  /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
  int forcelarge = 0;
  // 这个就是看因为我要插入一个元素，所以我要看该元素后面的元素的prevlen是不是够存储新插入的这个元素，否则后面那个元素也要扩容
  // 如果p已经是ziplist末尾，那就是0，否则就要计算插入的这个位置的元素的prevlen是多少，因为要用来存储新插入这个元素的长度
  nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
  // 这个就是后一个entry的prevlen太大了，分配了5个字节，但是插入的很小，所以多了4个字节，同时我新插入的这个元素家好又小于4个字节，意思就是我不用移动，直接把后面多出来的空间给这个新插入的元素用就可以了
  if (nextdiff == -4 && reqlen < 4) {
    nextdiff = 0;
    forcelarge = 1;
  }

  // 先计算当前指针p的偏移
  offset = p-zl;
  // 看一下是不是要扩容
  zl = ziplistResize(zl,curlen+reqlen+nextdiff);
  // 然后根据便宜重新从zl的指针位置算出p的位置
  p = zl+offset;

  // 如果不是最后
  if (p[0] != ZIP_END) {
    // 把这些数据往后挪一下
    memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

    // 更新下一个entry的prevlen
    if (forcelarge)
      zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
    else
      zipStorePrevEntryLength(p+reqlen,reqlen);

    // 更新尾指针
    ZIPLIST_TAIL_OFFSET(zl) =
      intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

    // 这个就是插入的reqlen没有包含nextdiff，所以要重新计算一遍
    zipEntry(p+reqlen, &tail);
    if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
      ZIPLIST_TAIL_OFFSET(zl) =
        intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
    }
  } else {
    /* This element will be the new tail. */
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
  }

  // 如果后面的数据prevlen变动过了，那么也就是有多余的这些字节需要计算，所以需要从插入位置的后一个开始一个个更新prevlen，这个就是级联操作
  if (nextdiff != 0) {
    // 这里的offset作用和上面一样，也是更新过ziplist之后保留当前entry的指针
    offset = p-zl;
    zl = __ziplistCascadeUpdate(zl,p+reqlen);
    p = zl+offset;
  }

  // 写prelen的数据
  p += zipStorePrevEntryLength(p,prevlen);
  // 写encoding
  p += zipStoreEntryEncoding(p,encoding,slen);
  // 写content的数据
  if (ZIP_IS_STR(encoding)) {
    memcpy(p,s,slen);
  } else {
    zipSaveInteger(p,value,encoding);
  }
  // ziplist的len+1
  ZIPLIST_INCR_LENGTH(zl,1);
  // 数据插入完毕
  return zl;
}
```

#### 删除一个元素：

```c
unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
  unsigned int i, totlen, deleted = 0;
  size_t offset;
  int nextdiff = 0;
  zlentry first, tail;

  // 这个方法就是把p指针所指着的这块内存构建成一个有完整信息的zlentry，就是first
  // 其实就是被删除的第一个元素
  zipEntry(p, &first);
  for (i = 0; p[0] != ZIP_END && i < num; i++) {
    // 这个就是因为要删除num个，所以我就不停的往后看，要删到哪个元素位置
    p += zipRawEntryLength(p);
    deleted++;
  }

  // first.p就是最最前面的指针，这个p就是最后面的位置，totlen就是总共要删除多少字节的数据
  totlen = p-first.p;
  // 如果要删除的内容大于0
  if (totlen > 0) {
    // 如果p没有指向最后的位置
    if (p[0] != ZIP_END) {
      // 这个就是计算我第一个删除元素记录的prevlen，和p指向的最后一个没有删的元素的prevlen是否使用的内存一致，因为中间的元素被删除之后，最后面保留的那个prevlen就是first的prevlen
      nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);

      // p的位置需要移动，因为prevlen可能多增加或减少
      p -= nextdiff;
      // 存储相关信息
      zipStorePrevEntryLength(p,first.prevrawlen);

      // 更新指向tail的指针
      ZIPLIST_TAIL_OFFSET(zl) =
        intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen);

      // 因为上面计算tail的时候用到了totlen，这个不包含nextdiff，所以如果后面还有数据，tail还要再加一遍totlen
      zipEntry(p, &tail);
      if (p[tail.headersize+tail.len] != ZIP_END) {
        ZIPLIST_TAIL_OFFSET(zl) =
          intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
      }

      // 把后面的数据拷过来
      memmove(first.p,p,
              intrev32ifbe(ZIPLIST_BYTES(zl))-(p-zl)-1);
    } else {
      /* The entire tail was deleted. No need to move memory. */
      ZIPLIST_TAIL_OFFSET(zl) =
        intrev32ifbe((first.p-zl)-first.prevrawlen);
    }

    // 更新ziplist的长度和相关数据
    offset = first.p-zl;
    zl = ziplistResize(zl, intrev32ifbe(ZIPLIST_BYTES(zl))-totlen+nextdiff);
    ZIPLIST_INCR_LENGTH(zl,-deleted);
    p = zl+offset;

    // 这个就是如果删除的时候有多余的nextdiff，后面的数据就要级联更新
    if (nextdiff != 0)
      zl = __ziplistCascadeUpdate(zl,p);
  }
  return zl;
}
```

可以看到在插入和删除的时候都设计一个级联更新方法__ziplistCascadeUpdate，

####级联更新：

```c
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
  size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), rawlen, rawlensize;
  size_t offset, noffset, extra;
  unsigned char *np;
  zlentry cur, next;

	// 只要不是尾巴
  while (p[0] != ZIP_END) {
    // 获取当前entry的详细信息
    zipEntry(p, &cur);
    // 计算一下各种size
    rawlen = cur.headersize + cur.len;
    rawlensize = zipStorePrevEntryLength(NULL,rawlen);

    // 如果后面没东西就停下来
    if (p[rawlen] == ZIP_END) break;
    // 获取下一个的数据
    zipEntry(p+rawlen, &next);

    // 如果长度没有变化，就不再继续级联更新
    if (next.prevrawlen == rawlen) break;

    // 如果原本的prevlen的大小小于现在这个节点的大小
    if (next.prevrawlensize < rawlensize) {
      // 保留指针的偏移
      offset = p-zl;
      // 看需要额外多少内存
      extra = rawlensize-next.prevrawlensize;
      // resize一下
      zl = ziplistResize(zl,curlen+extra);
      // 重新找到p的位置
      p = zl+offset;

      // 更新指针
      np = p+rawlen;
      noffset = np-zl;

      // 更新尾指针，因为数据已经移动过了
      if ((zl+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))) != np) {
        ZIPLIST_TAIL_OFFSET(zl) =
          intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
      }

      // 移动内存
      memmove(np+rawlensize,
              np+next.prevrawlensize,
              curlen-noffset-next.prevrawlensize-1);
      zipStorePrevEntryLength(np,rawlen);

      // 更新p指针，移动到下一个节点
      p += rawlen;
      curlen += extra;
    } else {
      // 如果后面节点的prevlen的大小比当前这个节点要大
      if (next.prevrawlensize > rawlensize) {
        // 这个就是用5个字节存储一个字节的数据，因为这里完美的话是需要收缩的，因为后一个entry的prelen有5个字节，但是当前节点1一个字节的长度就足够了，但是因为级联更新消耗太大，所以这里就浪费4个字节，用空间换取时间
        zipStorePrevEntryLengthLarge(p+rawlen,rawlen);
      } else {
        // 这个是只更新一下prevlen的值就好，不用继续了，因为内存空间没有变化
        zipStorePrevEntryLength(p+rawlen,rawlen);
      }

      /* Stop here, as the raw length of "next" has not changed. */
      break;
    }
  }
  return zl;
}
```

##Dict

上面一大坨都分析了ziplist情况下的Hash结构，其实用Dict就和HashMap这样的结构基本差不多了，基本上就是通过**dictFind**方法从dict中找到对应的元素。

但是有一个ziplist和dict的转换问题需要关注。

hash结构一上来都会优先使用ziplist（看到的代码是这样的，但是不是很确定是不是所有场景都是这样）。然后再不断的操作过程中如果满足这两个条件就回吧ziplist变成一个dict：

1. 是string类型同时操作的字符串长度超过64个字节（其实就是redis认为超过64个字节的就是大字符串了）
2. ziplist的entry数量超过512个

转换的过程就是创建一个新的dict，遍历ziplist，然后把key和value都插入到dict里，完成后把原来的ziplist内存释放掉即可。