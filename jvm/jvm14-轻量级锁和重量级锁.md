#轻量级锁和重量级锁

当偏向锁获取失败的时候，虚拟机会尝试获取轻量级锁。具体代码如下

```c++
 // traditional lightweight locking
if (!success) {
  // 被锁对象取消被锁的标志
  markOop displaced = lockee->mark()->set_unlocked();
  // 线程上下文的BasicLock设置被锁对象的header
  entry->lock()->set_displaced_header(displaced);
  // 是否使用重量级的锁，默认是false
  bool call_vm = UseHeavyMonitors;
  // 使用重量级锁或被锁对象的头的指针和BasicLock进行CAS交换
  // 这里就是被锁对象的对象头，现在变成一个指针指向BasicLockObject，然后原来的
  if (call_vm || lockee->cas_set_mark((markOop)entry, displaced) != displaced) {
    // Is it simple recursive case?
    if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
      entry->lock()->set_displaced_header(NULL);
    } else {
      // 这里就是monitorenter的实现。再上一章已经分析了fast_enter
      CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
    }
  }
}
```

```c++
if (UseBiasedLocking) {
	// 这里是偏向锁逻辑
  ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
} else {
  // 这里就是重量级锁的逻辑
  ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
}
```

## 轻量级、重量级锁的实现

```c++
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  // 获取对象头
  markOop mark = obj->mark();

  // 如果对象没有被锁，属于自然状态
  if (mark->is_neutral()) {
    // 把当前对象的对象头拷贝一份到lock上
    lock->set_displaced_header(mark);
    // 然后尝试CAS把对象头和lock对象交换（交换后对象头就是lock），如果交换成功，就获取轻量级锁成功
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      return;
    }
	// 当前对象已经被锁住了
  } else if (mark->has_locker() &&
             // 同时当前线程就是锁住该对象的锁持有者
             THREAD->is_lock_owned((address)mark->locker())) {
		// 当前lock对象没用了，因为当前线程上已经有一个lock对象持有了该对象的锁。
    lock->set_displaced_header(NULL);
    // 已经持有了锁直接返回
    return;
  }

	// 到了这里就是轻量级锁获取失败，需要进入锁的膨胀环节
  lock->set_displaced_header(markOopDesc::unused_mark());
  // 注意这里其实是两个方法，第一个方法是inflate，这个方法就是返回一个ObjectMonitor对象
  // 之后是enter方法，就是调用ObjectMonitor的enter方法。
  inflate(THREAD, obj(), inflate_cause_monitor_enter)->enter(THREAD);
}
```

其实到了锁膨胀的环节，就是使用的重量级锁，这块的逻辑比较多，我们一步一步解析。

首先我们要先认识一下**ObjectMonitor**对象

```c++
class ObjectMonitor {
 public:
  enum {
    OM_OK,                    // no error
    OM_SYSTEM_ERROR,          // operating system error
    OM_ILLEGAL_MONITOR_STATE, // IllegalMonitorStateException
    OM_INTERRUPTED,           // Thread.interrupt()
    OM_TIMED_OUT              // Object.wait() timed out
  };

 private:
  volatile markOop   _header;       // 对象头
  void*     volatile _object;       // backward object pointer - strong root
 public:
  ObjectMonitor*     FreeNext;      // Free list linkage
 private:
  DEFINE_PAD_MINUS_SIZE(0, DEFAULT_CACHE_LINE_SIZE,
                        sizeof(volatile markOop) + sizeof(void * volatile) +
                        sizeof(ObjectMonitor *));
 protected:
  void *  volatile _owner;          // 持有锁的线程指针或BasicLock指针
  volatile jlong _previous_owner_tid;  // 上一个持有锁的线程id
  volatile intptr_t  _recursions;   // 重入次数
  ObjectWaiter * volatile _EntryList; // 等待链表，一个线程Thread的链表
 private:
  ObjectWaiter * volatile _cxq;     // 最近刚来等待的线程的链表
  Thread * volatile _succ;          // 可能的下一个持有锁的线程，注意这个_succ一定是在自旋状态
  Thread * volatile _Responsible;

 protected:
  ObjectWaiter * volatile _WaitSet; // 等待的线程链表
  volatile jint  _waiters;          // 等待的数量
 private:
  volatile int _WaitSetLock;        // protects Wait Queue - simple spinlock
```

那么我们看一下具体的锁膨胀的实现，首先是jvm如何获取ObjectMonitor对象。

下面的代码主要分成几个部分

