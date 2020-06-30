# jvm-12 线程与锁

## jvm线程的结构

```
- Thread
  - JavaThread
    - various subclasses eg CompilerThread, ServiceThread # 编译线程、JVMTI支持等
  - NonJavaThread
    - NamedThread
      - VMThread # 相当于控制线程
      - ConcurrentGCThread # GC线程，比如说Background GC的CMS轮询线程
      - WorkerThread # 并发任务线程，相当于线程池的线程
        - GangWorker
        - GCTaskThread
    - WatcherThread # 一些定时任务
    - JfrThreadSampler # 运行时数据采集线程
```

## Java线程的创建

我们一般创建一个线程都是通过new Thread()的方式，然后再调用start()方法。当我们调用new方法的时候，其实就是简单的创建了一个Java对象，而在我们调用start方法的时候，虚拟机会为我们在创建一个JavaThread（这个JavaThread再和一个osThread绑定）的线程，线程start之后会默认调用run方法，虚拟机的做法是把调用run方法的这个函数指针作为参数传给JavaThread，在所有的初始化操作都做完之后会调用这个函数指针指向的函数。

osThread还会和操作系统原生的操作系统关联，比如Linux上就会和pthread的tid进行关联。分配的这个pthread会设置堆栈大小等。

> 这里关于线程默认的堆栈大小，一般我们可以通过命令ulimit -s来查看当前操作系统的配置，一般默认是8mb，但是jvm的实现是根据不同的操作系统和不同的硬件架构指定了default_stack_size，这个stack_size可以作为pthread_create调用的一个参数。通常在linux下默认给的线程堆栈大小编译线程为4mb或2mb，java线程为1mb或512k居多。
>
> 同时不同操作系统也配置了最小堆栈，java可以通过-Xss来指定，通常最小值为64k。
>
> 在linux的实现上，stack_size还需要加上一个guard_size，posix标准约定需要这个guard page，一旦访问到guard page就会报错。不过JavaThread有一个Hotspot Guard Page，所以再分配Linux线城时没有再额外添加guard page。

到这里是正常的线程创建流程，但是如果线程数量过多，就会抛出OutOfMemory异常。

```c++
// 这里的jthread就是我们java中创建的Thread对象
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  bool throw_illegal_thread_state = false;

  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {
    // Ensure that the C++ Thread and OSThread structures aren't freed before
    // we operate.
    MutexLocker mu(Threads_lock);

    // Since JDK 5 the java.lang.Thread threadStatus is used to prevent
    // re-starting an already started thread, so we should usually find
    // that the JavaThread is null. However for a JNI attached thread
    // there is a small window between the Thread object being created
    // (with its JavaThread set) and the update to its threadStatus, so we
    // have to check for this
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      // 栈的大小
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      // Allocate the C++ Thread structure and create the native thread.  The
      NOT_LP64(if (size > SIZE_MAX) size = SIZE_MAX;)
      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);
			// osthread就是操作系统线程，这个地方判断是因为有可能因为内存不够，这个操作系统线程无法创建，后面会处理这种osthread为空的场景。
      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        // 这里的prepare就是Java的Thread对象和c++的Thread进行连接，同时更新一些引用、设置线程优先级以及把自己加到Threads全局对象中。
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

	// 这个地方就是有可能操作系统内存不够导致osthread创建失败
  if (native_thread->osthread() == NULL) {
    // No one should hold a reference to the 'native_thread'.
    native_thread->smr_delete();
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        os::native_thread_creation_failed_msg());
    }
    // 抛出OutOfMemoryError异常
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              os::native_thread_creation_failed_msg());
  }

	// 启动这个线程
  Thread::start(native_thread);

JVM_END
```

## JavaThread的结构

JavaThread是一个非常重要的结构，所以这个对象上有非常的字段，我们只列举一些和我们有关的一些字段

```c++
// 所有的Thread构成一个ThreadList，这个就是指向那个list的指针
ThreadsList* volatile _threads_hazard_ptr;

// Thread-local eden，ThreadLocal分配内存
ThreadLocalAllocBuffer _tlab;

// 锁相关
ObjectMonitor* _current_pending_monitor;
ObjectMonitor* _current_waiting_monitor;

// 方法执行和异常
Method*       _callee_target;
oop  _pending_exception;

// 线程的工作栈
JavaFrameAnchor _anchor;
address          _stack_base;
size_t           _stack_size;

// hash相关
jint _hashStateW;
jint _hashStateX;
jint _hashStateY;
jint _hashStateZ;
```

