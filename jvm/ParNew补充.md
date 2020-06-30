#ParNew补充

## 多大的size才会跳过young区，不在young区分配

```c++
// defNewGeneration.hpp（247），parNew继承了这段代码
virtual bool should_allocate(size_t word_size, bool is_tlab) {
  assert(UseTLAB || !is_tlab, "Should not allocate tlab");
  // overflow_limit = 1 << (64 - 3)（64位机器）
  size_t overflow_limit    = (size_t)1 << (BitsPerSize_t - LogHeapWordSize);

  const bool non_zero      = word_size > 0;
  // 是否大于overflow_limit
  const bool overflows     = word_size >= overflow_limit;
	// _pretenure_size_threshold_words是GC可以配置的参数：PretenureSizeThreshold，默认0
  // product(size_t, PretenureSizeThreshold, 0,                           \     
  //    "Maximum size in bytes of objects allocated in DefNew "           \
  //    "generation; zero means no maximum")                              \
  //    range(0, max_uintx)  
  // 如果配置了PretenureSizeThreshold，就检查是否大于这个配置
  const bool check_too_big = _pretenure_size_threshold_words > 0;
  const bool not_too_big   = word_size < _pretenure_size_threshold_words;
  const bool size_ok       = is_tlab || !check_too_big || not_too_big;

  bool result = !overflows &&
    non_zero   &&
    size_ok;

  return result;
}
```

## 什么情况下会跳过YoungGC直接进行FullGC

```c++
// Returns true if an incremental collection is likely to fail.
// We optionally consult the young gen, if asked to do so;
// otherwise we base our answer on whether the previous incremental
// collection attempt failed with no corrective action as of yet.
bool incremental_collection_will_fail(bool consult_young) {
  // 看上一次是否失败
  return incremental_collection_failed() ||
    (consult_young && !_young_gen->collection_attempt_is_safe());
}

bool DefNewGeneration::collection_attempt_is_safe() {
  // 这段代码就是一个细腻的检查
  // to区域没有清空，一般情况下不可能
  if (!to()->is_empty()) {
    log_trace(gc)(":: to is not empty ::");
    return false;
  }
  if (_old_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _old_gen = gch->old_gen();
  }
  // used就是年轻代使用大小
  return _old_gen->promotion_attempt_is_safe(used());
}

// max_promotion_in_bytes就是年轻代大小
bool ConcurrentMarkSweepGeneration::promotion_attempt_is_safe(size_t max_promotion_in_bytes) const {
  // 老年代剩余大小
  size_t available = max_available();
  // 历次GC平均提升到老年代的大小
  size_t av_promo  = (size_t)gc_stats()->avg_promoted()->padded_average();
  // 剩余的大小比平均大或者剩余大小比年轻代全部提升过来还要大
  bool   res = (available >= av_promo) || (available >= max_promotion_in_bytes);
	// 省略一些日志的输出
  return res;
}
```

## AgeTable的具体使用

AgetTable是一个记录对象年龄的Map，根据AgetTable会动态的计算哪些年龄的对象需要被提升到老年代。那么这块的计算逻辑如下，

```c++
// 这段是回收结束后的处理
if (!promotion_failed()) {
  eden()->clear(SpaceDecorator::Mangle);
  from()->clear(SpaceDecorator::Mangle);
  if (ZapUnusedHeapArea) {
    to()->mangle_unused_area();
  }
  swap_spaces();
  size_policy->reset_gc_overhead_limit_count();
	// 这里就是计算AgeTable的相关逻辑
  adjust_desired_tenuring_threshold();
} else {
  handle_promotion_failed(gch, thread_state_set);
}
```

```c++
void DefNewGeneration::adjust_desired_tenuring_threshold() {
  // to区域大小
  size_t const survivor_capacity = to()->capacity() / HeapWordSize;
  // TargetSurvivorRatio是一个配置是一个数字，比如50（默认也是50），这个就是计算你期望的survivor大小是多少
  size_t const desired_survivor_size = (size_t)((((double)survivor_capacity) * TargetSurvivorRatio) / 100);
	// 根据你上面的期望survivor大小，再根据AgeTable里面记录的，算出来多少岁以上的提升可以满足你的要求
  _tenuring_threshold = age_table()->compute_tenuring_threshold(desired_survivor_size);

  //  product(bool, UsePerfData, true,                                      \
  //      "Flag to disable jvmstat instrumentation for performance testing "\
  //      "and problem isolation purposes")  																\
	//  默认是true
  if (UsePerfData) {
    GCPolicyCounters* gc_counters = GenCollectedHeap::heap()->counters();
    gc_counters->tenuring_threshold()->set_value(_tenuring_threshold);
    gc_counters->desired_survivor_size()->set_value(desired_survivor_size * oopSize);
  }

  age_table()->print_age_table(_tenuring_threshold);
}
```

```c++
uint AgeTable::compute_tenuring_threshold(size_t desired_survivor_size) {
  uint result;

  // 如果这两个又一个为ture，默认就是MaxTenuringThreshold，这个也是可以配置的
  if (AlwaysTenure || NeverTenure) {
    assert(MaxTenuringThreshold == 0 || MaxTenuringThreshold == markOopDesc::max_age + 1,
           "MaxTenuringThreshold should be 0 or markOopDesc::max_age + 1, but is " UINTX_FORMAT, MaxTenuringThreshold);
    result = MaxTenuringThreshold;
  } else {
    size_t total = 0;
    uint age = 1;
    assert(sizes[0] == 0, "no objects with age zero should be recorded");
    // 不然就一个while循环去从1岁开始不停的累积
    while (age < table_size) {
      total += sizes[age];
      // check if including objects of age 'age' made us pass the desired
      // size, if so 'age' is the new threshold
      if (total > desired_survivor_size) break;
      age++;
    }
    // 算出来的age和配置的比较一下
    result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;
  }

  return result;
}
```

## ParNewGC调优思路

**一定要有GC日志**，GC日志除了我们日常都会打的一些之外，还有一些是可选的，比如开启Safepoint相关记录看Safepoint的相关统计信息，还可以增加打印Reference处理的相关信息，总之就是要确定gc时间长，到底长在哪里。

通用思路：

- 合理设置TLAB大小，使得对象分配的时候可以更快

- 不要跳过YoungGC

  比如old区太满没有太多空间

- YoungGC对象直接被提升到老年代

  - to space不够用，直接提升到老年代
  - 对象年龄、期望的to space使用率设置不合理，导致年轻对象被提升到老年代

- PLAB大小设置不合理，申请新的

- 合理分配年轻代大小（比如年轻代和老年代的比例、eden+from+to的空间比例等）

- 尽量少使用FinalReference（实现了finalize方法）和PhantomReference，这两个在ReferenceProcessor阶段对性能有比较大的影响，尽量减少使用。

更多参数可以参见globals.hpp和gc_globals.hpp，里面有所有各个算法相关的gc配置，总结来说就是要先确定问题在哪里，然后看如何通过参数调整来降低这个问题带来的影响。