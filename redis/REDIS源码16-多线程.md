# REDIS源码16-多线程

我们一直说Redis是一个单线程的模型，这句话确实没错，Redis再处理我们的业务逻辑时的确只使用了一个线程，但是Redis本身并不是仅有一个线程的，Redis还有另外几种线程再处理一些和业务无关的事情。

```c
/* Background job opcodes */
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3
```

可以看到，Redis的有三个后台线程专门处理对应的事情，分别是阻塞close、AOF的Fsync刷盘和Lazy Free。

后面分析一下Redis的源码，这些线程是什么时候创建的，创建后这些线程具体都做了些什么事情。

## 多线程的创建

Redis创建多线程的函数是**bioInit**方法，这个函数是在main方法的最后（设置EventLoop并进入asMain方法之前）的**InitServerLast**方法里面去初始化的。

```c
void InitServerLast() {
  bioInit();
  server.initial_memory_usage = zmalloc_used_memory();
}

void bioInit(void) {
  pthread_attr_t attr;
  pthread_t thread;
  size_t stacksize;
  int j;

  // BIO_NUM_OPS = 3，就代表了有三个后台线程
  for (j = 0; j < BIO_NUM_OPS; j++) {
    // 通过pthread创建多线程
    // 和openjdk一样，在Linux上的多线程实现都是pthread
    // jvm在创建线程时，OSThread也会和pthread进行绑定
    pthread_mutex_init(&bio_mutex[j],NULL);
    pthread_cond_init(&bio_newjob_cond[j],NULL);
    pthread_cond_init(&bio_step_cond[j],NULL);
    bio_jobs[j] = listCreate();
    bio_pending[j] = 0;
  }

  // 初始化pthread的相关参数，比如栈深度等
  pthread_attr_init(&attr);
  pthread_attr_getstacksize(&attr,&stacksize);
  if (!stacksize) stacksize = 1; /* The world is full of Solaris Fixes */
  while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
  pthread_attr_setstacksize(&attr, stacksize);

  // bioProcessBackgroundJobs就是每个线程启动之后执行的方法
  for (j = 0; j < BIO_NUM_OPS; j++) {
    void *arg = (void*)(unsigned long) j;
    if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
      serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
      exit(1);
    }
    bio_threads[j] = thread;
  }
}
```

到这里为止，线程创建就已经结束了，一共创建了三个线程，并且每个线程启动后都会执行**bioProcessBackgroundJobs**方法，这也是一个函数。后面继续分析这个方法都干了些什么事情。

##线程的执行

```c
void *bioProcessBackgroundJobs(void *arg) {
  struct bio_job *job;
  unsigned long type = (unsigned long) arg;
  sigset_t sigset;

  // 因为只有三个线程，如果type大于等于3，就说有问题
  if (type >= BIO_NUM_OPS) {
    serverLog(LL_WARNING,
              "Warning: bio thread started with wrong type %lu",type);
    return NULL;
  }

  // 配置pthread的特性，变为可以cancel
  pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
  pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, NULL);

  // 获取锁
  pthread_mutex_lock(&bio_mutex[type]);
  sigemptyset(&sigset);
  sigaddset(&sigset, SIGALRM);
  if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
    serverLog(LL_WARNING,
              "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

  // 死循环，不停的执行
  while(1) {
    listNode *ln;

    // 这个就是如果没有要执行的任务，就等待在锁上就可以了
    if (listLength(bio_jobs[type]) == 0) {
      pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
      continue;
    }
    // 到了这里就说明已经有任务了，所以就获取第一个任务
    ln = listFirst(bio_jobs[type]);
    job = ln->value;
		// 要执行任务的时候释放后台系统锁，因为其他地方可能会阻塞在这个锁上，释放这个锁，就说明对应的后台线程开始工作了
    pthread_mutex_unlock(&bio_mutex[type]);

    // 三种类型的线程，分别对应不同的地方来处理，后面具体分析
    if (type == BIO_CLOSE_FILE) {
      close((long)job->arg1);
    } else if (type == BIO_AOF_FSYNC) {
      redis_fsync((long)job->arg1);
    } else if (type == BIO_LAZY_FREE) {
      /* What we free changes depending on what arguments are set:
             * arg1 -> free the object at pointer.
             * arg2 & arg3 -> free two dictionaries (a Redis DB).
             * only arg3 -> free the skiplist. */
      if (job->arg1)
        lazyfreeFreeObjectFromBioThread(job->arg1);
      else if (job->arg2 && job->arg3)
        lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
      else if (job->arg3)
        lazyfreeFreeSlotsMapFromBioThread(job->arg3);
    } else {
      serverPanic("Wrong job type in bioProcessBackgroundJobs().");
    }
    // job已经处理完了，所以释放job占用的内存
    zfree(job);

		// 我这个任务跑完了，就获取锁，说明当前线程已经不在工作状态了
    pthread_mutex_lock(&bio_mutex[type]);
    // 把链表中的node删掉
    listDelNode(bio_jobs[type],ln);
    // pending的工作 - 1
    bio_pending[type]--;

    // 通知其他阻塞在当前线程正在工作的锁
    pthread_cond_broadcast(&bio_step_cond[type]);
  }
}
```

到这里线程已经创建完成，并且也已经阻塞在对应的锁上了。

那么下面就要分析一下不同type的线程都做了一些什么事情。

###BIO_CLOSE_FILE

```c
if (type == BIO_CLOSE_FILE) {
  close((long)job->arg1);
}
```

这个close方法就是unitsd.h里close，传入一个fd就可以了。就是一个简单的调用操作系统API的方法。非常简单。

###BIO_AOF_FSYNC

```c
 else if (type == BIO_AOF_FSYNC) {
   redis_fsync((long)job->arg1);
 }
```

等到分析AOF的时候在具体分析，目前只要知道有一个专门的Fsync线程在后台一直跑就可以了。

### BIO_LAZY_FREE

```c
 else if (type == BIO_LAZY_FREE) {
   /* What we free changes depending on what arguments are set:
    * arg1 -> free the object at pointer.
    * arg2 & arg3 -> free two dictionaries (a Redis DB).
    * only arg3 -> free the skiplist. */
   if (job->arg1)
     lazyfreeFreeObjectFromBioThread(job->arg1);
   else if (job->arg2 && job->arg3)
     lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
   else if (job->arg3)
     lazyfreeFreeSlotsMapFromBioThread(job->arg3);
 }
```

这个就是之前分析Expire的时候又分析道的Lazy Free，就是创建BackgroundJob然后交给这个线程来执行。里面做的事情也很简单，三种不同类型的对象，对应三个不同的方法来释放内存，做的事情就是简单的释放内存而已（引用计数 - 1，全局变量lazyfree_objects原子 - 1等），非常好理解。

