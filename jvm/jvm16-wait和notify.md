#jvm16-wait和notify

在jdk5之前，所有的锁的等待都需要通过Object.wait()和Object.notify()、Object.notifyAll()来实现。所以在分析锁的过程中，研究这块是如何实现的也非常有必要。

```java
// 首先这个方法是一个native的方法
public final native void wait(long timeoutMillis) throws InterruptedException;
```

```c
// Object.c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};
```

所有jdk中的native方法都有一个同名的c文件对应实现了相关的方法。可以看到相关方法对应的就是JVM_MonitorWait、JVM_MonitorNotify、JVM_MonitorNotifyAll。

## wait

```c++
JVM_ENTRY(void, JVM_MonitorWait(JNIEnv* env, jobject handle, jlong ms))
  JVMWrapper("JVM_MonitorWait");
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  JavaThreadInObjectWaitState jtiows(thread, ms != 0);
  if (JvmtiExport::should_post_monitor_wait()) {
    JvmtiExport::post_monitor_wait((JavaThread *)THREAD, (oop)obj(), ms);

    // The current thread already owns the monitor and it has not yet
    // been added to the wait queue so the current thread cannot be
    // made the successor. This means that the JVMTI_EVENT_MONITOR_WAIT
    // event handler cannot accidentally consume an unpark() meant for
    // the ParkEvent associated with this ObjectMonitor.
  }
	// 这个就是wait的实现
  ObjectSynchronizer::wait(obj, ms, CHECK);
JVM_END
```

那么对应的wait的实现就是ObjectSynchronizer中的方法。

```c++
int ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  // 如果使用了偏向锁，那么必须撤销偏向锁，因为wait方法必须是重量级锁，因为偏向锁没有ObjectMonitor这个对象，而wait等必须有waitSet这样的数据结构去存储有多少线程目前正阻塞在上面
  if (UseBiasedLocking) {
    // 这个就是之前分析的撤销和重偏向，不过这里的参数是false，意思就是撤销后不需要重偏向
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  // 这个inflate方法之前也分析过，作用就是找到或分配一个ObjectMonitor对象
  ObjectMonitor* monitor = inflate(THREAD, obj(), inflate_cause_wait);

  // 监控探针信息收集
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  // 核心实现
  monitor->wait(millis, true, THREAD);

	// 监控探针
  return dtrace_waited_probe(monitor, obj, THREAD);
}
```

所以我们重点应该来关注monitor上的wait方法的实现。

wait的完整实现，大致流程如下

- 生成一个ObjectWaiter，以链表的形式链接到_WaitSet上
- 释放锁
- 把自己挂起
- 被唤醒后，通过常规的争抢锁的逻辑，如果争抢不到就按照_succ这样的进行排队（基本同monitorenter字节码）

