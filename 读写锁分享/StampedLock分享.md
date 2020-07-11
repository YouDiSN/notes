# StampedLock

------

![](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/AQS原理.png)

#Overview

1. StampedLock的state也是用来表示锁的状态。

2. 如果state的第八位是1，那就代表现在有了写锁，如果第八位不是1，那就说明当前不是写模式。
   $$
   {0000\ 0000\ 0000\ 0000\ 0000\ 0000}_{写锁版本}\ 1_{是否是写锁}{000\ 0000}_{当前悲观读的个数}
   $$

3. 读锁需要被记录，但是乐观读没有被记录在state上，目前读锁的记录用了7位来记录。如果读锁的获取次数超过这个数字，那么用了另外一个字段来记录溢出的部分

4. 等待的队列也是使用了CLH的队列，每一个排队的node都有一个字段标示，表明自己是写还是读。

   ![image-20180912221541856](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/StampedLock的node具体实现.png)

5. 所有等待读的线程都会被分配到一起（用链表的方式放到一起），然后头节点是一个普通节点。所有的读线程被一个节点串起来。

6. StampedLock为了防止写锁饥饿，对获取读锁的线程有这样的处理，如果当前锁是读锁，同时排队的node中有一个写的node，那么再来申请获取读锁的线程也需要变成一个node排队。不会直接获取这个读锁。

7. StampedLock在竞争锁时使用了随机的自旋次数来减少线程上下文的切换。

8. StampedLock支持锁的升级和转换（ReentrantReadWriteLock不具备这个功能）

# 内存屏障

------

![image-20180913215107811](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/为何需要了解屏障.png)

![image-20180913215644468](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/VarHandle.png)

![image-20180913215709003](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/unsafe读屏障.png)

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序。

## as-if-serial语义

     As-if-serial语义的意思是，所有的动作(Action)都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。Java编译器、运行时和处理器都会保证单线程下的as-if-serial语义。
     比如，为了保证这一语义，重排序不会发生在有数据依赖的操作之中。

### happens-before原则

在同一个线程中，字节码的先后顺序（program order）也暗含了 happens-before 关系：在程序控制流路径中靠前的字节码 happens-before 靠后的字节码。然而，这并不意味着前者一定在后者之前执行。实际上，如果后者没有观测前者的运行结果，即后者没有数据依赖于前者，那么它们可能会被重排序。

```java
int a = 1;
int b = 2;
int c = a + b;

// 如果是A线程执行这段代码，那么会确保第三行一定是在一二行执行之后的，但是一二行代码谁先执行，对于线程A来说感知不到。
// 如果线程A在执行的同时又有一个线程B，那么在线程B的观察下，完全有可能线程A的执行顺序是 2 -> 1 -> 3。这个就是指令冲排序。
```

> happens-before原则一定是在多线程的场景下，一个线程观察另一个线程时发生的事情。

除了线程内的 happens-before 关系之外，Java 内存模型还定义了下述线程间的 happens-before 关系。

1. 解锁操作 happens-before 之后（这里指时钟顺序先后）对同一把锁的加锁操作。
2. volatile 字段的写操作 happens-before 之后（这里指时钟顺序先后）对同一字段的读操作。
3. 线程的启动操作（即 Thread.starts()） happens-before 该线程的第一个操作。
4. 线程的最后一个操作 happens-before 它的终止事件（即其他线程通过 Thread.isAlive() 或 Thread.join() 判断该线程是否中止）。
5. 线程对其他线程的中断操作 happens-before 被中断线程所收到的中断事件（即被中断线程的 InterruptedException 异常，或者第三个线程针对被中断线程的 Thread.interrupted 或者 Thread.isInterrupted 调用）。
6. 构造器中的最后一个操作 happens-before 析构器的第一个操作。

### 内存一致性

每个CPU都存在Cache，当一个特定数据第一次被其他CPU获取时，此数据显然不在对应CPU的Cache中（这就是Cache Miss）。这意味着CPU要从内存中获取数据（这个过程需要CPU等待数百个周期），此数据将被加载到CPU的Cache中，这样后续就能直接从Cache上快速访问。当某个CPU进行写操作时，他必须确保其他CPU已将此数据从他们的Cache中移除（以便保证一致性），只有在移除操作完成后，此CPU才能安全地修改数据。存在多个Cache时，必须通过一个Cache一致性协议来避免数据不一致的问题，而这个通信的过程就可能导致乱序访问的出现，也就是运行时内存乱序访问。

CPU内存屏障中某些类型的屏障需要成对使用，否则会出错，详细来说就是：一个写操作屏障需要和读操作（或者数据依赖）屏障一起使用（当然，通用屏障也是可以的），反之亦然。

### Unsafe中的native方法

![image-20180905224612143](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/jvm_unsafe方法实现.png)

