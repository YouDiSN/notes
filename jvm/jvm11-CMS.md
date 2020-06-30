#jvm11-CMS

cms原本是要做一个源码分析的，但是几个流程刚刚写到第二个并发标记，字数就已经突破八千字了，考虑到本身CMS就已经非常复杂了，再加上茫茫多的源码分析，可能效果不是很好，所以CMS打算分成两个部分，一个部分是一些基本概念的梳理，不涉及源码的讲解。在这个基础上，CMS源码的分析作为补充即可。

## CMS有什么特点

Concurrent Mark Sweep，是一个针对老年代的GC算法。CMS的目的是通过尽可能少的停顿用户线程，来提高Java的响应时间。CMS长时间使用后，由于是一个标记清除的算法，所以一定会有碎片化的问题。CMS有浮动垃圾问题，但是作为一个并行的GC算法，浮动垃圾是无法避免的（GC原理中有提过，并行算法只有两种，一种是增量式，一种是SATB）

## Full GC、CMS GC、Background GC、Foreground GC

Full GC和CMS GC我们常常会搞错，以为只要触发了CMS GC就一定是Full GC，其实这是两件完全没有关系的事情，CMS GC只是针对老年代的一个业务程序并行、多GC线程并发的回收，而Full GC则是STW+压缩的GC算法。

Foreground/Background GC是两种不同的GC模式，Background GC是有一个后台CMS Thread，会定时（2秒）去看一下是不是需要执行CMS GC，这是和应用程序代码并行的。而Foreground GC则是一个STW的算法，会完完整整的回收并压缩整个堆（是否需要触发压缩这件事情jdk8还有参数可以配置，再之后就没有参数可以配置了，默认每次都压缩）

其实Background GC就是CMS GC，Foreground GC就是Full GC。

## 触发GC的时机

```c++
enum Cause {
    /* public */
    _java_lang_system_gc, //调用System.gc()
    _full_gc_alot,
    _scavenge_alot,
    _allocation_profiler,
    _jvmti_force_gc,
    _gc_locker,
    _heap_inspection,
    _heap_dump, // dump整个堆
    _wb_young_gc, // wb都是whitebox
    _wb_conc_mark,
    _wb_full_gc,
    _archive_time_gc,

    /* implementation independent, but reserved for GC use */
    _no_gc,
    _no_cause_specified,
    _allocation_failure, // 分配失败

    /* implementation specific */

    _tenured_generation_full,
    _metadata_GC_threshold, // metadata空间不足
    _metadata_GC_clear_soft_refs,

    _cms_generation_full, // cms
    _cms_initial_mark,
    _cms_final_remark,
    _cms_concurrent_mark,

    _old_generation_expanded_on_last_scavenge,
    _old_generation_too_full_to_scavenge,
    _adaptive_size_policy,

    _g1_inc_collection_pause,
    _g1_humongous_allocation,
    _g1_periodic_collection,

    _dcmd_gc_run,

    _shenandoah_stop_vm,
    _shenandoah_allocation_failure_evac,
    _shenandoah_concurrent_gc,
    _shenandoah_traversal_gc,
    _shenandoah_upgrade_to_full_gc,

    _z_timer,
    _z_warmup,
    _z_allocation_rate,
    _z_allocation_stall,
    _z_proactive,

    _last_gc_cause
  };
```



##CMS Background GC的流程

- 初始标记（STW）

  这个阶段就是从GC Roots出发，在BitMap中做标记，注意这里标记的只有GC Roots，和YoungGC的标记大致差不多。