```c++
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
  Thread * const Self = THREAD;
  // 找到java线程
  JavaThread *jt = (JavaThread *)THREAD;

  // 确认是当前线程池有了锁，并且没有要抛出异常(pending exception)
  CHECK_OWNER();

  EventJavaMonitorWait event;

  // 如果线程被interrupted，就抛出异常Interrupted异常
  if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
    // post monitor waited event.  Note that this is past-tense, we are done waiting.
    if (JvmtiExport::should_post_monitor_waited()) {
      // Note: 'false' parameter is passed here because the
      // wait was not timed out due to thread interrupt.
      JvmtiExport::post_monitor_waited(jt, this, false);

      // In this short circuit of the monitor wait protocol, the
      // current thread never drops ownership of the monitor and
      // never gets added to the wait queue so the current thread
      // cannot be made the successor. This means that the
      // JVMTI_EVENT_MONITOR_WAITED event handler cannot accidentally
      // consume an unpark() meant for the ParkEvent associated with
      // this ObjectMonitor.
    }
    if (event.should_commit()) {
      post_monitor_wait_event(&event, this, 0, millis, false);
    }
    THROW(vmSymbols::java_lang_InterruptedException());
    return;
  }

	// _Stalled暂存一下自己这个对象的指针，不知道干嘛用的，存了又不用
  Self->_Stalled = intptr_t(this);
  jt->set_current_waiting_monitor(this);

  // 创建一个可以被放到waitSet里的对象ObjectWaiter
  ObjectWaiter node(Self);
  // node的状态更换一下
  node.TState = ObjectWaiter::TS_WAIT;
  // 和当前线程相关的Parkevent重置
  // 这个就是如果线程要挂起，那么必须挂起在一个对象上，这个parkEvent就是该线程内部挂起在这个event对象上
  Self->_ParkEvent->reset();
  // 内存屏障
  OrderAccess::fence();          // ST into Event; membar ; LD interrupted-flag

  // 通过自旋获取_WaitSetLock
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - add");
  // 把自己链接到链表上
  AddWaiter(&node);
  // 释放_WaitSetLock锁
  Thread::SpinRelease(&_WaitSetLock);

  // _Responsible这个就是当线程被挂起为了防止所有的线程都没有被唤醒，所以会有一个_Responsible线程定时的被唤醒起来检查一下
  _Responsible = NULL;

  intptr_t save = _recursions; // record the old recursion count
  _waiters++;                  // increment the number of waiters
  _recursions = 0;             // set the recursion level to be 1
  // 当前线程退出当前的ObjectMonitor
  // 言外之意就是释放当前的锁
  exit(true, Self);                    // exit the monitor
  guarantee(_owner != Self, "invariant");

  // The thread is on the WaitSet list - now park() it.
  // On MP systems it's conceivable that a brief spin before we park
  // could be profitable.
  //
  // TODO-FIXME: change the following logic to a loop of the form
  //   while (!timeout && !interrupted && _notified == 0) park()

  int ret = OS_OK;
  int WasNotified = 0;
  {
    // 获取操作系统的线程对象
    OSThread* osthread = Self->osthread();
    OSThreadWaitState osts(osthread, true);
    {
      // 线程的状态转换
      ThreadBlockInVM tbivm(jt);
      // Thread is in thread_blocked state and oop access is unsafe.
      jt->set_suspend_equivalent();

      if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
        // Intentionally empty
      } else if (node._notified == 0) {
        // park就是挂起
        if (millis <= 0) {
          Self->_ParkEvent->park();
        } else {
          // 这个是挂起一段时间
          ret = Self->_ParkEvent->park(millis);
        }
      }

      // were we externally suspended while we were waiting?
      if (ExitSuspendEquivalent (jt)) {
        // TODO-FIXME: add -- if succ == Self then succ = null.
        jt->java_suspend_self();
      }

    } // Exit thread safepoint: transition _thread_blocked -> _thread_in_vm

    // 到了这里就说线程已经从被挂起的状态恢复了，所以这里要做的就是把node的状态设置为执行中，并且把node从waitSet的队列中移除
    if (node.TState == ObjectWaiter::TS_WAIT) {
      Thread::SpinAcquire(&_WaitSetLock, "WaitSet - unlink");
      if (node.TState == ObjectWaiter::TS_WAIT) {
        DequeueSpecificWaiter(&node);       // unlink from WaitSet
        node.TState = ObjectWaiter::TS_RUN;
      }
      Thread::SpinRelease(&_WaitSetLock);
    }

    // 内存屏障
    OrderAccess::loadload();
    // 如果自己本来要做succ的轮询线程，就只为NULL，因为自己已经可以执行了
    if (_succ == Self) _succ = NULL;
    // 看是不是被notify的
    WasNotified = node._notified;

    // post monitor waited event. Note that this is past-tense, we are done waiting.
    if (JvmtiExport::should_post_monitor_waited()) {
      JvmtiExport::post_monitor_waited(jt, this, ret == OS_TIMEOUT);

      if (node._notified != 0 && _succ == Self) {
        node._event->unpark();
      }
    }

    // 发布一个事件
    if (event.should_commit()) {
      post_monitor_wait_event(&event, this, node._notifier_tid, millis, ret == OS_TIMEOUT);
    }

    OrderAccess::fence();

		// 清空_Stalled
    Self->_Stalled = 0;
		
    // 这里就是尝试再次获取锁
    // 主要wait被唤醒之后也是要再去争取锁的，如果争取不到锁，就按照普通的顺序也要进入队列中排队等待执行
    ObjectWaiter::TStates v = node.TState;
    if (v == ObjectWaiter::TS_RUN) {
      enter(Self);
    } else {
			// 这里的ReenterI方法其实也是死循环去TryLock和TrySpin，如果获取不到就把自己park挂起
      ReenterI(Self, &node);
      node.wait_reenter_end(this);
    }
		// 方法到了这里就说明已经获取到了锁

  } // OSThreadWaitState()

  jt->set_current_waiting_monitor(NULL);

  _recursions = save;     // restore the old recursion count
  _waiters--;             // decrement the number of waiters

  // check if the notification happened
  if (!WasNotified) {
    // no, it could be timeout or Thread.interrupt() or both
    // check for interrupt event, otherwise it is timeout
    if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
      THROW(vmSymbols::java_lang_InterruptedException());
    }
  }
}
```

