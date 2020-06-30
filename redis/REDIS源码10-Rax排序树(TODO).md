#REDIS源码10-Rax字典树

前面分析的Stream有两个重要的数据结构，一个是listpack，也就是5.0里新增的将来可能用来代替ziplist的数据结构，还有一个就是rax字典树。

字典书本身非常经典，就不做过多的分析了，直接分析源码。

```c
typedef struct rax {
  raxNode *head; // 头节点
  uint64_t numele; // 存储的元素个数
  uint64_t numnodes; // 树的node数量
} rax;

typedef struct raxNode {
  uint32_t iskey:1; // 当前这个node是否包含key
  uint32_t isnull:1; // value是不是为null
  uint32_t iscompr:1; // 节点是否被压缩了
  uint32_t size:29; // children的数量或者压缩字符串的长度
  unsigned char data[]; // 具体的数据
} raxNode;
```

## 基本操作

```c
int raxInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old) {
  return raxGenericInsert(rax,s,len,data,old,1);
}
```

**raxGenericInsert**方法的实现实在是太长了，有三百多行。。但是还是要分析一下— —#

<img src="imgs/当我打算分析rax时.png" alt="image-20200229223005383" style="zoom:50%;" />

```c
int raxGenericInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old, int overwrite) {
  size_t i;
  int j = 0; /* Split position. If raxLowWalk() stops in a compressed
                  node, the index 'j' represents the char we stopped within the
                  compressed node, that is, the position where to split the
                  node for insertion. */
  raxNode *h, **parentlink;

  // 这个raxLowWalk的作用就是从当前的字典树上找到字符串s，返回的这个i表示当前树上已有的长度，比如apple，可能现在树上已经有app，那么返回的i = 3
  i = raxLowWalk(rax,s,len,&h,&parentlink,&j,NULL);

  // 这个就代表直接在已有的节点上添加指针就可以了
  if (i == len && (!h->iscompr || j == 0 /* not in the middle if j is 0 */)) {
    /* Make space for the value pointer if needed. */
    if (!h->iskey || (h->isnull && overwrite)) {
      h = raxReallocForData(h,data);
      if (h) memcpy(parentlink,&h,sizeof(h));
    }
    if (h == NULL) {
      errno = ENOMEM;
      return 0;
    }

    // 这个是直接修改已有的数据就可以了
    if (h->iskey) {
      if (old) *old = raxGetData(h);
      if (overwrite) raxSetData(h,data);
      errno = 0;
      return 0; /* Element already exists. */
    }

    // 设置data
    raxSetData(h,data);
    // 元素数量+1
    rax->numele++;
    return 1; /* Element inserted. */
  }

  // 后面的算法主要就是按照这边注释的流程，代码实在太长了。。仔细阅读这段翻译可以很好的理解都干了些什么事情
  /* If the node we stopped at is a compressed node, we need to
     * split it before to continue.
     *
     * Splitting a compressed node have a few possible cases.
     * Imagine that the node 'h' we are currently at is a compressed
     * node contaning the string "ANNIBALE" (it means that it represents
     * nodes A -> N -> N -> I -> B -> A -> L -> E with the only child
     * pointer of this node pointing at the 'E' node, because remember that
     * we have characters at the edges of the graph, not inside the nodes
     * themselves.
     *
     * In order to show a real case imagine our node to also point to
     * another compressed node, that finally points at the node without
     * children, representing 'O':
     *
     *     "ANNIBALE" -> "SCO" -> []
     *
     * When inserting we may face the following cases. Note that all the cases
     * require the insertion of a non compressed node with exactly two
     * children, except for the last case which just requires splitting a
     * compressed node.
     *
     * 1) Inserting "ANNIENTARE"
     *
     *               |B| -> "ALE" -> "SCO" -> []
     *     "ANNI" -> |-|
     *               |E| -> (... continue algo ...) "NTARE" -> []
     *
     * 2) Inserting "ANNIBALI"
     *
     *                  |E| -> "SCO" -> []
     *     "ANNIBAL" -> |-|
     *                  |I| -> (... continue algo ...) []
     *
     * 3) Inserting "AGO" (Like case 1, but set iscompr = 0 into original node)
     *
     *            |N| -> "NIBALE" -> "SCO" -> []
     *     |A| -> |-|
     *            |G| -> (... continue algo ...) |O| -> []
     *
     * 4) Inserting "CIAO"
     *
     *     |A| -> "NNIBALE" -> "SCO" -> []
     *     |-|
     *     |C| -> (... continue algo ...) "IAO" -> []
     *
     * 5) Inserting "ANNI"
     *
     *     "ANNI" -> "BALE" -> "SCO" -> []
     *
     * The final algorithm for insertion covering all the above cases is as
     * follows.
     *
     * ============================= ALGO 1 =============================
     *
     * For the above cases 1 to 4, that is, all cases where we stopped in
     * the middle of a compressed node for a character mismatch, do:
     *
     * Let $SPLITPOS be the zero-based index at which, in the
     * compressed node array of characters, we found the mismatching
     * character. For example if the node contains "ANNIBALE" and we add
     * "ANNIENTARE" the $SPLITPOS is 4, that is, the index at which the
     * mismatching character is found.
     *
     * 1. Save the current compressed node $NEXT pointer (the pointer to the
     *    child element, that is always present in compressed nodes).
     *
     * 2. Create "split node" having as child the non common letter
     *    at the compressed node. The other non common letter (at the key)
     *    will be added later as we continue the normal insertion algorithm
     *    at step "6".
     *
     * 3a. IF $SPLITPOS == 0:
     *     Replace the old node with the split node, by copying the auxiliary
     *     data if any. Fix parent's reference. Free old node eventually
     *     (we still need its data for the next steps of the algorithm).
     *
     * 3b. IF $SPLITPOS != 0:
     *     Trim the compressed node (reallocating it as well) in order to
     *     contain $splitpos characters. Change chilid pointer in order to link
     *     to the split node. If new compressed node len is just 1, set
     *     iscompr to 0 (layout is the same). Fix parent's reference.
     *
     * 4a. IF the postfix len (the length of the remaining string of the
     *     original compressed node after the split character) is non zero,
     *     create a "postfix node". If the postfix node has just one character
     *     set iscompr to 0, otherwise iscompr to 1. Set the postfix node
     *     child pointer to $NEXT.
     *
     * 4b. IF the postfix len is zero, just use $NEXT as postfix pointer.
     *
     * 5. Set child[0] of split node to postfix node.
     *
     * 6. Set the split node as the current node, set current index at child[1]
     *    and continue insertion algorithm as usually.
     *
     * ============================= ALGO 2 =============================
     *
     * For case 5, that is, if we stopped in the middle of a compressed
     * node but no mismatch was found, do:
     *
     * Let $SPLITPOS be the zero-based index at which, in the
     * compressed node array of characters, we stopped iterating because
     * there were no more keys character to match. So in the example of
     * the node "ANNIBALE", addig the string "ANNI", the $SPLITPOS is 4.
     *
     * 1. Save the current compressed node $NEXT pointer (the pointer to the
     *    child element, that is always present in compressed nodes).
     *
     * 2. Create a "postfix node" containing all the characters from $SPLITPOS
     *    to the end. Use $NEXT as the postfix node child pointer.
     *    If the postfix node length is 1, set iscompr to 0.
     *    Set the node as a key with the associated value of the new
     *    inserted key.
     *
     * 3. Trim the current node to contain the first $SPLITPOS characters.
     *    As usually if the new node length is just 1, set iscompr to 0.
     *    Take the iskey / associated value as it was in the orignal node.
     *    Fix the parent's reference.
     *
     * 4. Set the postfix node as the only child pointer of the trimmed
     *    node created at step 1.
     */

  /* ------------------------- ALGORITHM 1 --------------------------- */
  if (h->iscompr && i != len) {
    debugf("ALGO 1: Stopped at compressed node %.*s (%p)\n",
           h->size, h->data, (void*)h);
    debugf("Still to insert: %.*s\n", (int)(len-i), s+i);
    debugf("Splitting at %d: '%c'\n", j, ((char*)h->data)[j]);
    debugf("Other (key) letter is '%c'\n", s[i]);

    /* 1: Save next pointer. */
    raxNode **childfield = raxNodeLastChildPtr(h);
    raxNode *next;
    memcpy(&next,childfield,sizeof(next));
    debugf("Next is %p\n", (void*)next);
    debugf("iskey %d\n", h->iskey);
    if (h->iskey) {
      debugf("key value is %p\n", raxGetData(h));
    }

    /* Set the length of the additional nodes we will need. */
    size_t trimmedlen = j;
    size_t postfixlen = h->size - j - 1;
    int split_node_is_key = !trimmedlen && h->iskey && !h->isnull;
    size_t nodesize;

    /* 2: Create the split node. Also allocate the other nodes we'll need
         *    ASAP, so that it will be simpler to handle OOM. */
    raxNode *splitnode = raxNewNode(1, split_node_is_key);
    raxNode *trimmed = NULL;
    raxNode *postfix = NULL;

    if (trimmedlen) {
      nodesize = sizeof(raxNode)+trimmedlen+raxPadding(trimmedlen)+
        sizeof(raxNode*);
      if (h->iskey && !h->isnull) nodesize += sizeof(void*);
      trimmed = rax_malloc(nodesize);
    }

    if (postfixlen) {
      nodesize = sizeof(raxNode)+postfixlen+raxPadding(postfixlen)+
        sizeof(raxNode*);
      postfix = rax_malloc(nodesize);
    }

    /* OOM? Abort now that the tree is untouched. */
    if (splitnode == NULL ||
        (trimmedlen && trimmed == NULL) ||
        (postfixlen && postfix == NULL))
    {
      rax_free(splitnode);
      rax_free(trimmed);
      rax_free(postfix);
      errno = ENOMEM;
      return 0;
    }
    splitnode->data[0] = h->data[j];

    if (j == 0) {
      /* 3a: Replace the old node with the split node. */
      if (h->iskey) {
        void *ndata = raxGetData(h);
        raxSetData(splitnode,ndata);
      }
      memcpy(parentlink,&splitnode,sizeof(splitnode));
    } else {
      /* 3b: Trim the compressed node. */
      trimmed->size = j;
      memcpy(trimmed->data,h->data,j);
      trimmed->iscompr = j > 1 ? 1 : 0;
      trimmed->iskey = h->iskey;
      trimmed->isnull = h->isnull;
      if (h->iskey && !h->isnull) {
        void *ndata = raxGetData(h);
        raxSetData(trimmed,ndata);
      }
      raxNode **cp = raxNodeLastChildPtr(trimmed);
      memcpy(cp,&splitnode,sizeof(splitnode));
      memcpy(parentlink,&trimmed,sizeof(trimmed));
      parentlink = cp; /* Set parentlink to splitnode parent. */
      rax->numnodes++;
    }

    /* 4: Create the postfix node: what remains of the original
         * compressed node after the split. */
    if (postfixlen) {
      /* 4a: create a postfix node. */
      postfix->iskey = 0;
      postfix->isnull = 0;
      postfix->size = postfixlen;
      postfix->iscompr = postfixlen > 1;
      memcpy(postfix->data,h->data+j+1,postfixlen);
      raxNode **cp = raxNodeLastChildPtr(postfix);
      memcpy(cp,&next,sizeof(next));
      rax->numnodes++;
    } else {
      /* 4b: just use next as postfix node. */
      postfix = next;
    }

    /* 5: Set splitnode first child as the postfix node. */
    raxNode **splitchild = raxNodeLastChildPtr(splitnode);
    memcpy(splitchild,&postfix,sizeof(postfix));

    /* 6. Continue insertion: this will cause the splitnode to
         * get a new child (the non common character at the currently
         * inserted key). */
    rax_free(h);
    h = splitnode;
  } else if (h->iscompr && i == len) {
    /* ------------------------- ALGORITHM 2 --------------------------- */
    debugf("ALGO 2: Stopped at compressed node %.*s (%p) j = %d\n",
           h->size, h->data, (void*)h, j);

    /* Allocate postfix & trimmed nodes ASAP to fail for OOM gracefully. */
    size_t postfixlen = h->size - j;
    size_t nodesize = sizeof(raxNode)+postfixlen+raxPadding(postfixlen)+
      sizeof(raxNode*);
    if (data != NULL) nodesize += sizeof(void*);
    raxNode *postfix = rax_malloc(nodesize);

    nodesize = sizeof(raxNode)+j+raxPadding(j)+sizeof(raxNode*);
    if (h->iskey && !h->isnull) nodesize += sizeof(void*);
    raxNode *trimmed = rax_malloc(nodesize);

    if (postfix == NULL || trimmed == NULL) {
      rax_free(postfix);
      rax_free(trimmed);
      errno = ENOMEM;
      return 0;
    }

    /* 1: Save next pointer. */
    raxNode **childfield = raxNodeLastChildPtr(h);
    raxNode *next;
    memcpy(&next,childfield,sizeof(next));

    /* 2: Create the postfix node. */
    postfix->size = postfixlen;
    postfix->iscompr = postfixlen > 1;
    postfix->iskey = 1;
    postfix->isnull = 0;
    memcpy(postfix->data,h->data+j,postfixlen);
    raxSetData(postfix,data);
    raxNode **cp = raxNodeLastChildPtr(postfix);
    memcpy(cp,&next,sizeof(next));
    rax->numnodes++;

    /* 3: Trim the compressed node. */
    trimmed->size = j;
    trimmed->iscompr = j > 1;
    trimmed->iskey = 0;
    trimmed->isnull = 0;
    memcpy(trimmed->data,h->data,j);
    memcpy(parentlink,&trimmed,sizeof(trimmed));
    if (h->iskey) {
      void *aux = raxGetData(h);
      raxSetData(trimmed,aux);
    }

    /* Fix the trimmed node child pointer to point to
         * the postfix node. */
    cp = raxNodeLastChildPtr(trimmed);
    memcpy(cp,&postfix,sizeof(postfix));

    /* Finish! We don't need to continue with the insertion
         * algorithm for ALGO 2. The key is already inserted. */
    rax->numele++;
    rax_free(h);
    return 1; /* Key inserted. */
  }

  /* We walked the radix tree as far as we could, but still there are left
     * chars in our string. We need to insert the missing nodes. */
  while(i < len) {
    raxNode *child;

    /* If this node is going to have a single child, and there
         * are other characters, so that that would result in a chain
         * of single-childed nodes, turn it into a compressed node. */
    if (h->size == 0 && len-i > 1) {
      debugf("Inserting compressed node\n");
      size_t comprsize = len-i;
      if (comprsize > RAX_NODE_MAX_SIZE)
        comprsize = RAX_NODE_MAX_SIZE;
      raxNode *newh = raxCompressNode(h,s+i,comprsize,&child);
      if (newh == NULL) goto oom;
      h = newh;
      memcpy(parentlink,&h,sizeof(h));
      parentlink = raxNodeLastChildPtr(h);
      i += comprsize;
    } else {
      debugf("Inserting normal node\n");
      raxNode **new_parentlink;
      raxNode *newh = raxAddChild(h,s[i],&child,&new_parentlink);
      if (newh == NULL) goto oom;
      h = newh;
      memcpy(parentlink,&h,sizeof(h));
      parentlink = new_parentlink;
      i++;
    }
    rax->numnodes++;
    h = child;
  }
  raxNode *newh = raxReallocForData(h,data);
  if (newh == NULL) goto oom;
  h = newh;
  if (!h->iskey) rax->numele++;
  raxSetData(h,data);
  memcpy(parentlink,&h,sizeof(h));
  return 1; /* Element inserted. */

  oom:
  /* This code path handles out of memory after part of the sub-tree was
     * already modified. Set the node as a key, and then remove it. However we
     * do that only if the node is a terminal node, otherwise if the OOM
     * happened reallocating a node in the middle, we don't need to free
     * anything. */
  if (h->size == 0) {
    h->isnull = 1;
    h->iskey = 1;
    rax->numele++; /* Compensate the next remove. */
    assert(raxRemove(rax,s,i,NULL) != 0);
  }
  errno = ENOMEM;
  return 0;
}
```

