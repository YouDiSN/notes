# jvm15-自旋锁

自旋锁其实是一个优化，实现可以非常简单，一个for循环即可。但是虚拟机中对于自旋锁做了一些自适应的工作，所以导致自旋锁的相关代码也非常长，因此单独分析自旋锁的相关逻辑。

```c++
int ObjectMonitor::TrySpin(Thread * Self) {
  // 这个Knob_FixedSpin一直都是0，所以这段代码我理解是一个历史代码，下面的这个if条件拥有也进不去
	// 获取自旋的次数
  int ctr = Knob_FixedSpin;
  if (ctr != 0) {
    // 不断的自旋获取锁
    while (--ctr >= 0) {
      // 这个TryLock的实现就是通过CAS把ObjectMonitor的_owner设置为当前线程
      if (TryLock(Self) > 0) return 1;
      // 如果获取不到就稍微休息一下，这个方法不同的操作系统+硬件架构是不同的实现，但是基本上就是return一个数字
      SpinPause();
    }
    return 0;
  }

  // 上面的这段代码进不去，其实真正的逻辑都是从这里开始的。
  // Knob_PreSpin = 10
  for (ctr = Knob_PreSpin + 1; --ctr >= 0;) {
    if (TryLock(Self) > 0) {
      // 获取到了锁
      int x = _SpinDuration;
      // Knob_SpinLimit = 5000
      if (x < Knob_SpinLimit) {
        // Knob_Poverty = 1000，这个的意思是x最小也是1000
        if (x < Knob_Poverty) x = Knob_Poverty;
        // Knob_BonusB = 100
        // 意思就是在PreSpin阶段获取到了锁，自旋次数就+100
        _SpinDuration = x + Knob_BonusB;
      }
      return 1;
    }
    // 没有自旋到，就休息一下
    SpinPause();
  }

	// 这里就是PreSpin阶段没有获取到锁

  // 获取_SpinDuration，如果小于0，直接返回
  ctr = _SpinDuration;
  if (ctr <= 0) return 0;

  // 如果_owner线程不再Runnable状态
  if (NotRunnable(Self, (Thread *) _owner)) {
    return 0;
  }

  // 如果_succ没有，就把自己放上去
  if (_succ == NULL) {
    _succ = Self;
  }
  Thread * prv = NULL;

	// ctr = _SpinDuration
  while (--ctr >= 0) {

    // 0xFF是255，实际上就是每自旋255下看一下是不是在等待进入安全点
    // 因为有可能在自旋的过程中会触发一次GC，这个时候会要求所有的线程把自己挂起。如果这里一直在自旋就会浪费进入安全点的时间
    if ((ctr & 0xFF) == 0) {
      // 如果需要挂起
      if (SafepointMechanism::should_block(Self)) {
        // 直接运行Abort代码块，执行退出的部分
        goto Abort;
      }
      // 休息一下
      SpinPause();
    }

    // Probe _owner with TATAS
    // If this thread observes the monitor transition or flicker
    // from locked to unlocked to locked, then the odds that this
    // thread will acquire the lock in this spin attempt go down
    // considerably.  The same argument applies if the CAS fails
    // or if we observe _owner change from one non-null value to
    // another non-null value.   In such cases we might abort
    // the spin without prejudice or apply a "penalty" to the
    // spin count-down variable "ctr", reducing it by 100, say.

    // 拿到锁的持有线程
    Thread * ox = (Thread *) _owner;
    if (ox == NULL) {
      // 尝试把自己变成Owner
      ox = (Thread*)Atomic::cmpxchg(Self, &_owner, (void*)NULL);
      if (ox == NULL) {
				// 如果CAS成功
        
        // 如果自己是_succ，就清空，因为自己已经获取到锁了
        if (_succ == Self) {
          _succ = NULL;
        }

				// 由于通过自旋获取到了锁，所以增加下次自旋的次数
        int x = _SpinDuration;
        // Knob_SpinLimit = 5000
        if (x < Knob_SpinLimit) {
          // 这里的逻辑和上面一致
          if (x < Knob_Poverty) x = Knob_Poverty;
          _SpinDuration = x + Knob_Bonus;
        }
        return 1;
      }

			// 自旋失败
      prv = ox;
      // 进入退出的逻辑
      goto Abort;
    }

    // 在自旋的过程中锁的持有者已经改变过了
    // 进入退出逻辑
    if (ox != prv && prv != NULL) {
      goto Abort;
    }
    prv = ox;

    // 如果当前线程不在Runnable状态
    if (NotRunnable(Self, ox)) {
      goto Abort;
    }
    // 把自己设置为_succ
    if (_succ == NULL) {
      _succ = Self;
    }
  }

	// 自旋失败了，降低_SpinDuration的值
  {
    int x = _SpinDuration;
    if (x > 0) {
      // Knob_Penalty = 200
      x -= Knob_Penalty;
      // 不停的减少，一次减200，要是低于0了就变成0
      if (x < 0) x = 0;
      _SpinDuration = x;
    }
  }

  // 退出逻辑
 Abort:
  if (_succ == Self) {
    // 如果自己是_succ就清理一下
    _succ = NULL;
		// 内存屏障
    OrderAccess::fence();
    // 最后再抢救一下
    if (TryLock(Self) > 0) return 1;
  }
  return 0;
}
```

自旋锁的逻辑本身比较简单，就是不停的尝试。但是虚拟机具体的实现其实就是如果自旋成功就奖励次数，如果自旋失败就减少次数。同时在自旋的多个阶段都直接TryLock尝试直接获取锁。