- 并发标记

  在上一个环节标记出来的GC Roots出发，并发的标记存活对象。这里的并发过程中的一些处理就是一旦有对对象应用的改动，就对对应的BitMap中标记为脏（通过WriteBarrier），后续在重新扫描。

  > 这里其实是CMS对于修改老年代引用的对象是灰色假定，对象在并发过程中可以接触任何对象（无论白色黑色灰色），但是在标记（标记的方式就在CardTable中标记一下）结束之前一定要在又一次标记来存活对象不会被误回收即可。
  >
  > CMS回收过程中，对于并发分配在新生代的对象又是黑色假定，因为在最终标记的时候会把GC Roots重新再标记一边（因为作为GC Roots，所以只要是在新生代中的对象一定就是GC Roots，这就是黑色假定），这个GC Roots就包含整个新生代对象。这也是为什么有些人会推荐在老年代回收钱先进行一次年轻代回收，这样会把GC Roots的数量降到足够低。

- 并发预清理

  这个阶段说是说预清理，其实也是标记的一种。通过遍历堆区（默认不包含survivor区域）、Final软弱虚4种引用，如果遍历到的对象之前的BitMap没有标记过，那么就说明这是一个新对象，BitMap中标记一下，同时把它放到一个MarkStack中。如果MarkStack中满了，就放到ModUnionTable中（也是一个BitMap）。这个的目的就是为了不丢失这次的引用变更。因为BitMap在CMS期间如果发生YoungGC还会做清理，比如ParNew过程中某一个Card的young对象都被清空了，那么BitMap会把dirty标记翻转。如果直接清理BitMap，那么可能会有并发问题（ParNew在清，CMS在标），所以需要分成两个地方来记录。

  上面标记出来的BitMap和ModUnionTable，会在接下来继续扫描已经被标记为Dirty的Card，然后重新扫描里面的对象。

  这个过程是和Java业务代码一起并发执行的，所以如果不加以限制，那么这个过程理论上可以永远持续下去，因为只要有Java代码在跑，就一定会有新的对象产生和老对象的无效，那么这个边界条件是：循环次数可控、每次循环会收到的DirtyCard比较多（说明有较大价值），回收脏卡片的速度和脏卡片生成的速度相比要足够快

  >  当前循环次数 <= (CMSPrecleanIter = 3) && (单次回收的DirtyCard <= (CMSPrecleanThreshold <= 1000)  || (curNumCards * (CMSPrecleanDenominator = 3) > lastNumCards * (CMSPrecleanNumerator = 2)))

  这个阶段还会对Eden区进行抽样，抽样的目的是最终remark的时候，因为要通过任务队列来做负载均衡，肯定要分配工作区域，这个抽样的目的是就是这个。

- 可失败的并发预清理（AbortablePreClean）

  上面那个过程就是不停的在并发的过程中更新那些被修改过的指针。那么在并发预清理结束之后，就需要在最终标记之前，尽可能多的清理一下没有用的引用，这样在最终标记的时候作为GC Roots的对象可以少一点，那么STW的时间也会更短。本次默认包含survivor但是不包含引用。

  如果Eden区的使用大于CMSScheduleRemarkEdenSizeThreshold（默认2mb）&& 总时间 <= CMSMaxAbortablePrecleanTime (默认5s) && Eden区使用率 < CMSScheduleRemarkEdenPenetration（50%），就会开启本次逻辑。

  之后只要没有ForeGroundGC的干扰以及老年代的剩余空间依然足够，就会一直执行preclean的逻辑，直到循环次数达到CMSMaxAbortablePrecleanLoops（默认是0，就是不限制）。如果单次执行处理的Dirty Card的数量太少，线程会sleep一段时间后再继续执行。

  > 这个过程做的事情和并发预清理基本上是一致的，两个过程实用的都是preclean_work函数，只是控制是否继续循环的策略不同。

  这个阶段还会对Eden区进行抽样，目的和作用同上。

- 最终标记（STW）

  最终标记阶段需要停止所有Java线程，如果配置了CMSScavengeBeforeRemark（默认false），就优先执行一次Young GC。

  1. 处理GC Roots，处理范围如下：年轻代、ClassLoaderDataGraph、Java线程、Universe的对象（Java Mirror Class以及一些常见的异常类等）、JNI、synchronize锁、Management类（监控信息）、JVMTI、AOT、SystemDictionary、CodeCache。
  2. 新的ClassLoader。有专门一个List保存新的ClassLoader
  3. DirtyCard
  4. 处理各种引用