上面的代码实在有点长，不过大概的意思是就是通过遍历政客字典树，然后根据现有的数据来决定是不是需要分割当前的节点，然后根据不同的case，来决定分割后的节点如何分配和存储。

### 遍历

关于字典树的遍历代码也很长，所以就没有再去花费大量的笔墨去分析了，大概做的事情就是遍历一整棵树，然后根据比对的结果判断是否依旧存在所需要的字符串了。

### 删除

```c
int raxRemove(rax *rax, unsigned char *s, size_t len, void **old) {
  raxNode *h;
  raxStack ts;

  // 初始化一个栈，这个栈的作用就是用来记录整个树的遍历过程的
  // 因为删除之后要根据一些情况对parent节点进行调整，所以如果重新遍历就会比较好费性能，就用一个stack记录下来，这样后续就不用在重新遍历一次了。
  raxStackInit(&ts);
  int splitpos = 0;
  // 这个还是一样的找到对应的节点
  size_t i = raxLowWalk(rax,s,len,&h,NULL,&splitpos,&ts);
  // 如果找不到节点就直接返回没有，同时释放stack的内存。
  if (i != len || (h->iscompr && splitpos != 0) || !h->iskey) {
    raxStackFree(&ts);
    return 0;
  }
  // 获取之前的指针，这个指针指向之前的数据
  if (old) *old = raxGetData(h);
  h->iskey = 0;
  rax->numele--;

  /* If this node has no children, the deletion needs to reclaim the
     * no longer used nodes. This is an iterative process that needs to
     * walk the three upward, deleting all the nodes with just one child
     * that are not keys, until the head of the rax is reached or the first
     * node with more than one child is found. */

  int trycompress = 0; /* Will be set to 1 if we should try to optimize the
                            tree resulting from the deletion. */

  // 这个size等于0的意思是就是没有孩子节点，可以直接释放自己
  if (h->size == 0) {
    raxNode *child = NULL;
    // 只要h不是头节点，就一层层往上遍历去释放节点
    while(h != rax->head) {
      child = h;
      // 释放内存
      rax_free(child);
      // node数量-1
      rax->numnodes--;
      // h就从前面遍历找这个节点的时候，存储下来的栈中弹出一个节点，意思就是找到父节点，然后从下往上处理
      h = raxStackPop(&ts);
      // 如果这个节点不可以删除就到此为止。
      if (h->iskey || (!h->iscompr && h->size != 1)) break;
    }
    if (child) {
      // 这个就是上一个节点
      raxNode *new = raxRemoveChild(h,child);
      if (new != h) {
        raxNode *parent = raxStackPeek(&ts);
        raxNode **parentlink;
        if (parent == NULL) {
          parentlink = &rax->head;
        } else {
          parentlink = raxFindParentLink(parent,h);
        }
        memcpy(parentlink,&new,sizeof(new));
      }

      /* If after the removal the node has just a single child
             * and is not a key, we need to try to compress it. */
      if (new->size == 1 && new->iskey == 0) {
        trycompress = 1;
        h = new;
      }
    }
  } else if (h->size == 1) {
    /* If the node had just one child, after the removal of the key
         * further compression with adjacent nodes is pontentially possible. */
    trycompress = 1;
  }

  /* Don't try node compression if our nodes pointers stack is not
     * complete because of OOM while executing raxLowWalk() */
  if (trycompress && ts.oom) trycompress = 0;

  /* Recompression: if trycompress is true, 'h' points to a radix tree node
     * that changed in a way that could allow to compress nodes in this
     * sub-branch. Compressed nodes represent chains of nodes that are not
     * keys and have a single child, so there are two deletion events that
     * may alter the tree so that further compression is needed:
     *
     * 1) A node with a single child was a key and now no longer is a key.
     * 2) A node with two children now has just one child.
     *
     * We try to navigate upward till there are other nodes that can be
     * compressed, when we reach the upper node which is not a key and has
     * a single child, we scan the chain of children to collect the
     * compressable part of the tree, and replace the current node with the
     * new one, fixing the child pointer to reference the first non
     * compressable node.
     *
     * Example of case "1". A tree stores the keys "FOO" = 1 and
     * "FOOBAR" = 2:
     *
     *
     * "FOO" -> "BAR" -> [] (2)
     *           (1)
     *
     * After the removal of "FOO" the tree can be compressed as:
     *
     * "FOOBAR" -> [] (2)
     *
     *
     * Example of case "2". A tree stores the keys "FOOBAR" = 1 and
     * "FOOTER" = 2:
     *
     *          |B| -> "AR" -> [] (1)
     * "FOO" -> |-|
     *          |T| -> "ER" -> [] (2)
     *
     * After the removal of "FOOTER" the resulting tree is:
     *
     * "FOO" -> |B| -> "AR" -> [] (1)
     *
     * That can be compressed into:
     *
     * "FOOBAR" -> [] (1)
     */
  if (trycompress) {
    debugf("After removing %.*s:\n", (int)len, s);
    debugnode("Compression may be needed",h);
    debugf("Seek start node\n");

    /* Try to reach the upper node that is compressible.
         * At the end of the loop 'h' will point to the first node we
         * can try to compress and 'parent' to its parent. */
    raxNode *parent;
    while(1) {
      parent = raxStackPop(&ts);
      if (!parent || parent->iskey ||
          (!parent->iscompr && parent->size != 1)) break;
      h = parent;
      debugnode("Going up to",h);
    }
    raxNode *start = h; /* Compression starting node. */

    /* Scan chain of nodes we can compress. */
    size_t comprsize = h->size;
    int nodes = 1;
    while(h->size != 0) {
      raxNode **cp = raxNodeLastChildPtr(h);
      memcpy(&h,cp,sizeof(h));
      if (h->iskey || (!h->iscompr && h->size != 1)) break;
      /* Stop here if going to the next node would result into
             * a compressed node larger than h->size can hold. */
      if (comprsize + h->size > RAX_NODE_MAX_SIZE) break;
      nodes++;
      comprsize += h->size;
    }
    if (nodes > 1) {
      /* If we can compress, create the new node and populate it. */
      size_t nodesize =
        sizeof(raxNode)+comprsize+raxPadding(comprsize)+sizeof(raxNode*);
      raxNode *new = rax_malloc(nodesize);
      /* An out of memory here just means we cannot optimize this
             * node, but the tree is left in a consistent state. */
      if (new == NULL) {
        raxStackFree(&ts);
        return 1;
      }
      new->iskey = 0;
      new->isnull = 0;
      new->iscompr = 1;
      new->size = comprsize;
      rax->numnodes++;

      /* Scan again, this time to populate the new node content and
             * to fix the new node child pointer. At the same time we free
             * all the nodes that we'll no longer use. */
      comprsize = 0;
      h = start;
      while(h->size != 0) {
        memcpy(new->data+comprsize,h->data,h->size);
        comprsize += h->size;
        raxNode **cp = raxNodeLastChildPtr(h);
        raxNode *tofree = h;
        memcpy(&h,cp,sizeof(h));
        rax_free(tofree); rax->numnodes--;
        if (h->iskey || (!h->iscompr && h->size != 1)) break;
      }
      debugnode("New node",new);

      /* Now 'h' points to the first node that we still need to use,
             * so our new node child pointer will point to it. */
      raxNode **cp = raxNodeLastChildPtr(new);
      memcpy(cp,&h,sizeof(h));

      /* Fix parent link. */
      if (parent) {
        raxNode **parentlink = raxFindParentLink(parent,start);
        memcpy(parentlink,&new,sizeof(new));
      } else {
        rax->head = new;
      }

      debugf("Compressed %d nodes, %d total bytes\n",
             nodes, (int)comprsize);
    }
  }
  raxStackFree(&ts);
  return 1;
}
```

。。。啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊

以后有机会再填坑— —#实在有点不大好理解，看不懂