```
for(;;) {
	1. 如果已经有ObjectMonitor，通过对象的指针，找到对应的ObjectMonitor并返回；
	2. 如果没有ObjectMonitor，就看是不是在INFLATING状态中，如果是那就等待；
	3. 如果已经被锁了，就分配一个对象，并尝试翻转对象的状态为INFLATING，如果尝试失败（说明别人在竞争锁）就重试，否则就设置一些状态并返回；
	4. 如果是对象没有被锁的状态，分配一个ObjectMonitor，并尝试做一些设置，如果成功就返回
}
```

下面是代码的具体实现

```c++
ObjectMonitor* ObjectSynchronizer::inflate(Thread * Self,
                                           oop object,
                                           const InflateCause cause) {
  EventJavaMonitorInflate event;

  for (;;) {
    // 获取对象头
    const markOop mark = object->mark();
		// 这里获取的对象头有几种可能
    // *  Inflated     - 锁已经变成重量级锁，可以直接返回
    // *  Stack-locked - 强迫膨胀
    // *  INFLATING    - 别人正在膨胀过程中，等待
    // *  Neutral      - 没有被锁，可以直接进行重量级锁
    // *  BIASED       - 偏向，错误的状态，不应该看到这个状态

    // CASE: inflated
    if (mark->has_monitor()) {
      // 获取对象的ObjectMonitor
      ObjectMonitor * inf = mark->monitor();
      markOop dmw = inf->header();
      assert(dmw->is_neutral(), "invariant: header=" INTPTR_FORMAT, p2i((address)dmw));
      assert(oopDesc::equals((oop) inf->object(), object), "invariant");
      assert(ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
      return inf;
    }

    // CASE: inflation in progress
    if (mark == markOopDesc::INFLATING()) {
      // 这个方法会阻塞该线程，直到锁膨胀结束后进行重试
      ReadStableMark(object);
      continue;
    }

    // CASE: stack-locked
    LogStreamHandle(Trace, monitorinflation) lsh;

    // 已经是被锁的状态了
    if (mark->has_locker()) {
      // 分配一个ObjectMonitor对象，注意这个对象是被缓存起来的，每个线程有一个omFreeList，如果当前线程上的分配不出来，会从全局上进行分配，只有两个都分配不出来，才会新建一个对象，这里的对象是重用的。
      // 这个对象不是在java堆上的，是直接通过os::malloc
      ObjectMonitor * m = omAlloc(Self);
      // 重用对象
      m->Recycle();
      m->_Responsible  = NULL;
      m->_recursions   = 0;
      // 自旋 5000次
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;   // Consider: maintain by type/class

      // 把对象状态设置为膨胀中
      markOop cmp = object->cas_set_mark(markOopDesc::INFLATING(), mark);
      // 如果设置失败
      if (cmp != mark) {
        // 就说明别的线程也在做同样的事情
        // 刚刚分配到的ObjectMonitor释放
        omRelease(Self, m, true);
        // 重试
        continue;
      }

      // 到了这里就是设置成功
      
      markOop dmw = mark->displaced_mark_helper();

      // ObjectMonitor拷贝header
      m->set_header(dmw);

      // 设置锁的的持有者
      m->set_owner(mark->locker());
      m->set_object(object);
      // TODO-FIXME: assert BasicLock->dhw != 0.

      // Must preserve store ordering. The monitor state must
      // be stable at the time of publishing the monitor address.
      object->release_set_mark(markOopDesc::encode(m));

      // 日志
      if (log_is_enabled(Trace, monitorinflation)) {
        ResourceMark rm(Self);
        lsh.print_cr("inflate(has_locker): object=" INTPTR_FORMAT ", mark="
                     INTPTR_FORMAT ", type='%s'", p2i(object),
                     p2i(object->mark()), object->klass()->external_name());
      }
      // 发布一个事件
      if (event.should_commit()) {
        post_monitor_inflate_event(&event, object, cause);
      }
      return m;
    }

    // 没有被锁的自然对象状态
    
    // 先分配一个ObjectMonitor对象
    ObjectMonitor * m = omAlloc(Self);
    // 和上面的操作基本类似，获取到的对象进行重置
    m->Recycle();
    m->set_header(mark);
    m->set_owner(NULL);
    m->set_object(object);
    m->_recursions   = 0;
    m->_Responsible  = NULL;
    m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;       // consider: keep metastats by type/class

    // 尝试设置Header
    if (object->cas_set_mark(markOopDesc::encode(m), mark) != mark) {
      // 如果设置成功
      m->set_header(NULL);
      m->set_object(NULL);
      m->Recycle();
      omRelease(Self, m, true);
      m = NULL;
      continue;
      // interference - the markword changed - just retry.
      // The state-transitions are one-way, so there's no chance of
      // live-lock -- "Inflated" is an absorbing state.
    }

    // 同上，日志+发布时间
    if (log_is_enabled(Trace, monitorinflation)) {
      ResourceMark rm(Self);
      lsh.print_cr("inflate(neutral): object=" INTPTR_FORMAT ", mark="
                   INTPTR_FORMAT ", type='%s'", p2i(object),
                   p2i(object->mark()), object->klass()->external_name());
    }
    if (event.should_commit()) {
      post_monitor_inflate_event(&event, object, cause);
    }
    
    // 返回找到的ObjectMonitor对象
    return m;
  }
}
```