![image-20180905224703961](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/java.png)

虚拟机中的OrderAccess类是一个根据不同操作系统有不同实现的。

![image-20180905224817560](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/OrderAccess.png)

mac上的实现：

![image-20180905224905484](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/内存屏障.png)

------

#获取凭证

![image-20180909112842087](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/stampedLock的state字段运用.png)

![image-20180913230246242](/Users/xiwang/Documents/java学习笔记/java源码阅读/读写锁分享/StampedLock位运算.png)

不论是获取读锁还是获取写锁，虽然代码块的内容很长，但是总结了下来整体的逻辑还是比较类似的。

1. 先尝试获取一次锁，如果成功就直接返回凭证，如果失败就进入对应的acquireRead或者acquireWrite方法。
2. acquireXXX方法中主要分成以下几个逻辑
   - 第一个死循环就是尝试获取锁，如果不成功就放到CLH队列中
   - 在队列中，如果是头节点后的第一个节点，就不停的自旋尝试获取锁，否则就挂起或者帮助头节点释放被block的线程等。
   - 然后判断自己是不是头节点后的第一个排队节点，如果是的话，就不停的的自旋获取锁；
     如果不是就挂起等待唤醒。

### 乐观读获取凭证

```java
public long tryOptimisticRead() {
	long s;
	// WBIT = 128, 转换成二进制也就是1000 0000,所以（state & WBIT）获取到的结果就是看第八位是1还是0
    // SBITS = -128, 转换成二进制就是 11111111111111111111111110000000，那么(state & SBITS)就是获取state前25位的状态作为一个凭证返回。
	return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}
```

### 写锁获取凭证