其中线程的堆栈遍历相对比较复杂，不同的操作系统有不同的实现，基本的实现思路是有一个stack_base指针，这个指针指向栈最初的地址，然后栈上有一个保护区域，一旦触碰到这个保护区域就是栈溢出，然后通过JavaFrameAnchor来walk整个栈，从last_frame开始一直遍历到最初的栈，其中针对一些编译优化过后的信息又一些特殊处理。

## 如何启动一个Thread

上面的源码分析降到了如何启动一个JavaThread，但是启动一个Java线程并正常的运行其实流程略微有些复杂。

- 在父线程（创建子线程的线程）中，会通过构建osthread对象，并获取osthread上的一个锁**startThread_lock**，然后会一直等待这个锁。

- 而创建出来的子线程会通过pthread的**pthread_create**方法（操作系统方法）执行被传入的一个启动函数（通过函数指针传入），在这个启动函数中，会对子线程进行自身的初始化，然后再初始化完成后会设置自己的线程状态为Initialized，并通知等待在**startThread_lock**的父线程。而后子线程会把自己挂在**startThread_lock**上，等待父线程通知自己继续执行后续的逻辑。

- 此时父线程会对初始化完的子线程进行一些检查并继续流程，并把子线程的状态翻转为**Runnable**。之后调用**pd_start_thread**（一个和操作系统相关的函数，不同的操作系统又不同的实现），在这个方法中会继续获取**startThread_lock**，并notify之前等待在**startThread_lock**的子线程继续执行。

- 子线程继续执行后会执行后面的方法，比如初始化一些tlab相关的逻辑、设置栈的guard_page防止爆栈等，之后会执行**thread_main_inner**，这个方法会执行初始化线程传入的一个函数指针

  ```c++
  static void thread_entry(JavaThread* thread, TRAPS) {
    HandleMark hm(THREAD);
    Handle obj(THREAD, thread->threadObj());
    JavaValue result(T_VOID);
    // 这里的run_method_name就是java线程的run方法
    JavaCalls::call_virtual(&result,
                            obj,
                            SystemDictionary::Thread_klass(),
                            vmSymbols::run_method_name(),
                            vmSymbols::void_method_signature(),
                            THREAD);
  }
  ```

以上就是完整的一个线程从创建到start，最后jvm给我们调用线程的run方法的完整流程。

下面我们具体看一下整个的具体实现。

```c++
void Thread::start(Thread* thread) {
	// DisableStartThread是一个开发参数，默认是生产时false
  if (!DisableStartThread) {
    if (thread->is_Java_thread()) {
      // Initialize the thread state to RUNNABLE before starting this thread.
      // Can not set it after the thread started because we do not know the
      // exact thread state at that time. It could be in MONITOR_WAIT or
      // in SLEEPING or some other state.
      // 设置RUNNABLE的状态
      java_lang_Thread::set_thread_status(((JavaThread*)thread)->threadObj(),
                                          java_lang_Thread::RUNNABLE);
    }
    os::start_thread(thread);
  }
}

void os::start_thread(Thread* thread) {
  // guard suspend/resume
  MutexLockerEx ml(thread->SR_lock(), Mutex::_no_safepoint_check_flag);
  OSThread* osthread = thread->osthread();
  osthread->set_state(RUNNABLE);
  // 这里就是实际启动线程的逻辑了，
  pd_start_thread(thread);
}
```

在每个操作系统启动线程的时候，可以传一个函数指针，作为线程启动后执行的参数，在jvm中这个方法是**thread_native_entry**，在linux下这个方法的具体实现如下：