上面这一段我们成功的获取了一个ObjectMonitor对象。那么后面就是enter方法的具体实现。

注意下面这段方法是ObjectMonitor上面的实例方法。

```c++
void ObjectMonitor::enter(TRAPS) {
	// 设置线程
  Thread * const Self = THREAD;

  // cas把ObjectMonitor的_owner字段指向当前线程
  void * cur = Atomic::cmpxchg(Self, &_owner, (void*)NULL);
  // 不是很清楚什么情况会返回null
  if (cur == NULL) {
    // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
    assert(_recursions == 0, "invariant");
    assert(_owner == Self, "invariant");
    return;
  }

  // 如果持有锁的线程已经是当前线程了，那就重入次数+1
  if (cur == Self) {
    // TODO-FIXME: check for integer overflow!  BUGID 6557169.
    _recursions++;
    return;
  }

  // 如果当前线程已经持有了这个锁
  if (Self->is_lock_owned ((address)cur)) {
    assert(_recursions == 0, "internal state error");
    // 重入次数为1
    _recursions = 1;
		// _owner设置为当前线程，不知道为什么要再多设置一次
    _owner = Self;
    return;
  }
	
  // 不知道这个干什么的。。暂存一下？
  Self->_Stalled = intptr_t(this);

  // Try one round of spinning *before* enqueueing Self
  // and before going through the awkward and expensive state
  // transitions.  The following spin is strictly optional ...
  // Note that if we acquire the monitor from an initial spin
  // we forgo posting JVMTI events and firing DTRACE probes.
  
  // 尝试自旋，这个TrySpin方法的实现也非常长，我们先忽略
  // 如果if判断条件成立，就说通过自旋获取到了锁
  if (TrySpin(Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_recursions == 0, "invariant");
    assert(((oop)(object()))->mark() == markOopDesc::encode(this), "invariant");
    Self->_Stalled = 0;
    return;
  }

  // 变成一个JavaThread，现在由于获取锁失败，所以要进入排队的流程
  JavaThread * jt = (JavaThread *) Self;

  // 排队的数量+1
  Atomic::inc(&_count);

  // 发布事件
  JFR_ONLY(JfrConditionalFlushWithStacktrace<EventJavaMonitorEnter> flush(jt);)
  EventJavaMonitorEnter event;
  if (event.should_commit()) {
    event.set_monitorClass(((oop)this->object())->klass());
    event.set_address((uintptr_t)(this->object_addr()));
  }

  { 
    // 这个是类似一个监控打点
    JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);

    // 给当前线程设置被阻塞的Monitor对象
    Self->set_current_pending_monitor(this);

    // 发布事件
    DTRACE_MONITOR_PROBE(contended__enter, this, object(), jt);
    if (JvmtiExport::should_post_monitor_contended_enter()) {
      JvmtiExport::post_monitor_contended_enter(jt, this);

      // The current thread does not yet own the monitor and does not
      // yet appear on any queues that would get it made the successor.
      // This means that the JVMTI_EVENT_MONITOR_CONTENDED_ENTER event
      // handler cannot accidentally consume an unpark() meant for the
      // ParkEvent associated with this ObjectMonitor.
    }

    // 翻转osThread的状态
    OSThreadContendState osts(Self->osthread());
    // 翻转JavaThread线程的状态
    ThreadBlockInVM tbivm(jt);

    for (;;) {
      jt->set_suspend_equivalent();
			
      // 尝试获取锁
      EnterI(THREAD);

      // 如果线程没有被其他人挂起就结束死循环
      if (!ExitSuspendEquivalent(jt)) break;

      // 到这里就是我们已经获取了锁，但是我们的线程被别人挂起了，所以我们就要放弃获取到的锁
      _recursions = 0;
      _succ = NULL;
      exit(false, Self);

      jt->java_suspend_self();
    }
    // 到了这里就是当前线程已经获取到了锁，并且可以执行
    Self->set_current_pending_monitor(NULL);

    // We cleared the pending monitor info since we've just gotten past
    // the enter-check-for-suspend dance and we now own the monitor free
    // and clear, i.e., it is no longer pending. The ThreadBlockInVM
    // destructor can go to a safepoint at the end of this block. If we
    // do a thread dump during that safepoint, then this thread will show
    // as having "-locked" the monitor, but the OS and java.lang.Thread
    // states will still report that the thread is blocked trying to
    // acquire it.
  }

  // 因为是已经获取到了锁并且可以执行代码，所以count - 1
  Atomic::dec(&_count);
	
  // 这个东西置为0，不知道有啥用
  Self->_Stalled = 0;

  // 做一些事件的publish
  DTRACE_MONITOR_PROBE(contended__entered, this, object(), jt);
  if (JvmtiExport::should_post_monitor_contended_entered()) {
    JvmtiExport::post_monitor_contended_entered(jt, this);

    // The current thread already owns the monitor and is not going to
    // call park() for the remainder of the monitor enter protocol. So
    // it doesn't matter if the JVMTI_EVENT_MONITOR_CONTENDED_ENTERED
    // event handler consumed an unpark() issued by the thread that
    // just exited the monitor.
  }
  if (event.should_commit()) {
    event.set_previousOwner((uintptr_t)_previous_owner_tid);
    event.commit();
  }
  OM_PERFDATA_OP(ContendedLockAttempts, inc());
}
```