```java
@ReservedStackAccess
public long writeLock() {
    long next;
    // 尝试获取写锁，如果获取了，那就直接返回，如果没有获取就阻塞直到获取为止。
    return ((next = tryWriteLock()) != 0L) ? next : acquireWrite(false, 0L);
}

@ReservedStackAccess
public long tryWriteLock() {
    long s;
    // ABITS = 255 11111111, 由于有写锁的的时候，第8位一定是1，有悲观读锁时后7位一定有值，所以这里和ABITS的8个1做比较，就是没有持有写锁，也没有持有悲观读锁，那么就会执行tryWriteLock方法。
    return (((s = state) & ABITS) == 0L) ? tryWriteLock(s) : 0L;
}

private long tryWriteLock(long s) {
    // assert (s & ABITS) == 0L;
    long next;
    // WBIT是255，就是11111111，(s | WBIT)就是把state的后面八位全部变成1。
    if (casState(s, next = s | WBIT)) {
        // 插入storeStore屏障
        VarHandle.storeStoreFence();
        return next;
    }
    return 0L;
}

// 这个方法是最核心的获取写锁的方法。
private long acquireWrite(boolean interruptible, long deadline) {
        WNode node = null, p;
        for (int spins = -1;;) { // spin while enqueuing
            long m, s, ns;
            // 如果没有持有写锁也没有持有悲观读锁
            if ((m = (s = state) & ABITS) == 0L) {
                // 尝试获取锁
                if ((ns = tryWriteLock(s)) != 0L)
                    return ns;
            }
            else if (spins < 0)
                // spins就是自旋去获取锁的次数
                // SPINS = 1 << 6 = 64
                // WBIT = 128，就是如果当前的状态只有第8位是1
                // 尾节点和头节点都是同一个节点（就代表着时第一次来获取写锁），就让他自旋64次
                spins = (m == WBIT && wtail == whead) ? SPINS : 0;
            else if (spins > 0) {
                // 这里就是让他自旋等待。
                // 这里在一个for循环内表示自旋的写法，非常巧妙，可以借鉴。就是for循环不设置终止和每次见啥哦的田间，在代码逻辑里面去减。
                --spins;
                // 这个是java9才有的方法
                Thread.onSpinWait();
            }
            // 如果没有尾节点
            else if ((p = wtail) == null) { // initialize queue
                // new一个节点设置成头节点，同时头和尾都指向同一个节点。
                WNode hd = new WNode(WMODE, null);
                if (WHEAD.weakCompareAndSet(this, null, hd))
                    wtail = hd;
            }
            else if (node == null)
                // 给当前的node设置为写模式的node
                node = new WNode(WMODE, p);
            else if (node.prev != p)
                // 如果node的前驱节点不是尾节点，就把node的前驱节点设置为尾节点。这里在死循环里面反复设置是因为有可能会有并发的情况。
                node.prev = p;
            // 把尾节点重新设置为新的尾节点，同时原来的尾节点的next设置为新的node
            else if (WTAIL.weakCompareAndSet(this, p, node)) {
                p.next = node;
                break;
            }
        }
        /* 
            上面这个死循环其实就做了自旋的获取锁，如果获取锁失败，就尝试做CLH的入队操作。
        */
    
        boolean wasInterrupted = false;
	    // 一个新的自旋操作
        for (int spins = -1;;) {
            WNode h, np, pp; int ps;
            // 这里的p指的是，新的node的前驱节点，这个判断的目的就是说，前面新的node是不是CLH队列中的第一个排队节点。
            if ((h = whead) == p) {
				// 这里也是也是自旋
                if (spins < 0)
                    // 这个HEAD_SPINS = 1 << 10 = 1024。这个HEAD_SPINS的意思是就是如果你的node已经是头节点下的第一个待操作的节点，就让他自旋次数多一点。
                    spins = HEAD_SPINS;
                // MAX_HEAD_SPINS = 1 << 16 = 65536
                else if (spins < MAX_HEAD_SPINS)
                    // 这个就是如果自旋次数没有超过最大自旋次数，就让自旋次数翻一倍
                    spins <<= 1;
                for (int k = spins; k > 0; --k) { // spin at head
                    long s, ns;
                    // 这个又是标准的StampedLock获取锁的流程，就是看当前有没有写锁和悲观读，如果没有就尝试获取写锁
                    if (((s = state) & ABITS) == 0L) {
                        if ((ns = tryWriteLock(s)) != 0L) {
                            // 如果获取写锁成功，就把头节点设置为新的节点，同时释放原来的头节点
                            // 这里头节点不用CAS的原因是，走到这里已经获取了写锁，其实就是独占锁，所以不存在多线程的竞争问题，所以可以不用CAS的去设置
                            whead = node;
                            node.prev = null;
                            if (wasInterrupted)
                                Thread.currentThread().interrupt();
                            return ns;
                        }
                    }
                    else
                        // 自旋
                        Thread.onSpinWait();
                }
            }
            // 到了这个地方，代表当前节点不是头节点后的第一个待操作节点
            // h != null就是头节点不为空
            else if (h != null) { // help release stale waiters
                WNode c; Thread w;
                // 这个cowait就是，所有的读锁线程都会有一个common节点统一代理，那么这个cowait就是代表这是一个代理所有读锁线程的common节点。这个就是如果头节点的cowait节点不为空，就帮助一起release那些被block的线程
                while ((c = h.cowait) != null) {
                    // 这段代码的意思是，修改头节点的cowait字段，从c改到c的cowait，然后如果这个节点的cowait不为空，就唤醒这个读锁线程。因为这个cowait是一个链表，所以就帮助逐个释放，加快速度。
                    if (WCOWAIT.weakCompareAndSet(h, c, c.cowait) &&
                        (w = c.thread) != null)
                        LockSupport.unpark(w);
                }
            }
            // 如果头节点没有被替换过
            if (whead == h) {
                // 判断node的前驱节点是不是原来的tail节点？这个有点没看懂，难道还能不是嘛。。。
                // 下面这个if判断，反复读源码，都认为是不可能进去的一个判断，应该是为了防止一些极为特殊的情况吧。。一段保护性代码。
                // 这里做的事情就是把node前驱节点的next设置为node
                if ((np = node.prev) != p) {
                    if (np != null)
                        (p = np).next = node;   // stale
                }
                // status有0, WAITING = -1, or CANCELLED = 1
                // 上面那个if跑完就是确保p始终都是node的前驱节点
                else if ((ps = p.status) == 0)
                    WSTATUS.compareAndSet(p, 0, WAITING);
                // 如果已经被取消了
                else if (ps == CANCELLED) {
                    // 就把前驱的前驱设置为node的前驱，等于就是跳过这个被取消的节点
                    if ((pp = p.prev) != null) {
                        node.prev = pp;
                        pp.next = node;
                    }
                }
                else {
                    long time; // 0 argument to park means no timeout
                    // 如果没有超时时间就继续
                    if (deadline == 0L)
                        time = 0L;
                    // 判断是不是超时了
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        return cancelWaiter(node, node, false);
                    Thread wt = Thread.currentThread();
                    node.thread = wt;
                    // 前驱节点的状态是WAITING
                    // && 前驱不是头节点 || 持有了写锁或持有悲观读锁
                    // && 从主内存刷出来的whead和h一样
                    // && node的前驱节点就是p
                    if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                        whead == h && node.prev == p) {
                        // 直接把线程挂起
                        if (time == 0L)
                            LockSupport.park(this);
                        else
                            // 挂起一小段时间
                            LockSupport.parkNanos(this, time);
                    }
                    // 清理thread
                    node.thread = null;
                    if (Thread.interrupted()) {
                        if (interruptible)
                            return cancelWaiter(node, node, true);
                        wasInterrupted = true;
                    }
                }
            }
        }
    }
```

其中如果不满足条件就会尝试去cancelWaiter，该方法具体实现如下

