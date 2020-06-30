#REDIS源码6-Intset

Redis的Set和Java的HashSet实现原理基本类似，前面分析Dict的时候已经分析过Redis Set的一个形态。但是Redis Set还有另外一种形态，就是Intset。

## Intset

Intset存在的意义和emdstr存在的意义非常类似，就是在某些特殊场景下通过一些特殊的实现，以实现内存的最大优化。那么intset优化的场景就是一个Set中，只有整数，并且元素数量不多的情况下，通过intset这样一个紧凑的数据结构来节省内存。

这里要注意intset里面的存储的内容都是排序的，后面的插入、删除查找都是基于有序来实现的

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

结构非常简单，4个字节编码，4个字节长度，后面就是数据，非常好理解。

> 不过考虑到intset是存储长度不长的数字，所以encoding和length是否需要共8个字节去存储？可能是intset本身不会被大量创建，所以一个intset浪费几个字节并不是很致命的问题，不想SDS这样的字符串节省一个字节也很有意义。又或者是因为考虑到高速缓存行（？）

## 插入和删除

###插入

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
  // 这个就是看是16、32还是64位的int
  uint8_t valenc = _intsetValueEncoding(value);
  uint32_t pos;
  if (success) *success = 1;

  // 看是不是需要提升encoding的级别，比如之前都是16位int，现在要变成32位int
  if (valenc > intrev32ifbe(is->encoding)) {
    /* This always succeeds, so we don't need to curry *success. */
    return intsetUpgradeAndAdd(is,value);
  } else {
    // 这个就是如果不需要提升编码，就从set里面找一下这个数字是不是已经存在了
    // 如果原来的数据不存在，pos会在查找的过程中被更新为应该插入的位置，因为是有序的所以不可以随便插
    if (intsetSearch(is,value,&pos)) {
      // 如果存在了就不成功，因为根本就没插进去
      if (success) *success = 0;
      return is;
    }

    // 如果发现可以插入，就先调整一下空间，因为要空出来一块内存给新的数据
    // 其实就是根据新的length，重新分配一块内存
    is = intsetResize(is,intrev32ifbe(is->length)+1);
    // 调整插入的位置，就是确保pos指向intset的尾部
    // 注意这里要根据存储数据的实际大小来计算尾部，比如4个字节还是8个字节等
    if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
  }

  // 插入数据，就是把这个值写到这块内存上即可
  _intsetSet(is,pos,value);
  // 调整length的大小
  is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
  return is;
}
```

还有一个需要关注的就是**intsetUpgradeAndAdd**方法，这是涉及到编码的提升。

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
  // 当前编码
  uint8_t curenc = intrev32ifbe(is->encoding);
  // 新的编码
  uint8_t newenc = _intsetValueEncoding(value);
  // 当前长度
  int length = intrev32ifbe(is->length);
  // 根据value是不是小于零，来确定这个值放在最前面还是最后面。这个有什么用吗？
  int prepend = value < 0 ? 1 : 0;

  // 重新设置编码
  is->encoding = intrev32ifbe(newenc);
  // 调整intset的大小
  is = intsetResize(is,intrev32ifbe(is->length)+1);

  // 这个要从尾巴开始，这样就不会覆盖以前的值
  // 比如原来是1个字节00000001|00000010|00000011这样的存储，现在要变成2个字节，先扩容变成
  // 00000001|00000010|00000011|00000000|00000000|00000000|00000000|00000000
  // 这样如果从最后一个00000011放到最后，前面的数据就不会被覆盖
  // 然后把数据一个个拷贝过去，最后变成如下，最后两个字节留给新插入的数据（最前还是最后是根据prepend来的）
  // 00000000,00000001|00000000,00000010|00000000,000000011|00000000,00000000

  // 从最后一个往前
  while(length--)
    // 这个prepend如果是1，那就是插入的在最前，如果是0，就是插入的在最后
    _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

  // 这里说明一下为什么是头和尾，因为这个数据不可能在别的地方。
  // 能进到这个方法说明新插入的数据所需要的空间比原来要大，因为每一个插入的数据redis都会尝试用最合适的字节数来存储，也就是说比如1，只会用最小的2个字节来存储，而不会用8个字节。
  // 因此既然能做到需要调整encoding，言外之意就是这个数字用现在的几个字节是不够了，那么如果小于零一定是最小的，大于零一定是最大的，否则就可以用之前的编码来存储而不需要扩容了。
  // 因此这里只要判断prepend就可以这个数据是最大还是最小了
  if (prepend)
    _intsetSet(is,0,value);
  else
    _intsetSet(is,intrev32ifbe(is->length),value);
  // 更新length
  is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
  return is;
}
```

其实插入也是很好理解的，就是看一看int的大小是不是一致，如果大小一样，就挪一个空间出来插入；如果大小不一样，就根据新的int大小重新调整，然后把之前的数据从最后一个开始用新的int size分配一下，然后再把要插入的数据插入即可。

### 删除

删除比插入更简单，就是找到被删数据，然后把后面的数据往前挪一位，最后多出来的部分就free即可

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
  // 删除数据编码
  uint8_t valenc = _intsetValueEncoding(value);
  uint32_t pos;
  if (success) *success = 0;

  // 被删数据的编码 小于等于 intset编码，就找到那个数据所在的位置
  if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
    // intset的长度
    uint32_t len = intrev32ifbe(is->length);

    /* We know we can delete */
    if (success) *success = 1;

    // 把最后一个数据拷过来
    if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
    // 把最后的几个字节free掉
    is = intsetResize(is,len-1);
    // 长度-1
    is->length = intrev32ifbe(len-1);
  }
  return is;
}
```

其中的**intsetSearch**方法需要看一下具体的实现，

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
  int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
  int64_t cur = -1;

  // 为空直接返回
  if (intrev32ifbe(is->length) == 0) {
    if (pos) *pos = 0;
    return 0;
  } else {
    // intset是排过序的，所以如果比最小的小或者比最大的大，就肯定不存在了
    if (value > _intsetGet(is,max)) {
      if (pos) *pos = intrev32ifbe(is->length);
      return 0;
    } else if (value < _intsetGet(is,0)) {
      if (pos) *pos = 0;
      return 0;
    }
  }

  // 二分法查找
  while(max >= min) {
    mid = ((unsigned int)min + (unsigned int)max) >> 1;
    cur = _intsetGet(is,mid);
    if (value > cur) {
      min = mid+1;
    } else if (value < cur) {
      max = mid-1;
    } else {
      break;
    }
  }

  // 如果找到了就返回pos，否则就没有
  if (value == cur) {
    if (pos) *pos = mid;
    return 1;
  } else {
    if (pos) *pos = min;
    return 0;
  }
}
```

这个查找是一个非常简单好理解的二分法查找。

## 使用场景

前面分析了一下intset的基本操作，和其他的连续分配的大块内存的做法其实大致类似，比如ziplist，但是也要关注什么场景下才会使用intset。

```c
 if (intsetLen(subject->ptr) > server.set_max_intset_entries)
   setTypeConvert(subject,OBJ_ENCODING_HT);
```

每次插入的时候都会根据这个条件判断，**set_max_intset_entries = 512**，也就是如果intset长度超过512，就会变成一个HashTable的实现。

还有一个就是如果放到set里面的内容不再是数组，就会直接变成HashTable，无视长度，因为intset本来就是放数字的。