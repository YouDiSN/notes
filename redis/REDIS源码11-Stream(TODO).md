#REDIS源码11-Stream

前面分析了Stream中用到的两个非常重要的数据结构，rax和listpack，那么具体看一下Stream是如何实现一个简单的消息队列的。

```c
typedef struct stream {
  rax *rax;               /* The radix tree holding the stream. */
  uint64_t length;        /* Number of elements inside this stream. */
  streamID last_id;       /* Zero if there are yet no items. */
  rax *cgroups;           /* Consumer groups dictionary: name -> streamCG */
} stream;
```

##操作

[Redis Stream分析](https://blog.csdn.net/qq_31720329/article/details/103643038)

源码实在太长了，而且结构非常复杂，所以通过别人的分析来做一些直观的认识就可以了。

不过总结的来说，大致是以下这样的流程：

1. 根据命令的key找到dict中的Stream结构
2. 根据传入消息id（必须确保递增），或者通过“*”可以让redis自己生成一个（时间戳-序数）。
3. 然后通过消息ID，在Stream中的Rax中去寻找RaxNode，如果找到了就获取里面的listpack，否则就生成一个新的并插入
4. 上面一步已经有了listpack，然后看是不是长度过长或者bytes数量过多，这样就重新弄一个新的把老的替换掉
   1. 如果是新建一个，那么就要作为master node插入一些结构，比如总数、deleted数量和fields数量等
   2. 否则就找到最后一个节点，然后在最后一个节点插入。插入的时候和master的field对比，如果是相同的，那么就可以用节省的方式存储，否则就field+value的方式存储
5. 这里就已经把一个entry插入到listpack中，然后其实作为消息队列而言，入队已经完毕了

### 插入

大概的做法就是，

首先从Redis的Dict上着根据client提供的key找到stream对象，然后根据entry ID在字典树中寻找字典树节点，找到的节点里面存了一个listpack，然后把要添加的内容放到这个listpack上，这个listpack的结果如下：

第一个节点是master节点

```c
+-------+---------+------------+---------+--/--+---------+---------+-+
| count | deleted | num-fields | field_1 | field_2 | ... | field_N |0|
+-------+---------+------------+---------+--/--+---------+---------+-+
```

然后后面的数据分为两种情况：第一种是field不同，第二种是field都相同

```c
+-----+--------+----------+-------+-------+-/-+-------+-------+--------+
|flags|entry-id|num-fields|field-1|value-1|...|field-N|value-N|lp-count|
+-----+--------+----------+-------+-------+-/-+-------+-------+--------+
```

```c
+-----+--------+-------+-/-+-------+--------+
|flags|entry-id|value-1|...|value-N|lp-count|
+-----+--------+-------+-/-+-------+--------+
```

其实就节省了不用每次都存field了。

##读取数据

上面是消息队列的入队，既然已经入队了，我们就需要出队。

啊啊啊啊啊看不懂！！！啊啊啊啊啊看不懂！！！啊啊啊啊啊看不懂！！！

啊啊啊啊啊太难了！！！啊啊啊啊啊太难了！！！啊啊啊啊啊太难了！！！

啊啊啊啊啊后面再补！！！啊啊啊啊啊后面再补！！！啊啊啊啊啊后面再补！！！