```java
private long cancelWaiter(WNode node, WNode group, boolean interrupted) {
    // 暂时还看不出来什么情况下node或group可能为null
    if (node != null && group != null) {
        Thread w;
        // 把node的状态设置为CANCELLED
        node.status = CANCELLED;
        // unsplice cancelled nodes from group
        // 这个for循环就是，group是那个持有cowait的common节点，把这个链表里面被CANCELLED的节点都去掉。
        for (WNode p = group, q; (q = p.cowait) != null;) {
            if (q.status == CANCELLED) {
                WCOWAIT.compareAndSet(p, q, q.cowait);
                p = group; // restart
            }
            else
                p = q;
        }
        if (group == node) {
            // 悲观读都唤醒，因为当前节点要被cancel了，所以如果这个节点上如果有连着悲观读的线程，都要全部唤醒。
            for (WNode r = group.cowait; r != null; r = r.cowait) {
                if ((w = r.thread) != null)
                    LockSupport.unpark(w); // wake up uncancelled co-waiters
            }
            for (WNode pred = node.prev; pred != null; ) { // unsplice
                WNode succ, pp;        // find valid successor
                
                // 这个while循环的目的是这样的，因为当前node准备cancel掉，所以就找到node的前一个节点，然后把node的后继节点全部连到这个前驱节点上。
                // 后继节点为空或者后继节点被取消了
                while ((succ = node.next) == null ||
                       succ.status == CANCELLED) {
                    WNode q = null;    // find successor the slow way
                    // 这个for循环其实就是从尾节点开始找到在node之后最靠近node的那个没有被取消的节点。
					// 尾节点不为空同时当前node不是尾节点
                    for (WNode t = wtail; t != null && t != node; t = t.prev)
                        // 尾节点没有被取消
                        if (t.status != CANCELLED)
                            q = t;     // don't link if succ cancelled
                    // 后继节点和从尾节点一路查过来的节点对比是否相等
                    if (succ == q ||   // ensure accurate successor
                        WNEXT.compareAndSet(node, succ, succ = q)) {
                        if (succ == null && node == wtail)
                            // 如果node已经是尾节点了，就把尾节点重新设置成node的前驱节点
                            WTAIL.compareAndSet(this, node, pred);
                        break;
                    }
                }
                // 这个的目的就是把node的前驱节点连到后继上，跳开当前的node。
                if (pred.next == node) // unsplice pred link
                    WNEXT.compareAndSet(pred, node, succ);
                // 唤醒后继节点的线程，然后把后继节点的thread设为空？这是什么骚操作？
                if (succ != null && (w = succ.thread) != null) {
                    // wake up succ to observe new pred
                    succ.thread = null;
                    LockSupport.unpark(w);
                }
                // 前驱节点没有被取消 || 前驱节点如果被取消了，就看前驱的前驱。。。
                if (pred.status != CANCELLED || (pp = pred.prev) == null)
                    break;
                node.prev = pp;        // repeat if new pred wrong/cancelled
                WNEXT.compareAndSet(pp, pred, succ);
                pred = pp;
            }
        }
    }
    WNode h; // Possibly release first waiter
    while ((h = whead) != null) {
        long s; WNode q; // similar to release() but check eligibility
        // q就是找到第一个可以被唤醒的节点。
        if ((q = h.next) == null || q.status == CANCELLED) {
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        // head没有变过
        if (h == whead) {
            // q不为空 && 头节点的状态是0
            // && 当前没有写锁
            // && state = 0 为什么要等于0？ || q节点的模式是读模式
            if (q != null && h.status == 0 &&
                ((s = state) & ABITS) != WBIT && // waiter is eligible
                (s == 0L || q.mode == RMODE))
                release(h);
            break;
        }
    }
    return (interrupted || Thread.interrupted()) ? INTERRUPTED : 0L;
}
```

### 读锁获取凭证