- 并发清理

  这个过程其实就那些垃圾都已经找出来了，只要把这些带有垃圾的内存块合并到FreeList之中就可以下一次分配的时候使用了。不过这个阶段虽然说是说并发，但是实际上也是有很多限制的，这个阶段Java线程虽然可以继续运行，但是不可以触发YoungGC、不能触发CMS GC、bitMap也会被锁住不能看不能修改。

  这个过程另一个比较重要的点是如何计算和合并到FreeChunk中。

- 重新计算大小

  一次CMS Background GC已经完成了，之前收集到的一些统计进行计算，用于下一次的收集。收集到的数据可能用来调整上面的一些配置信息。包括堆大小的调整等。

- 重置

  清空BitMap等。CMS的GC State恢复到Idling的状态。
  
  ```c++
  switch (_collectorState) {
        case InitialMarking:
          {
            ReleaseForegroundGC x(this);
            stats().record_cms_begin();
            VM_CMS_Initial_Mark initial_mark_op(this);
            VMThread::execute(&initial_mark_op);
          }
          // The collector state may be any legal state at this point
          // since the background collector may have yielded to the
          // foreground collector.
          break;
        case Marking:
          // initial marking in checkpointRootsInitialWork has been completed
          if (markFromRoots()) { // we were successful
            assert(_collectorState == Precleaning, "Collector state should "
              "have changed");
          } else {
            assert(_foregroundGCIsActive, "Internal state inconsistency");
          }
          break;
        case Precleaning:
          // marking from roots in markFromRoots has been completed
          preclean();
          assert(_collectorState == AbortablePreclean ||
                 _collectorState == FinalMarking,
                 "Collector state should have changed");
          break;
        case AbortablePreclean:
          abortable_preclean();
          assert(_collectorState == FinalMarking, "Collector state should "
            "have changed");
          break;
        case FinalMarking:
          {
            ReleaseForegroundGC x(this);
  
            VM_CMS_Final_Remark final_remark_op(this);
            VMThread::execute(&final_remark_op);
          }
          assert(_foregroundGCShouldWait, "block post-condition");
          break;
        case Sweeping:
          // final marking in checkpointRootsFinal has been completed
          sweep();
          assert(_collectorState == Resizing, "Collector state change "
            "to Resizing must be done under the free_list_lock");
  
        case Resizing: {
          // Sweeping has been completed...
          // At this point the background collection has completed.
          // Don't move the call to compute_new_size() down
          // into code that might be executed if the background
          // collection was preempted.
          {
            ReleaseForegroundGC x(this);   // unblock FG collection
            MutexLockerEx       y(Heap_lock, Mutex::_no_safepoint_check_flag);
            CMSTokenSync        z(true);   // not strictly needed.
            if (_collectorState == Resizing) {
              compute_new_size();
              save_heap_summary();
              _collectorState = Resetting;
            } else {
              assert(_collectorState == Idling, "The state should only change"
                     " because the foreground collector has finished the collection");
            }
          }
          break;
        }
        case Resetting:
          // CMS heap resizing has been completed
          reset_concurrent();
          assert(_collectorState == Idling, "Collector state should "
            "have changed");
  
          MetaspaceGC::set_should_concurrent_collect(false);
  
          stats().record_cms_end();
          // Don't move the concurrent_phases_end() and compute_new_size()
          // calls to here because a preempted background collection
          // has it's state set to "Resetting".
          break;
        case Idling:
        default:
          ShouldNotReachHere();
          break;
      }
  ```

## CMS ForegroundGC