```c++
static void *thread_native_entry(Thread *thread) {

  thread->record_stack_base_and_size();

  // Try to randomize the cache line index of hot stack frames.
  // This helps when threads of the same stack traces evict each other's
  // cache lines. The threads can be either from the same JVM instance, or
  // from different JVM instances. The benefit is especially true for
  // processors with hyperthreading technology.
  static int counter = 0;
  int pid = os::current_process_id();
  alloca(((pid ^ counter++) & 7) * 128);

  // 初始化一些操作系统相关
  thread->initialize_thread_current();

  OSThread* osthread = thread->osthread();
  // 获取线程启动的锁
  Monitor* sync = osthread->startThread_lock();

  // 设置线程id
  osthread->set_thread_id(os::current_thread_id());

  // log
  log_info(os, thread)("Thread is alive (tid: " UINTX_FORMAT ", pthread id: " UINTX_FORMAT ").",
    os::current_thread_id(), (uintx) pthread_self());

  // initialize signal mask for this thread
  os::Linux::hotspot_sigmask(thread);

  // initialize floating point control register
  os::Linux::init_thread_fpu_state();

  // handshaking with parent thread
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);

    // 这里就是把自己的状态设置为INITIALIZED，然后通知父线程继续执行后续的逻辑
    osthread->set_state(INITIALIZED);
    sync->notify_all();

    // 这里就是子线程等待父线程把状态翻过来，只要还是INITIALIZED，自己就要等待在这个锁上
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }

  // 执行线程的工作
  thread->call_run();

  // Note: at this point the thread object may already have deleted itself.
  // Prevent dereferencing it from here on out.
  thread = NULL;

  log_info(os, thread)("Thread finished (tid: " UINTX_FORMAT ", pthread id: " UINTX_FORMAT ").",
    os::current_thread_id(), (uintx) pthread_self());

  return 0;
}
```

```c++
void Thread::call_run() {
  // 已删除一些无用方法

  register_thread_stack_with_NMT();

  JFR_ONLY(Jfr::on_thread_start(this);)

	// JavaThread没有实现
  this->pre_run();

	// 执行方法
  this->run();

	// 到了这里就是Java线程的run方法已经结束

	// 后置处理
  // 主要是一些资源的清理，线程上未处理的异常处理等
  this->post_run();
}
```

```c++
void JavaThread::run() {
  // initialize thread-local alloc buffer related fields
  this->initialize_tlab();
  this->record_base_of_stack_pointer();
  this->create_stack_guard_pages();
  this->cache_global_variables();

  ThreadStateTransition::transition_and_fence(this, _thread_new, _thread_in_vm);

  DTRACE_THREAD_PROBE(start, this);
  this->set_active_handles(JNIHandleBlock::allocate_block());

  if (JvmtiExport::should_post_thread_life()) {
    JvmtiExport::post_thread_start(this);

  }

	// 执行Java线程对象的run方法
  thread_main_inner();
}

void JavaThread::thread_main_inner() {
  assert(JavaThread::current() == this, "sanity check");
  assert(this->threadObj() != NULL, "just checking");

  // 确认当前线程没有未抛出的异常或启动失败
  if (!this->has_pending_exception() &&
      !java_lang_Thread::is_stillborn(this->threadObj())) {
    {
      ResourceMark rm(this);
      this->set_native_thread_name(this->get_thread_name());
    }
    HandleMark hm(this);
    // 这个entry_point就是调用thread.run()的函数指针
    this->entry_point()(this, this);
  }

  DTRACE_THREAD_PROBE(stop, this);
}
```

到这里为止，整个Java线程已经成功启动并且已经开始正常工作了。那么在线程退出之后又会做什么样的工作呢。这块逻辑主要就是上面的post_run方法。

post_run做的事情比较多，主要包含一下几个方面

- 清空当前线程未被抛出的异常

  如果当前线程上有未被处理的异常，调用Thread上的dispatchUncaughtException方法（java方法）来处理这个异常。如果没有在线程上设置uncaughtExceptionHandler，默认会调用ThreadGroup的方法来处理，具体实现如下

  ```java
  public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
      parent.uncaughtException(t, e);
    } else {
      // 这个通常是null
      Thread.UncaughtExceptionHandler ueh =
        Thread.getDefaultUncaughtExceptionHandler();
      if (ueh != null) {
        ueh.uncaughtException(t, e);
      } else if (!(e instanceof ThreadDeath)) {
        System.err.print("Exception in thread \""
                         + t.getName() + "\" ");
        e.printStackTrace(System.err);
      }
    }
  }
  ```

- 调用线程退出清理方法Thread.exit()

  ```java
  private void exit() {
    if (group != null) {
      group.threadTerminated(this);
      group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
  }
  ```

- JVMTI事件

- 设置Thread的状态为TERMINATED

- 移除栈guard_page

- 清空TLAB空间

- 清空**BarrierSet**中没有flush的改动

- 从全局Thread列表中移除自己