```java
@ReservedStackAccess
public long readLock() {
    long s, next;
    // bypass acquireRead on common uncontended case
    // 当前没有排队的线程（应该是为了防止写线程饥饿，ReentrantReadWriteLock里面是看有没有写锁）
    // && 当前读锁数量没有溢出
    // cas尝试去获取读锁，state + 1
    return (whead == wtail
            && ((s = state) & ABITS) < RFULL
            && casState(s, next = s + RUNIT))
        ? next
        : acquireRead(false, 0L);
}

private long acquireRead(boolean interruptible, long deadline) {
    boolean wasInterrupted = false;
    WNode node = null, p;
    for (int spins = -1;;) {
        WNode h;
        // 如果头节点等于尾节点，就是还没有初始化
        if ((h = whead) == (p = wtail)) {
            for (long m, s, ns;;) {
                // 读锁数量没有溢出
                // m就是state的后八位，RFULL = 126
                if ((m = (s = state) & ABITS) < RFULL ?
                    // cas去获取读锁
                    casState(s, ns = s + RUNIT) :
                    // WBIT = 10000000，m < WBIT就是代表当前没有写锁，就去尝试更改读锁的溢出
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                    if (wasInterrupted)
                        Thread.currentThread().interrupt();
                    return ns;
                }
                // 这个就是代表当前已经有了写锁
                else if (m >= WBIT) {
                    // 还是自旋
                    if (spins > 0) {
                        --spins;
                        Thread.onSpinWait();
                    }
                    else {
                        if (spins == 0) {
                            WNode nh = whead, np = wtail;
                            // 头节点和尾节点都没有变化过 || h和p都重新赋值为新的头尾节点，然后判断头尾节点是否一样，如果一样就退出循环，因为代表当前CLH队列为空
                            if ((nh == h && np == p) || (h = nh) != (p = np))
                                break;
                        }
                        // SPINS = 1 << 6 = 64
                        spins = SPINS;
                    }
                }
            }
        }
        // 没有尾节点，初始化整个CLH队列
        if (p == null) { // initialize queue
            // 生成一个写模式的node？？？WTF？？？这个写模式的节点感觉意思是一个阻塞后续节点的感觉。。。
            WNode hd = new WNode(WMODE, null);
            // 设置头节点
            if (WHEAD.weakCompareAndSet(this, null, hd))
                // 设置尾节点
                wtail = hd;
        }
        else if (node == null)
            // 生成一个读模式的节点，前驱节点就是尾节点
            node = new WNode(RMODE, p);
        // 头尾相等或者尾节点的模式不是读，因为尾节点如果是读，就需要连到cowait上，而不是新建一个节点
        else if (h == p || p.mode != RMODE) {
            // 前驱节点设置为尾节点
            if (node.prev != p)
                node.prev = p;
            // 设置尾节点
            else if (WTAIL.weakCompareAndSet(this, p, node)) {
                // 原尾节点的next指向当前node
                p.next = node;
                break;
            }
        }
        // 把p的cowait节点设置为node，同时的node的cowait设置为p.cowait — —#这也太绕了。。。
        else if (!WCOWAIT.compareAndSet(p, node.cowait = p.cowait, node))
            // 如果失败，node.cowait设置为空，因为上面让node.cowait = p.cowait了
            node.cowait = null;
        else {
            for (;;) {
                WNode pp, c; Thread w;
                // 头尾节点不为空
                if ((h = whead) != null && (c = h.cowait) != null &&
                    // 成功的把自己连到前一个节点的链表中
                    WCOWAIT.compareAndSet(h, c, c.cowait) &&
                    (w = c.thread) != null) // help release
                    // 如果连进去就把自己阻塞
                    LockSupport.unpark(w);
                if (Thread.interrupted()) {
                    if (interruptible)
                        return cancelWaiter(node, p, true);
                    wasInterrupted = true;
                }
                if (h == (pp = p.prev) || h == p || pp == null) {
                    long m, s, ns;
                    do {
                        // 没有写锁，同时共享读锁的共享次数没有超过overflow的上限
                        if ((m = (s = state) & ABITS) < RFULL ?
                            // cas获取锁
                            casState(s, ns = s + RUNIT) :
                            // 这里判断没有写锁的目的是什么？上面不是已经判断了，这里是再来一次确保没问题嘛。。。就算这里被并发的获取了写锁，代码也是没有问题的 && overflow修改锁
                            (m < WBIT &&
                             (ns = tryIncReaderOverflow(s)) != 0L)) {
                            if (wasInterrupted)
                                Thread.currentThread().interrupt();
                            return ns;
                        }
                    // 只要没有写锁，就持续重试
                    } while (m < WBIT);
                }
                // 头节点没有变化
                if (whead == h && p.prev == pp) {
                    long time;
                    if (pp == null || h == p || p.status > 0) {
                        node = null; // throw away
                        break;
                    }
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L) {
                        if (wasInterrupted)
                            Thread.currentThread().interrupt();
                        return cancelWaiter(node, p, false);
                    }
                    Thread wt = Thread.currentThread();
                    node.thread = wt;
                    if ((h != pp || (state & ABITS) == WBIT) &&
                        whead == h && p.prev == pp) {
                        if (time == 0L)
                            LockSupport.park(this);
                        else
                            LockSupport.parkNanos(this, time);
                    }
                    node.thread = null;
                }
            }
        }
    }

    // 这里就是队列已经初始化了
    for (int spins = -1;;) {
        WNode h, np, pp; int ps;
        if ((h = whead) == p) {
            if (spins < 0)
                // 这个HEAD_SPINS = 1 << 10 = 1024。这个HEAD_SPINS的意思是就是如果你的node已经是头节点下的第一个待操作的节点，就让他自旋次数多一点。
                spins = HEAD_SPINS;
            // MAX_HEAD_SPINS = 1 << 16 = 65536
            else if (spins < MAX_HEAD_SPINS)
                // 同上，如果不足，就翻一倍继续自旋
                spins <<= 1;
            for (int k = spins;;) { // spin at head
                long m, s, ns;
                // 这段和上面其实基本上是完全一致的
                if ((m = (s = state) & ABITS) < RFULL ?
                    casState(s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                    WNode c; Thread w;
                    whead = node;
                    node.prev = null;
                    while ((c = node.cowait) != null) {
                        if (WCOWAIT.compareAndSet(node, c, c.cowait) &&
                            (w = c.thread) != null)
                            LockSupport.unpark(w);
                    }
                    if (wasInterrupted)
                        Thread.currentThread().interrupt();
                    return ns;
                }
                // 别人拿到了写锁，或者不能自旋了
                else if (m >= WBIT && --k <= 0)
                    break;
                else
                    // 自旋等待
                    Thread.onSpinWait();
            }
        }
        // 这里就是把自己连到cowait上，如果成功连上去了，就阻塞
        else if (h != null) {
            WNode c; Thread w;
            while ((c = h.cowait) != null) {
                if (WCOWAIT.compareAndSet(h, c, c.cowait) &&
                    (w = c.thread) != null)
                    LockSupport.unpark(w);
            }
        }
        if (whead == h) {
            // 这个代码和上面获取写锁的地方是一样的，还是感觉不可能进去的一个逻辑。应该就是一个保护性代码。
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale
            }
            else if ((ps = p.status) == 0)
                // 把前驱节点设置为WAITING
                WSTATUS.compareAndSet(p, 0, WAITING);
            else if (ps == CANCELLED) {
                // 如果前驱节点被Cancel掉了，那么就跳开这个前驱，继续往前找。
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            else {
                long time;
                if (deadline == 0L)
                    time = 0L;
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    // 超时就取消节点
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                node.thread = wt;
                if (p.status < 0 &&
                    (p != h || (state & ABITS) == WBIT) &&
                    whead == h && node.prev == p) {
                    if (time == 0L)
                        LockSupport.park(this);
                    else
                        LockSupport.parkNanos(this, time);
                }
                node.thread = null;
                if (Thread.interrupted()) {
                    if (interruptible)
                        return cancelWaiter(node, node, true);
                    wasInterrupted = true;
                }
            }
        }
    }
}
```