上面是完整的获取锁的流程，但是其中有两个比较关键的流程没有分析，一个是SpinLock的具体原理，我们放到后面的自旋锁单独分析，所以下面就单独分析一下EnterI方法。

```c++
void ObjectMonitor::EnterI(TRAPS) {
  Thread * const Self = THREAD;

  // 先尝试获取一次锁
  if (TryLock (Self) > 0) {
    // 如果获取成功就返回
    return;
  }

  // 再尝试自旋获取锁
  if (TrySpin(Self) > 0) {
    // 如果获取成功就返回
    return;
  }

  // The Spin failed -- Enqueue and park the thread ...
  // 自旋锁失败，尝试把自己添加到锁队列并且挂起该线程
  // 把当前线程添加到ObjectMonitor的_cxq链表中
  
  // 把当前线程封装成一个ObjectWaiter对象
  ObjectWaiter node(Self);
  Self->_ParkEvent->reset();
  node._prev   = (ObjectWaiter *) 0xBAD;
  node.TState  = ObjectWaiter::TS_CXQ;

  ObjectWaiter * nxt;
  for (;;) {
    // 链表入队
    node._next = nxt = _cxq;
    // 如果入队成功就返回
    if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;

    // 如果入队不成功就再尝试一次获取锁
    if (TryLock (Self) > 0) {
      // 成功就返回
      return;
    }
  }

  // _Responsible的作用就是获取不到锁的线程会被挂起，但是为了防止所有等待锁的线程都被挂起所以会有一个线程是会被定时唤醒来防止锁已经被释放但是其他阻塞的线程没有人可以持有锁继续执行。这个被定时唤醒的线程就是_Responsible
  if (nxt == NULL && _EntryList == NULL) {
    // Try to assume the role of responsible thread for the monitor.
    // CONSIDER:  ST vs CAS vs { if (Responsible==null) Responsible=Self }
    Atomic::replace_if_null(Self, &_Responsible);
  }

  int nWakeups = 0;
  int recheckInterval = 1;

  // 到了这里其实线程已经入队了
  for (;;) {

    // 再尝试一下是否可以获取到锁
    if (TryLock(Self) > 0) break;

    // 获取不到就把自己挂起
    
    // 如果自己是_Responsible线程，就挂起一段时间
    if (_Responsible == Self) {
      Self->_ParkEvent->park((jlong) recheckInterval);
      // Increase the recheckInterval, but clamp the value.
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      // 否则就被挂死
      Self->_ParkEvent->park();
    }

    //到这里就是被唤醒了继续执行，唤醒就尝试获取锁，如果获取到就退出死循环
    if (TryLock(Self) > 0) break;

		// 到这里就是被唤醒但是还是没有抢到锁（这个是很可能的，因为很有可能上一个持有锁的线程在唤醒队列线程的过程中，其他新进来的线程已经获取到了锁）
    OM_PERFDATA_OP(FutileWakeups, inc());
    // 被叫醒的次数+1
    ++nWakeups;

		// 自旋再次尝试
    if (TrySpin(Self) > 0) break;

    // We can find that we were unpark()ed and redesignated _succ while
    // we were spinning.  That's harmless.  If we iterate and call park(),
    // park() will consume the event and return immediately and we'll
    // just spin again.  This pattern can repeat, leaving _succ to simply
    // spin on a CPU.

    // 因为_succ是自旋的，所以如果是自己就没必要了，因为继续跑下去自己是会被挂起来了，站着_succ的位置就不合适了
    if (_succ == Self) _succ = NULL;

    // 内存屏障
    OrderAccess::fence();
  }

  // 到了这里就是我已经获取到了锁
  // 获取到了锁，所以要看看等待队列里面是不是有自己，如果有的话把自己从队列中删掉
  UnlinkAfterAcquire(Self, &node);
  // 如果是_succ也移除，因为自己已经获取了锁
  if (_succ == Self) _succ = NULL;

	// 如果自己是_Responsible线程，就清空
  if (_Responsible == Self) {
    _Responsible = NULL;
    // 内存屏障
    OrderAccess::fence(); // Dekker pivot-point
  }

  return;
}
```