ForegroundGC触发的路径通常都是young gc收集失败，需要进行一次Full GC。首先一次Foreground GC会全局暂停所有的线程，专心用来做GC，因此Foreground暂停的时间会非常长。如果Background GC的一些并发过程做到一半，发生了ForegroundGC，有些情况是会跳过Background GC直接进入ForegroundGC，有些需要获取非常底层的锁，则会让ForegroundGC等待这个过程结束之后才会触发。总的来说，Foreground GC在并发安全的情况下优先级是最高的。

### Lisp2

那么在学习CMS具体实现之前，我们还是要先学习一下LISP2的算法实现。

Lisp2算法是一个历史悠久的回收算法，在2001年得到了并行回收的改进版本，大约在05、06年有了CMS的实现。Lisp2其实是一个标记-整理的算法。

**Lisp2算法需要遍历三次堆。**

- 第一阶段，回收器从GC Roots开始，遍历所有存活对象，同时在遍历的过程中，会计算出每个存活对象的最终地址，并把这个指针保存在对象上（jvm中就是对象头）
- 第二次堆遍历过程，回收器将对象头重记录的转发地址来更新所有对该对象的引用，以确保这些指针指向新对象的位置
- 第三次堆遍历过程，将每一个存活对象移动到新的目标位置。

Lisp2算法的整理顺序是滑动顺序（任意顺序、线性顺序和滑动顺序）

CMS ForegroundGC使用的就是Lisp2算法，算法分为几个阶段

1. 第一个阶段遍历所有GC Roots（和上面GC Roots范围基本差不多），标记所有活着的对象。

   这个阶段还会对类进行卸载、CodeCache中的编译后方法卸载

2. 给所有活着的对象计算新的内存地址。

3. 重新校准，所有引用老地址都修正到新地址

4. 把所有对象都挪过去

在上述4个阶段完成后，在执行一些后置的清理工作即可。

```c++
void GenMarkSweep::invoke_at_safepoint(ReferenceProcessor* rp, bool clear_all_softrefs) {

  GenCollectedHeap* gch = GenCollectedHeap::heap();

  set_ref_processor(rp);
  rp->setup_policy(clear_all_softrefs);

  gch->trace_heap_before_gc(_gc_tracer);

  // When collecting the permanent generation Method*s may be moving,
  // so we either have to flush all bcp data or convert it into bci.
  CodeCache::gc_prologue();

  // Increment the invocation count
  _total_invocations++;

  // Capture used regions for each generation that will be
  // subject to collection, so that card table adjustments can
  // be made intelligently (see clear / invalidate further below).
  gch->save_used_regions();

  allocate_stacks();

  mark_sweep_phase1(clear_all_softrefs);

  mark_sweep_phase2();

  mark_sweep_phase3();

  mark_sweep_phase4();

  restore_marks();

  // Set saved marks for allocation profiler (and other things? -- dld)
  // (Should this be in general part?)
  gch->save_marks();

  deallocate_stacks();

  // If compaction completely evacuated the young generation then we
  // can clear the card table.  Otherwise, we must invalidate
  // it (consider all cards dirty).  In the future, we might consider doing
  // compaction within generations only, and doing card-table sliding.
  CardTableRS* rs = gch->rem_set();
  Generation* old_gen = gch->old_gen();

  // Clear/invalidate below make use of the "prev_used_regions" saved earlier.
  if (gch->young_gen()->used() == 0) {
    // We've evacuated the young generation.
    rs->clear_into_younger(old_gen);
  } else {
    // Invalidate the cards corresponding to the currently used
    // region and clear those corresponding to the evacuated region.
    rs->invalidate_or_clear(old_gen);
  }

  CodeCache::gc_epilogue();
  JvmtiExport::gc_epilogue();

  // refs processing: clean slate
  set_ref_processor(NULL);

  // Update heap occupancy information which is used as
  // input to soft ref clearing policy at the next gc.
  Universe::update_heap_info_at_gc();

  // Update time of last gc for all generations we collected
  // (which currently is all the generations in the heap).
  // We need to use a monotonically non-decreasing time in ms
  // or we will see time-warp warnings and os::javaTimeMillis()
  // does not guarantee monotonicity.
  jlong now = os::javaTimeNanos() / NANOSECS_PER_MILLISEC;
  gch->update_time_of_last_gc(now);

  gch->trace_heap_after_gc(_gc_tracer);
}
```

