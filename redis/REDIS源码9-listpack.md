#REDIS源码9-listpack

listpack的诞生目的应该就是为了替换Redis种被广泛使用的ziplist。它的结构和ziplist相比非常接近，但是更精简。

> [listpack作者介绍](https://github.com/antirez/listpack/blob/master/listpack.md)
>
> 作者的大致意思是：
>
> 在Redis的开发早期，花了很多时间精力来设计数据结构。因为如果大量的结构都使用链表的话，那么就会包含大量的指针（64位指针需要8个字节），那么在存储的数据量比较小的情况下，去维护数据结构本身所需要的metadata所占用的空间很容易就超过了50%。同时一些遍历等需求，如果使用一整块内存的array形式，在代码的局部性上也要更优，所以Redis的做法基本都是在数据量比较小的情况下，使用压缩的数据结构，如果数据量比较大，才会转为链式的结构。
>
> 所以这也是作者为什么在Redis中大量使用了ziplist的原因。
>
> 然后作者曾经花了数周，和多位开发者一起定位一个ziplist引起的bug，但是他们花了很长时间都没有办法定位到那个bug。后来大家也都同意，ziplist的相关处理实在太复杂了。
>
> ziplist的基本数据结构如下：
>
> ```
> <header> <entry> <entry> ... <entry> <end-of-ziplist>
> ```
>
> 但是ziplist为了能支持从后往前查询，每一个entry又被定义成如下结构：
>
> ```
> <previous-entry-length> <entry-data>
> ```
>
> 其中previous-entry-length这个长度1-5个字节不等，同时如果长度变更，还可能引发级联更新。
>
> 所以为了能进一步提升这块代码的实现，所以作者设计了listpack数据结构以替换ziplist。
>
> 不过目前Redis5.0的源码中，只有Stream采用了listpack，因为其他地方使用ziplist非常非常多，一下子全部替换可能有很大的风险，所以作者暂时只在最新的Stream结构中使用了该结构。

##listpack基本设计

listpack和ziplist的基本基本设计比较类似，其目的都是利用一大块的内存，来提高内存使用的紧凑型、代码的局部性以及内存使用率的优化。

```c
<tot-bytes> <num-elements> <element-1> ... <element-N> <listpack-end-byte>
```

header部分占用6个字节

**tot-bytes：**32个bit，包含header在内的整个listpack共占用了多少内存。

**num-elements：**16个bit，代表当前listpack的element共有多少个，因为是16个bit，所以最大是65536，所以如果这个值等于65536则表示，长度有可能是超过65536。也就是说如果不是65536，就是精确的长度，如果等于65536表示长度可能是65536也有可能大于65536。不过这种情况要想知道确切的长度，需要便利整个listpack。

上面是listpack的设计，下面看一下element的设计：

```c
<encoding-type><element-data><element-tot-len>
```

encoding-type和total-len永远都存在，不过data部分可能因为就是空而不存在。

**encoding-type：**然后根据type的不同取值，来知道后面的结构是如何存储的，具体可以看作者的文档，不过大致上就是比如第一个bit为1，那么就是7位的小数字；如果是10就是tiny string；如果是110或者1110就是表示encoding字段本身就需要多个字节来记录。

**total：**这个字段存在的意义就是为了从右像左遍历的。由于一个element的长度本身可长可短，所以这个total字段的长度也是不好固定的。那么作者是这样设计的，就是一个字节是8个bit，作者用7个bit来记录数据，第一个bit如果是0，就表示左边的内容不是total，如果是1，就表示左边的那个字节还是total。下面举个简单的例子：

比如500，在二进制表示位：111110100

然后如果存储为total是这样存储的：0000011 1110100。从右向左遍历的时候，取到1110100，然后后7个bit是数据，第一个1表示左侧的那个字节依然是total。然后取左侧的那个字节0000011，发现第一个bit是0，表示total的内容到此为止，再把两个字节的后7个bit拼起来就是真正的total字节数。

## 源码分析

源码分析还是从基本的插入、删除、获取等基本的操作开始。

### 插入和删除

```c
// 这个方法包含了插入和删除，具体是体现在不同的参数上的。
// 一个ele可以在p这个指针指向的位置进行操作，操作的方式根据where的不同取值可以是p指针前、p指针后以及替换。
// 如果ele是null，那就是删除p指针指向的位置。
// newp相当于就是返回回去的新的指针位置
unsigned char *lpInsert(unsigned char *lp, unsigned char *ele, uint32_t size, unsigned char *p, int where, unsigned char **newp) {
  unsigned char intenc[LP_MAX_INT_ENCODING_LEN];
  unsigned char backlen[LP_MAX_BACKLEN_SIZE];

  uint64_t enclen; /* The length of the encoded element. */

  // 如果ele是NULL，那就是删除，删除就是删除p指针所在的位置，所以把模式变成REPLACE
  if (ele == NULL) where = LP_REPLACE;

  // 如果是AFTER，就变成BEFORE，p的后面，就是p后面的那个节点的前面。。。所以就玩一个小技巧
  if (where == LP_AFTER) {
    p = lpSkip(p);
    where = LP_BEFORE;
  }

  // 记录偏移量，因为不论插入删除，都是要对内存重新分配的，所以到时候还可以根据偏移重新找到位置
  unsigned long poff = p-lp;

  // encoded-type
  int enctype;
  // 如果有元素，就获取元素的type
  if (ele) {
    enctype = lpEncodeGetType(ele,size,intenc,&enclen);
  } else {
    enctype = -1;
    enclen = 0;
  }

  // 前一个节点的长度占用了几个字节
  unsigned long backlen_size = ele ? lpEncodeBacklen(backlen,enclen) : 0;
  // 在修改前，listpack占用的整体字节数
  uint64_t old_listpack_bytes = lpGetTotalBytes(lp);
  uint32_t replaced_len  = 0;
  if (where == LP_REPLACE) {
    // 如果是替换，就计算出当前p指针指向的节点占用了多少空间
    replaced_len = lpCurrentEncodedSize(p);
    // 然后加上back-length的长度。这个地方的意思是就是一个entry有header、数据和这个entry的总长度，因为这个entry的总长度是不定长的，而且没有记录占用了多少字节，所以是需要计算的。
    replaced_len += lpEncodeBacklen(NULL,replaced_len);
  }

  // 计算新的listpack的长度
  uint64_t new_listpack_bytes = old_listpack_bytes + enclen + backlen_size
    - replaced_len;
  if (new_listpack_bytes > UINT32_MAX) return NULL;

  unsigned char *dst = lp + poff; /* May be updated after reallocation. */

  // 如果需要更多的空间，就先分配出足够多的内存
  if (new_listpack_bytes > old_listpack_bytes) {
    if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
    dst = lp + poff;
  }

  // 这个就是操作内存，如果是插入，就把后面的往后，如果是删除就把后面的往前
  if (where == LP_BEFORE) {
    // 如果是before类型
    memmove(dst+enclen+backlen_size,dst,old_listpack_bytes-poff);
  } else {
    // 如果是replace类型
    long lendiff = (enclen+backlen_size)-replaced_len;
    memmove(dst+replaced_len+lendiff,
            dst+replaced_len,
            old_listpack_bytes-poff-replaced_len);
  }

  // 如果新的空间占用的更少，就把多出来的部分free
  if (new_listpack_bytes < old_listpack_bytes) {
    if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
    dst = lp + poff;
  }

  // 计算newp的位置
  if (newp) {
    *newp = dst;
    /* In case of deletion, set 'newp' to NULL if the next element is
         * the EOF element. */
    if (!ele && dst[0] == LP_EOF) *newp = NULL;
  }
  // 有ele就说明是插入或者replace，就把ele的数据拷贝到对应的位置
  if (ele) {
    if (enctype == LP_ENCODING_INT) {
      memcpy(dst,intenc,enclen);
    } else {
      lpEncodeString(dst,ele,size);
    }
    dst += enclen;
    // 然后是backlen
    memcpy(dst,backlen,backlen_size);
    dst += backlen_size;
  }

  /* Update header. */
  if (where != LP_REPLACE || ele == NULL) {
    uint32_t num_elements = lpGetNumElements(lp);
    if (num_elements != LP_HDR_NUMELE_UNKNOWN) {
      // 插入或删除元素，总数就更新一下
      if (ele)
        lpSetNumElements(lp,num_elements+1);
      else
        lpSetNumElements(lp,num_elements-1);
    }
  }
  // total bytes也更新一下
  lpSetTotalBytes(lp,new_listpack_bytes);

  return lp;
}
```

从上面的源码分析可以看出来，虽然代码流程比较长，但是总体来说还是比较好理解的，基本的做法就是找到要处理的位置，然后分配空间，然后进行处理，然后把元素释放掉或者把内容拷贝到这里来。最后再整体更新一下total elements和total size这些元数据内容。

###遍历查找

上面看了插入和删除的处理，还有一个用的非常多的就是我要从listpack里面找到一个元素，那么就分析一下这个查找的处理。

```c
unsigned char *lpSkip(unsigned char *p) {
  unsigned long entrylen = lpCurrentEncodedSize(p);
  entrylen += lpEncodeBacklen(NULL,entrylen);
  p += entrylen;
  return p;
}
```

查找的方法很简单，当我有了p指针指向一个entry的开头位置的时候，先找到当前entry的大小，然后计算出backlen的大小，然后更新p指针即可。

这里要注意的是entry的大小就是通过encoded-type存储的，而backlen则是根据计算出来的entrylen来计算的，下面简单贴一下，不必太在意，其实就是给定一个长度看看需要多少字节来存储。

```c
unsigned long lpEncodeBacklen(unsigned char *buf, uint64_t l) {
  // 因为是7个bit存储数据，所以是从127而不是256开始，后面做法类似
  if (l <= 127) {
    if (buf) buf[0] = l;
    return 1;
  } else if (l < 16383) {
    if (buf) {
      buf[0] = l>>7;
      buf[1] = (l&127)|128;
    }
    return 2;
  } else if (l < 2097151) {
    if (buf) {
      buf[0] = l>>14;
      buf[1] = ((l>>7)&127)|128;
      buf[2] = (l&127)|128;
    }
    return 3;
  } else if (l < 268435455) {
    if (buf) {
      buf[0] = l>>21;
      buf[1] = ((l>>14)&127)|128;
      buf[2] = ((l>>7)&127)|128;
      buf[3] = (l&127)|128;
    }
    return 4;
  } else {
    if (buf) {
      buf[0] = l>>28;
      buf[1] = ((l>>21)&127)|128;
      buf[2] = ((l>>14)&127)|128;
      buf[3] = ((l>>7)&127)|128;
      buf[4] = (l&127)|128;
    }
    return 5;
  }
}
```