所以上面的流程其实总结一下也非常好理解，先尝试直接获取锁，获取不到就是自旋尝试，还是获取不到就把自己放到等待队列里面，放队列的过程还是会不停的尝试。如果确实获取不到就把自己挂起。同时还有一个_Responsible线程防止所有的等待线程都没有被唤醒（不是很清楚什么情况会出现所有的等待线程都不被唤醒，可能只是一个保护措施）。

上面的整个流程中还有一个非常重要的TryLock，这个方法出现的频率非常高，不过实现非常简单。

```c++
int ObjectMonitor::TryLock(Thread * Self) {
  void * own = _owner;
  if (own != NULL) return 0;
  // 所谓TryLock就是把ObjectMonitor的_owner通过cas设置为当前线程
  if (Atomic::replace_if_null(Self, &_owner)) {
    // Either guarantee _recursions == 0 or set _recursions = 0.
    assert(_recursions == 0, "invariant");
    assert(_owner == Self, "invariant");
    return 1;
  }
  return -1;
}
```

## 释放锁

通过上面的流程我们已经获取了锁或者被放到了重量级锁的队列中。

当一个持有锁的线程退出同步代码块之后，一方面它需要释放自己持有的锁，另一方面它需要唤醒后面排队的线程。所以我们看一下具体释放锁的实现是如何的。

释放锁的字节码是monitorexit，我们看一下这个字节码对应的虚拟机实现。

```c++
//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))
  // 获取对象
  Handle h_obj(thread, elem->obj());
	// 如果这个对象不存在或并没有被锁住就抛出异常
	// 一般我理解只有自己通过一些字节码生成的工具错误的拼接了字节码才会进入到这个if条件里面。
  if (elem == NULL || h_obj()->is_unlocked()) {
    THROW(vmSymbols::java_lang_IllegalMonitorStateException());
  }
	// 释放
  ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
	// 清空对象
  elem->set_obj(NULL);
IRT_END
  
// 上面的slow_exit其实也是调用的fast_exit
void ObjectSynchronizer::slow_exit(oop object, BasicLock* lock, TRAPS) {
  fast_exit(object, lock, THREAD);
}
```

所以我们来重点分析一下fast_exit这个方法的具体实现。

```c++
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
	// 获取对象头
  markOop mark = object->mark();

  // 获取锁上的header
  markOop dhw = lock->displaced_header();
  // 如果这个header为空，说明这个锁的释放是一个重复的操作，所以啥都不用干
  if (dhw == NULL) {
    // 如果不是膨胀中
    if (mark != markOopDesc::INFLATING()) {
      // 省略一些assert
      
      // 如果对象头上是有monitor的
      if (mark->has_monitor()) {
        // 只有一种特殊情况会到这个if里面
        // 就是一个锁在持有的过程中被膨胀为了ObjectMonitor的重量级锁，所以这里也做一些断言即可
        ObjectMonitor * m = mark->monitor();
        // 做一些断言
        assert(((oop)(m->object()))->mark() == mark, "invariant");
        assert(m->is_entered(THREAD), "invariant");
      }
    }
    return;
  }

  // 如果对象时轻量级锁状态，就把对象头给交换回来即可
  if (mark == (markOop) lock) {
    if (object->cas_set_mark(dhw, mark) == mark) {
      // 交换成功就返回
      return;
    }
  }

  // 这里还是一样的，通过inflate方法找到对应的ObjectMonitor，找的方式和上面相同，所以我们来看一下后面的exit的具体实现就可以了。
  inflate(THREAD, object, inflate_cause_vm_internal)->exit(true, THREAD);
}
```

