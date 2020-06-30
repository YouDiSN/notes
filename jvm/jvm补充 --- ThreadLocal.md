#jvm补充 --- ThreadLocal

ThreadLocal是一个非常常见的性能优化，并不只有在Java中才有，其实只要涉及多线程，就一定会有ThreadLocal的身影，比如GC中的对象提升，也是带有ThreadLocal的，每一个线程会申请一块promotion空间，然后优先使用自己的这块空间。

那么在内存的分配上有这样的优化也是非常常见的。那么在Java的ThreadLocal使用上（非Java类，虚拟机层面的tlab），tlab则是当有一个对象需要在jvm中分配时，每一个线程优先在自己独占的内存空间中分配（不过这里要注意的是所谓独占只是逻辑上独占，划出一块内存给到特定的线程，该特定线程可以在这块区域分配对象，其他线程技术上是可以读写这块内存空间的，只不过有可能会读，但是不会写）。

##ThreadLocalAllocBuffer

在Java中，tlab的实现就是**ThreadLocalAllocBuffer**，

```c++

```