## CMS的锁

CMS有一下这些锁的概念，

*Safepoint*：全局安全点。

*CMS token*：如果获取了这个CMS Token，STW的GC（或者GC的一个阶段）无法进行。这个token通常都是自己线程在执行，不希望其他线程来打扰自己，就会获取这个锁。这个锁的Level比下面的CGC_lock要高。同时，VMThread和CMSThread都有可能会尝试获取这个锁，VMThread的优先级要更高。

*CGC_lock*：这是一个比较底层的锁，通常都是用来锁住然后翻一些状态，比如一个field要设置为ture，为了确保线程安全，就通过这把锁。

*freelistLock*：获取了这个锁，老年代无法继续分配。比如超大对象或YoungGC对象提升。

*bitMapLock*：BitMap无法查看或更新。

## CMS调优基本思路

1. 尽可能不要出发Foreground GC，或者在业务低峰期定期触发一次，比如凌晨2点可以逐批系统执行FullGC。
2. CMS不适用于堆非常大的应用，一般来说超过16g就不建议用CMS了，因为一旦触发Full GC，停顿的时间非常长。同时CMS也不适用于非常小的堆，一般4g以下也没有使用CMS的必要。
3. CMS GC是否需要非常频繁。CMS GC默认是60%执行一次，但是考虑Java Web系统很少有常驻内存的东西，因此可考虑适当调大比例。
4. 合理设置survivor区提升年龄，如果对象确实需要进入老年代的，就不要把这个年龄设置的太大，防止平凡拷贝，
5. DisableExplicitGC，很多网上的文章都推荐做这个配置，禁止显示的调用System.gc()。但是这个不应该作为一个闭着眼睛配置的参数，有些可能会需要依赖这个配置对一些堆外内存的堆内引用进行清理等（比如一些IO的Bytebuffer），由于我们所引用的依赖一般都是一些适用度比较高的依赖，很少会滥用System.gc()。
6. CMS中的一些并行开关。这些并行开关还是应该根据日志来打开，只有确认在这个环节花费了大量的时间，才应该打开这个开关，否则其实没有必要。比如我们陆金所的一些业务，并发处理引用可能未必需要，因为我们软弱虚Final引用不会很多。
7. 并发收集线程数不能太大。GC线程数不要一股脑的等于CMS核数，太多的GC线程反而会导致比较严重的线程并发争用。在我们的4C4G的虚拟机机器其实还好，主要是一些大数据的物理机达到128C512G才有对CMS GC线程数调优的必要，不过这种堆的规模，目前主推也不应该是CMS，应该开G1。
8. 在Final Remark前执行一次Young GC，这个开关设置的理由应该是：不执行YoungGC的GC Roots遍历时间 > 执行YoungGC的停顿 + 没有回收的对象的GC遍历，这种情况才应该开启这个开关。不过鉴于JavaWeb应用大部分对象都没有活着的必要，所以其实可以闭着眼睛打开。但是比如一些大数据应用，大量数据都是驻留内存的，执行一次Young GC可能回收不了太多，可能就需要具体考虑了。
9. 如果因为GC时间比较长，第一步要先增加日志，只有根据日志发现哪些时间的占比比较高，然后针对性的调优。
10. 如果打了日志发现GC时间不长，但是系统还是一只会停顿，有可能是Safepoint的问题，需要打开Safepoint的日志。
11. 如果有大量对象晋升，PLAG可以优先考虑调优。
12. jdk9之后不要再想着用CMS了，已经过时了，用G1吧。