```c++
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  Thread * const Self = THREAD;
  // 因为owner可能是线程也可能是BasicLock，所以判断一下
  if (THREAD != _owner) {
    // 如果_owner不是线程那就是BasicLock，所以当前线程要拥有BasicLock
    if (THREAD->is_lock_owned((address) _owner)) {
      // 把_owner设置为线程
      _owner = THREAD;
      // 重入变成0？为什么要变成0。。。
      _recursions = 0;
    } else {
      // 没锁的情况
      assert(false, "Non-balanced monitor enter/exit! Likely JNI locking");
      return;
    }
  }

  // 如果重入次数不为零，重入次数-1，
  if (_recursions != 0) {
    _recursions--;
    // 直接返回即可
    return;
  }

  // _Responsible清空
  _Responsible = NULL;

  for (;;) {
    // 内存屏障清空_owner
    OrderAccess::release_store(&_owner, (void*)NULL);   // drop the lock
    // 内存屏障
    OrderAccess::storeload();                        // See if we need to wake a successor
    // 如果后面没有等待的线程，就直接返回
    if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
      return;
    }

    // 这里尝试把自己设置为_owner
    if (!Atomic::replace_if_null(THREAD, &_owner)) {
      // 如果设置不上去就说_succ的自旋线程（或者其他新来的）已经抢占了锁，直接返回即可
      return;
    }

    ObjectWaiter * w = NULL;

    // 这个队列是排队获取锁的队列
    w = _EntryList;
    // 如果有排队的_EntryList
    if (w != NULL) {
      // 如果不为空就唤醒，唤醒后直接结束
      ExitEpilog(Self, w);
      return;
    }

    // 这个_cxq队列是，最近为了获取锁没获取到排队的队列
    // 然后看_cxq队列
    w = _cxq;
    // 如果为空就从循环开始重试
    if (w == NULL) continue;

    // 这个死循环没看懂。。感觉是确保(w = _cxq) != NULL
    for (;;) {
      assert(w != NULL, "Invariant");
      ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w);
      if (u == w) break;
      w = u;
    }

    // 把_cxq队列给连到_EntryList
    _EntryList = w;
    ObjectWaiter * q = NULL;
    ObjectWaiter * p;
    for (p = w; p != NULL; p = p->_next) {
      guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
      p->TState = ObjectWaiter::TS_ENTER;
      p->_prev = q;
      q = p;
    }

		// 如果_succ已经不为空就继续，因为_succ是一个自旋的线程
    if (_succ != NULL) continue;

    // 因为_cxq已经连到_EntryList上了，所以唤醒_EntryList里排队的线程。
    w = _EntryList;
    if (w != NULL) {
      guarantee(w->TState == ObjectWaiter::TS_ENTER, "invariant");
      // 唤醒就离开
      ExitEpilog(Self, w);
      return;
    }
  }
}
```

------

## 瞎编

总结一下，重量级锁大致一下五个角色，涉及两个队列，和三个线程

- \_owner：皇帝线程，当前持有锁的线程。

  皇帝回持有锁并运行，当皇帝需要让出自己的

- \_succ：太子线程，这个太子会一直自旋的状态试图获取皇位

- \_EntryList：皇位候选人队列。

  这些候选人是 从\_cxq中晋升上来的，皇帝如果无法传递皇位会主动把\_cxq的备选皇子晋升到\_EntryList中

- \_cxq：备选皇子队列。

  _cxq和\_EntryList要分开的目的是因为如果只用一个队列，这个队列可能被各种线程疯狂CAS，所以使用两个队列，可以降低一些冲突。

- 其他新出生的皇子（新来的线程）：新来的会直接争夺皇位，如果争夺不到才回去自旋排队，自旋排队过程中还会不停的尝试争夺皇位（自旋过程中会尝试争夺太子的位置）。