## 锁的释放

------

### 释放写锁

```java
@ReservedStackAccess
public void unlockWrite(long stamp) {
    // 因为写锁是独占锁，如果已经有了写锁，state是被lock不能更改的。
    if (state != stamp || (stamp & WBIT) == 0L)
        throw new IllegalMonitorStateException();
    unlockWriteInternal(stamp);
}

private long unlockWriteInternal(long s) {
    long next; WNode h;
    // volatile的设置值，这个方法和CAS设置state有什么区别？
    STATE.setVolatile(this, next = unlockWriteState(s));
    if ((h = whead) != null && h.status != 0)
        // 唤醒头节点
        release(h);
    return next;
}

private static long unlockWriteState(long s) {
    // WBIT = 128 10000000，因为设置写锁就是让第八位变成1，这里加上WBIT就是让第八位变成0，然后向前进一位.
    // ORIGIN = 256 100000000,初始状态，第9位为1，没有读锁（后7位为0），也没有写锁（第八位为0）
    // 这里s == 0L的意义就是因为int只有32位，如果写锁获取的此时比较多之后，完全是可能溢出32位的，所以如果有一个状态，加上256之后，溢出的部分不算，后32位正好变成0。
    // 但是这个也有一个问题，为什么初始状态一定要一个数字呢？如果初始状态就是0会如何？
    return ((s += WBIT) == 0L) ? ORIGIN : s;
}

private void release(WNode h) {
    if (h != null) {
        WNode q; Thread w;
        // 设置头节点的状态
        WSTATUS.compareAndSet(h, WAITING, 0);
        // 如果头节点的后继节点为空或者被取消了，就从尾节点开始向前找，找到一个WAITING的。
        // INTERRUPTED = 1L; WAITING = -1; CANCELLED = 1;
        if ((q = h.next) == null || q.status == CANCELLED) {
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        // 唤醒
        if (q != null && (w = q.thread) != null)
            LockSupport.unpark(w);
    }
}
```



### 释放读锁

