#REDIS源码12-HyperLogLog

Redis面试的过程中经常会问，如果想要统计每个用户的uv数量如何实现。一种方式就是通过HashMap，但是如果元素数量不多的话可以用HashMap来实现。如果用户量轻松破亿，要维护一个过亿数量的HashMap本身就不容易，所以需要一个更好的方式来高效的实现这个需求。

于是乎就有了HyperLogLog。

首先这是一个算法，并不是Redis独创的算法，在Java中也有对应的实现（其他语言也都有实现）。所以对于HyperLogLog我们有两个点需要分析，第一个是什么是HyperLogLog，第二个是Redis是如何实现HyperLogLog的。

##什么是HyperLogLog

> [HyperLogLog 算法的原理讲解以及 Redis的应用](https://www.cnblogs.com/linguanh/p/10460421.html)

这个链接中整理的非常详细，但是还是希望用自己的语言描述一遍才能加深理解。

##数学基础 -- 伯努利实验

一枚硬币抛一次有正面和反面，而且通常来说概率是50%，如果把一次抛掷硬币作为一次实验，其中就有可能会出现连续多次都是正面的情况。

那么就有在n次实验中，连续出现的最多次的正面数量记为k。那么就可以发现在投掷次数n和k中是有一个关系的，通过极大似然估计可以得到如下结论：
$$
n\leq2^{k}
$$
这样的公式虽然很好，但是在样本数量很小的时候，估计的误差会非常大。

###调节误差

那么既然上述估计的误差非常大，就需要通过一些手段来避免这样的误差，最终采取的方式就是一次连续实验带来误差很多，我就通过很多组实现求平均值来实现更为精确的估计。

那么对于k组实验求代数平均数的就是LogLog算法，对于k组实验求调和平均数的，就是HyperLogLog算法了。

## Redis的HyperLogLog

####思想

在分析源码之前，先来分析清楚Redis是如何把HyperLogLog用计算机算法和内存模型对应起来的。

对于任意输入x，这个x总可以通过一个hash算法得到一个数字，既然是一个数字，也一定有二进制的表达，那么只要看从左至右出现的第一个1（从右至左或者出现的第一个0都是一样的，就用这个距离），出现的第一个1就好比是连续抛掷硬币出现的第一个非正面，那么就可以通过这样的方式来估计n的数量。

#### 设计

Redis把每一个输入通过MurMur64的Hash算法，变成一个64位的哈希值。然后把14位作为分组（就好比多次实验取调和平均数），然后前50位（二进制0和1）作为抛掷硬币的结果。

Redis总过分了16384个组，因为2^14^ = 16384，这也是为什么要取14个bit作为分组的理由。

每个组用6个bit位来表示连续出现正面的最大次数，言外之意就是不能超过2^6^ - 1 = 63个，但是因为我们只用了50个bit来计算，所以一定不会超过63。

那么对于HyperLogLog的实现需要的内存就是16384 * 6 / 8 /1024 = 12kb。也就是可以通过12k的内存来估计最大不超过2^64^个（大约是10^19^次方这个量级的）数字。那么为什么是2的64次方，因为我们把一个输入hash为了64位hash值，那么这个64位hash值就有2^64^种取值，因此就是这么计算得到的。

> 不过这里插一句，Redis的实现并不是每一个HyperLogLog都用了12k，因为对于Redis这样把内存使用到极致的KV数据库，一定有一种节省内存的方式，如果看源码就是Dense模式和Sparse模式，稀疏和密集两种存储，之后再来分析，不过即便是稀疏，实现的原理也是相同的，只不过存储数据的时候用了一些小技巧。

#### 偏差修正

由于HyperLogLog是一个估计算法，那么为了提高估计的精度，有一个修正参数，把估计到的结果再乘以这个修正系数就是我们本次估计的最终结果。

### Redis代码实现

关于代码实现，又是一个超级复杂的实现@_@，所以总体来说还是要分析流程为主，一些实现和算法的细节，就没有办法面面俱到了。

```c
struct hllhdr {
  char magic[4]; // 魔术字符串“HYLL”
  uint8_t encoding; // 编码类型，稀疏或密集
  uint8_t notused[3]; // 暂时没用
  uint8_t card[8]; // 貌似是为缓存稀疏时频繁遍历的问题做的一个缓存。
  uint8_t registers[]; // 数据
};
```

### 稀疏、密集编码

稀疏类型是为了解决这一的一个问题而出现的：Redis分了16384个组，那么很有可能数据量不大的情况下，只有三四个组里面是有数据的，其他组都没有数据。在这样的情况下如果用12k去存储对内存的浪费就非常非常大了，因此就用了稀疏的方式来存储。其逻辑就是比如我现在有如下数据：

1. 5号小组：8
2. 1456号小组：3
3. 9800号小组：3

那么存储的时候就可以这样存储：

1. 连续4个小组0
2. 一个小组8
3. 连续1451个小组0
4. 一个小组3
5. 连续8344个小组0
6. 一个小组3

这样的方式就可以通过几十个字节来达到代替12k内存的目的。

当然这样的稀疏方式在一定的情况下会变成12k的密集方式，这个之后会分析。

### 添加一个元素到HyperLogLog

```c
int hllAdd(robj *o, unsigned char *ele, size_t elesize) {
  struct hllhdr *hdr = o->ptr;
  switch(hdr->encoding) {
    case HLL_DENSE: return hllDenseAdd(hdr->registers,ele,elesize);
    case HLL_SPARSE: return hllSparseAdd(o,ele,elesize);
    default: return -1; /* Invalid representation. */
  }
}
```

可以看到，在处理时会根据是稀疏类型还是密集类型来分开处理。其中密集类型**HLL_SPARSE**其实就是12k的完整版，重点有区别的其实是DENSE稀疏类型。

```c
int hllDenseAdd(uint8_t *registers, unsigned char *ele, size_t elesize) {
  long index;
  // 这个count就是前面说的有连续多少个0，然后又了这个count之后就可以估算到底有多少次实验
  uint8_t count = hllPatLen(ele,elesize,&index);
	// 这个方法就是更新该分组下的连续最大的0的个数
  return hllDenseSet(registers,index,count);
}

int hllPatLen(unsigned char *ele, size_t elesize, long *regp) {
  uint64_t hash, bit, index;
  int count;

	// murmur64哈希算法计算hash值
  hash = MurmurHash64A(ele,elesize,0xadc83b19ULL);
  // 找到第几个分组，后14位作为index
  index = hash & HLL_P_MASK; /* Register index. */
  // 右移14位，前50位作为hash值
  hash >>= HLL_P; /* Remove bits used to address the register. */
  // 最左边的那个数字确保为1，目的就是为了确保至少有一个1
  hash |= ((uint64_t)1<<HLL_Q);
  bit = 1;
  count = 1; /* Initialized to 1 since we count the "00000...1" pattern. */
  // 这个就是看连续多少个0，一直到出现一个1为止
  while((hash & bit) == 0) {
    count++;
    bit <<= 1;
  }
  *regp = (int) index;
  return count;
}

int hllDenseSet(uint8_t *registers, long index, uint8_t count) {
  uint8_t oldcount;
	
  // 这个就是看该分组下以前的值是多少
  // 这个方法的大致实现就是，因为每6个bit一个组，所以看我们需要第几个组来计算在第几个字节数据下，然后看是不是有前后跨字节的问题，反正就是准确定位到属于本组的那6个bit，然后返回最终结果。
  HLL_DENSE_GET_REGISTER(oldcount,registers,index);
  // 如果新的count比以前的大，就更新
  if (count > oldcount) {
    // 这个就是更新一下count
    HLL_DENSE_SET_REGISTER(registers,index,count);
    return 1;
  } else {
    // 否则就直接返回，因为我们要的是连续抛掷硬币出现正面次数最多的那个数字，其他的可以忽略
    return 0;
  }
}
```

### 根据HyperLogLog进行估计

上面说了如何添加一个元素，那么添加了元素之后的目的就是为了对总量进行估计，下面就看一下如何统计HyperLogLog里面的总量。

```c
void pfcountCommand(client *c) {
  robj *o;
  struct hllhdr *hdr;
  uint64_t card;

	// 省略统计多个HyperLogLog的场景，做法就是先merge一下，然后在count，所以就忽略了。

	// 找到HyperLogLog这个RedisObject
  o = lookupKeyWrite(c->db,c->argv[1]);
  if (o == NULL) {
    /* No key? Cardinality is zero since no element was added, otherwise
         * we would have a key as HLLADD creates it as a side effect. */
    addReply(c,shared.czero);
  } else {
    if (isHLLObjectOrReply(c,o) != C_OK) return;
    o = dbUnshareStringValue(c->db,c->argv[1],o);

    // 看缓存是不是有效
    hdr = o->ptr;
    if (HLL_VALID_CACHE(hdr)) {
      /* Just return the cached value. */
      card = (uint64_t)hdr->card[0];
      card |= (uint64_t)hdr->card[1] << 8;
      card |= (uint64_t)hdr->card[2] << 16;
      card |= (uint64_t)hdr->card[3] << 24;
      card |= (uint64_t)hdr->card[4] << 32;
      card |= (uint64_t)hdr->card[5] << 40;
      card |= (uint64_t)hdr->card[6] << 48;
      card |= (uint64_t)hdr->card[7] << 56;
    } else {
      int invalid = 0;
      // 计算真正的估计值
      card = hllCount(hdr,&invalid);
      if (invalid) {
        addReplySds(c,sdsnew(invalid_hll_err));
        return;
      }
      // 把结果缓存起来
      hdr->card[0] = card & 0xff;
      hdr->card[1] = (card >> 8) & 0xff;
      hdr->card[2] = (card >> 16) & 0xff;
      hdr->card[3] = (card >> 24) & 0xff;
      hdr->card[4] = (card >> 32) & 0xff;
      hdr->card[5] = (card >> 40) & 0xff;
      hdr->card[6] = (card >> 48) & 0xff;
      hdr->card[7] = (card >> 56) & 0xff;
      signalModifiedKey(c->db,c->argv[1]);
      server.dirty++;
    }
    addReplyLongLong(c,card);
  }
}

uint64_t hllCount(struct hllhdr *hdr, int *invalid) {
  double m = HLL_REGISTERS;
  double E;
  int j;
  int reghisto[64] = {0};

  /* Compute register histogram */
  if (hdr->encoding == HLL_DENSE) {
    // 紧密排列
    hllDenseRegHisto(hdr->registers,reghisto);
  } else if (hdr->encoding == HLL_SPARSE) {
    // 稀疏排列
    hllSparseRegHisto(hdr->registers,
                      sdslen((sds)hdr)-HLL_HDR_SIZE,invalid,reghisto);
  } else if (hdr->encoding == HLL_RAW) {
    // 好像是merge的时候才会用到？一个内部使用的东西，对外无感，而且也不是什么优化。。有点没看懂
    hllRawRegHisto(hdr->registers,reghisto);
  } else {
    serverPanic("Unknown HyperLogLog encoding in hllCount()");
  }

	// 这里就是根据收集来数据进行估计，不过有一点没看懂。。。反正就是一个估计算法，不打算花太多精力研究了。
  double z = m * hllTau((m-reghisto[HLL_Q+1])/(double)m);
  for (j = HLL_Q; j >= 1; --j) {
    z += reghisto[j];
    z *= 0.5;
  }
  z += m * hllSigma(reghisto[0]/(double)m);
  E = llroundl(HLL_ALPHA_INF*m*m/z);

  return (uint64_t) E;
}
```

就以密集排列为例，

```c
void hllDenseRegHisto(uint8_t *registers, int* reghisto) {
  int j;

  // Redis的实现
  if (HLL_REGISTERS == 16384 && HLL_BITS == 6) {
    uint8_t *r = registers;
    unsigned long r0, r1, r2, r3, r4, r5, r6, r7, r8, r9,
    r10, r11, r12, r13, r14, r15;
    for (j = 0; j < 1024; j++) {
      // 下面就是用1024次循环，每次去16个小组的值，然后把得到的这些值放到reghisto里面。
      r0 = r[0] & 63;
      r1 = (r[0] >> 6 | r[1] << 2) & 63;
      r2 = (r[1] >> 4 | r[2] << 4) & 63;
      r3 = (r[2] >> 2) & 63;
      r4 = r[3] & 63;
      r5 = (r[3] >> 6 | r[4] << 2) & 63;
      r6 = (r[4] >> 4 | r[5] << 4) & 63;
      r7 = (r[5] >> 2) & 63;
      r8 = r[6] & 63;
      r9 = (r[6] >> 6 | r[7] << 2) & 63;
      r10 = (r[7] >> 4 | r[8] << 4) & 63;
      r11 = (r[8] >> 2) & 63;
      r12 = r[9] & 63;
      r13 = (r[9] >> 6 | r[10] << 2) & 63;
      r14 = (r[10] >> 4 | r[11] << 4) & 63;
      r15 = (r[11] >> 2) & 63;

      // reghisto[rx]++的意思就是，因为reghisto可以放64个数字，其实就是0-63
      // 那么又因为一个组可以放6个bit，也是64个取值范围
      // 这里算出来的rx就是连续多少次抛掷硬币正面向上的最大次数
      // 所以这里就是统计各个小组的连续最大次数分别出现了多少，后续需要根据这个算调和平均数
      reghisto[r0]++;
      reghisto[r1]++;
      reghisto[r2]++;
      reghisto[r3]++;
      reghisto[r4]++;
      reghisto[r5]++;
      reghisto[r6]++;
      reghisto[r7]++;
      reghisto[r8]++;
      reghisto[r9]++;
      reghisto[r10]++;
      reghisto[r11]++;
      reghisto[r12]++;
      reghisto[r13]++;
      reghisto[r14]++;
      reghisto[r15]++;

      // 这个就是6个bit*16个小组 = 96个bit
      // 8个bit * 12个字节 = 96个bit
      // 所以一次迭代需要读取12个字节的数据
      // 一次迭代结束之后就往后挪12个字节，继续推进
      r += 12;
    }
  } else {
    // 这是个啥。。有可能会进来嘛？
    for(j = 0; j < HLL_REGISTERS; j++) {
      unsigned long reg;
      HLL_DENSE_GET_REGISTER(reg,registers,j);
      reghisto[reg]++;
    }
  }
}
```

那么到这里为止，HyperLogLog的基本原理已经分析清楚了

###稀疏和密集的转换

稀疏的存储前面已经大致分析过，那么时候稀疏的编码方式会被转化为密集的方式呢，主要分成两种：

1. 单个小组连续出现0的次数 > 32，这个是因为稀疏的存储方式，就存不下超过32的情况
2. 稀疏的编码占用内存超过3000个字节

然后就会一次性的从稀疏转化为密集，并且转成密集了，就不会再回退到稀疏了。