##notify和notifyAll

上面是让线程等待，下面是如何唤醒，因为这两个方法的实现几乎一模一样就放到一起了。

```c++
void ObjectSynchronizer::notify(Handle obj, TRAPS) {
  // 偏向锁的话就撤销偏向锁
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  markOop mark = obj->mark();
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    return;
  }
  // 找到ObjectMonitor，然后执行notify方法
  inflate(THREAD, obj(), inflate_cause_notify)->notify(THREAD);
}

void ObjectSynchronizer::notifyall(Handle obj, TRAPS) {
  // 如果是偏向锁就撤销偏向锁
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  markOop mark = obj->mark();
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    return;
  }
  // 找到ObjectMonitor，然后执行notifyAll方法
  inflate(THREAD, obj(), inflate_cause_notify)->notifyAll(THREAD);
}
```

那么重点来分析一下ObjectMonitor的notify和notifyAll方法。

```c++
void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
    return;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);
  INotify(THREAD);
  OM_PERFDATA_OP(Notifications, inc(1));
}

void ObjectMonitor::notifyAll(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
    return;
  }

  DTRACE_MONITOR_PROBE(notifyAll, this, object(), THREAD);
  int tally = 0;
  while (_WaitSet != NULL) {
    tally++;
    INotify(THREAD);
  }

  OM_PERFDATA_OP(Notifications, inc(tally));
}
```

那么可以看到这两个方法的差别就是，notify只调用了一次INotify方法， 而notifyAll就是把所有的_WaitSet所有等待的ObjectWaiter的node全部唤醒。所以关键就是INotify的实现。

```c++
void ObjectMonitor::INotify(Thread * Self) {
  // 获取_WaitSet的锁
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - notify");
  
  // 从WaitSet的队列中获取头节点
  ObjectWaiter * iterator = DequeueWaiter();
  if (iterator != NULL) {

    // 翻转状态
    iterator->TState = ObjectWaiter::TS_ENTER;

    // 表明该节点是被唤醒的
    iterator->_notified = 1;
    // 唤醒的线程ID设置一下
    iterator->_notifier_tid = JFR_THREAD_ID(Self);

    // 看一下_EntryList是不是为空
    ObjectWaiter * list = _EntryList;
    if (list != NULL) {
      assert(list->_prev == NULL, "invariant");
      assert(list->TState == ObjectWaiter::TS_ENTER, "invariant");
      assert(list != iterator, "invariant");
    }

    // 把_WaitSet里面取出来的那个节点给链接到_EntryList上或者_cxq上
    // 具体就是看情况，如果没有_EntryList，就放到EntryList上，否则就放到_cxq上
    if (list == NULL) {
      iterator->_next = iterator->_prev = NULL;
      _EntryList = iterator;
    } else {
      iterator->TState = ObjectWaiter::TS_CXQ;
      for (;;) {
        ObjectWaiter * front = _cxq;
        iterator->_next = front;
        if (Atomic::cmpxchg(iterator, &_cxq, front) == front) {
          break;
        }
      }
    }

    iterator->wait_reenter_begin(this);
  }
  // 释放锁
  Thread::SpinRelease(&_WaitSetLock);
}
```

根据具体的实现其实可以看到，所谓的notify其实就是把等待的ObjectWaiter从\_WaitSet中放到\_EntryList或者\_cxq上，然后等到主线程的代码退出syncronized同步块，自然而然的就会唤醒后面排队的线程。

> 这也是为什么wait和notify、notifyAll一定要在同步块中执行的一个原因