```java
@ReservedStackAccess
public void unlockRead(long stamp) {
    long s, m; WNode h;
	// SBITS = -128 11111111111111111111111110000000
    // (s = state) & SBITS) == (stamp & SBITS) 这个的意思是前25位应该是不会变化的，就是不可能有人能拿到写锁
    while (((s = state) & SBITS) == (stamp & SBITS)
           // RBITS = 111111，就是后七位一定大于零，就说明传入的stamp是有读锁的凭证
           && (stamp & RBITS) > 0L
           // 当前的state也是有读锁的
           && ((m = s & RBITS) > 0L)) {
        // 没有超过上限
        if (m < RFULL) {
            // RUNIT = 1，就是state - 1
            if (casState(s, s - RUNIT)) {
                // 读锁只剩下了一个，头节点存在且不为0，释放头节点后的排队节点
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                return;
            }
        }
        // 减溢出部分
        else if (tryDecReaderOverflow(s) != 0L)
            return;
    }
    throw new IllegalMonitorStateException();
}

private long tryDecReaderOverflow(long s) {
    // 注意这里传进来的s是当前的state
    // ABITS = 11111111，RFULL = 111110
    // 就是看后八位是不是和RFULL相等，
    if ((s & ABITS) == RFULL) {
        // RBITS = 111111
        // 把当前state的后六位都置位1，这个其实也是相当于一个锁，就是为了要修改readerOverflow字段所以加的锁。
        if (casState(s, s | RBITS)) {
            int r; long next;
            // 当前溢出的值 - 1
            if ((r = readerOverflow) > 0) {
                readerOverflow = r - 1;
                next = s;
            }
            else
                next = s - RUNIT;
            // 这个就是相当于释放STATE的锁，这里能用setVolatile是就是因为在这个if判断里面，其实已经拿了锁。
            STATE.setVolatile(this, next);
            return next;
        }
    }
    // 这个地方让出cpu执行权的目的就是方式大量线程频繁自旋，但是大家都失败，所以随机让出执行权，防止类似饥饿的问题。
    else if ((LockSupport.nextSecondarySeed() & OVERFLOW_YIELD_RATE) == 0)
        Thread.yield();
    else
        // 自旋等待一下
        Thread.onSpinWait();
    return 0L;
}
```



##锁的升级

------

### 转换成写锁

1. 先匹配stamp和当前的state是否符合
   - 如果当前已经是写锁，就忽略
   - 如果当前是读锁，就释放读锁，并返回一个写锁stamp
   - 如果当前是乐观读，就返回一个写锁stamp。

```java
public long tryConvertToWriteLock(long stamp) {
    long a = stamp & ABITS, m, s, next;
    // while就是验证stamp是否一致，也就是state的前25位是否一致
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        // 后八位是否为零，就是当前如果不是写锁也没有读锁，进入if的条件就是初始状态或者是乐观读
        if ((m = s & ABITS) == 0L) {
            // 说明stamp是有读锁的，所以就停止
            if (a != 0L)
                break;
            // 尝试获取写锁
            if ((next = tryWriteLock(s)) != 0L)
                return next;
        }
        // 当前已经是写锁了
        else if (m == WBIT) {
            // 这个难道还有可能不一样嘛..
            if (a != m)
                break;
            // 已经是写锁就直接返回
            return stamp;
        }
        // 最后一个读锁
        else if (m == RUNIT && a != 0L) {
            // 直接state - 1 + 写锁
            if (casState(s, next = s - RUNIT + WBIT)) {
                // 内存屏障，storestore
                VarHandle.storeStoreFence();
                return next;
            }
        }
        else
            // 其余情况没法转成写锁。
            break;
    }
    return 0L;
}
```

### 转换成读锁

```java
public long tryConvertToReadLock(long stamp) {
    long a, s, next; WNode h;
    // while就是验证stamp是否一致，也就是state的前25位是否一致
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        // 当前已经有了写锁
        if ((a = stamp & ABITS) >= WBIT) {
            // 写锁版本不一致，无法转换
            if (s != stamp)
                break;
            // 释放写锁，并且读锁数量+1
            STATE.setVolatile(this, next = unlockWriteState(s) + RUNIT);
            if ((h = whead) != null && h.status != 0)
                // 释放排队节点，如果是读锁的话，就可以一起持有读锁
                release(h);
            return next;
        }
        else if (a == 0L) {
            // 走进这里，就代表是乐观读锁转成悲观读锁
            // 获取悲观读锁
            if ((s & ABITS) < RFULL) {
                if (casState(s, next = s + RUNIT))
                    return next;
            }
            // 如果读锁数量溢出，就特殊处理
            else if ((next = tryIncReaderOverflow(s)) != 0L)
                return next;
        }
        else {
            // 已经是一个读锁
            if ((s & ABITS) == 0L)
                // 这个if就是如果读锁已经被释放了，就返回
                break;
            return stamp;
        }
    }
    return 0L;
}
```

### 转换成乐观读

```java
public long tryConvertToOptimisticRead(long stamp) {
    long a, m, s, next; WNode h;
    // 内存屏障
    VarHandle.acquireFence();
    // stamp和当前state依然是匹配的
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        // 给出的stamp是写锁
        if ((a = stamp & ABITS) >= WBIT) {
            // 如果版本不一致，直接返回
            if (s != stamp)
                break;
            return unlockWriteInternal(s);
        }
        else if (a == 0L)
            // 已经是乐观读
            return stamp;
        else if ((m = s & ABITS) == 0L)
            // 当前读锁已经被释放了
            break;
        // 当前的读锁没有超过最大值
        else if (m < RFULL) {
            // 尝试释放一个读锁
            if (casState(s, next = s - RUNIT)) {
                // 判断是否最后一个读锁，如果是的话，就需要唤醒头节点
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                // 返回乐观锁的stamp
                return next & SBITS;
            }
        }
        // 溢出部分-1，然后返回一个乐观锁。
        else if ((next = tryDecReaderOverflow(s)) != 0L)
            return next & SBITS;
    }
    return 0L;
}
```

#读写锁相关总结

------

1. 如果有大量的乐观读场景（比如读的数量是写的几十倍），那么**StampedLock**的性能可以达到**ReentrantReadWriteLock**性能100倍以上。而且乐观锁的性能在各种测试场景下均位居前列

2. 无论如何不要去用公平模式！！！！无论竞争是否激烈，ReentrantReadWriteLock的非公平模式性能都远远好于公平模式（基于AQS实现的其他类同理）。

3. **ReentrantReadWriteLock**在读写不均衡的场景下的性能不及**synchronized**，同时在读和写竞争不那么激烈的时候（比如1读1写，10读10写等）性能也不及**synchronized**，所以java8之后**ReentrantReadWriteLock**应当减少使用

4. 如果需要的是计数功能，应当使用**AtomoicXXXX**类或者**XXXAdder**类，使用读写锁或其他锁的性能不及前两者。但是**StampedLock**的乐观读可以和他们匹敌。

5. 乐观读有实效性。因为乐观读返回的是前25位作为凭证。而一旦写锁溢出之后state的状态会被重置为初始状态。那么在重头开始之后有可能又恰好处于你当时获取乐观锁的那个状态，此时你在验证就有可能验证通过，但是数据已经不是原来的数据（正常情况下不会出现，除非你把stamp存数据库这种。。。所以尽量确保获取了stamp，就尽快验证尽快使用）

6. 自旋操作的一些注意点

   ```java
   // volatile变量必须在死循环开始的地方获取并保存为local变量，内部逻辑必须使用local变量确保逻辑不会错
   for (;;) {
       int s = state;
       // 后续操作各种设计state的运算只能s，不可以直接用state，否则会有并发问题
   }
   ```

7. StampedLock用法比较难用，所以可以和Lamda函数结合

   ==写锁代码封装==

   ```java
   // 写锁代码封装前
   public boolean moveIfAt(double oldX, double oldY, double newX, double newY) {
        long stamp = sl.readLock();
        try {
            while (x == oldX && y == oldY) {
                long writeStamp = sl.tryConvertToWriteLock(stamp);
                if (writeStamp != 0L) {
                    stamp = writeStamp;
                    x = newX; y = newY;
                    return true;
                } else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
   	     return false;
        } finally { sl.unlock(stamp); }
   }
   
   // 写锁代码封装后
   public boolean moveIfAt(double oldX, double oldY, double newX, double newY) {
       return StampedLockIdioms.conditionalWrite(
           sl,
           () -> x == oldX && y == oldY,
           () -> {
               x = newX;
               y = newY;
           }
       ); 
   }
   ```

   ==乐观读代码封装==

   ```java
   // 乐观读封装前
   public double distanceFromOrigin() {
       long stamp = sl.tryOptimisticRead();
       double currentX = x, currentY = y;
       if (!sl.validate(stamp)) {
           stamp = sl.readLock();
           try {
               currentX = x;
               currentY = y;
           } finally {
               sl.unlockRead(stamp);
           }
       }
       return Math.hypot(currentX, currentY);
   }
   
   // 乐观读封装后
   public double distanceFromOrigin() {
       double[] current = new double[2];
       return StampedLockIdioms.optimisticRead(
           sl,
           () -> {
               current[0] = x;
               current[1] = y;
           },
           () -> Math.hypot(current[0], current[1])
       );
   }
   ```


[性能链接参考1]: http://isuru-perera.blogspot.com/2016/05/benchmarking-java-locks-with-counters.html
[性能链接参考2]: https://blog.takipi.com/java-8-stampedlocks-vs-readwritelocks-and-synchronized/
[Doug Lea：Using JDK 9 Memory Order Modes]: http://gee.cs.oswego.edu/dl/html/j9mm.html

