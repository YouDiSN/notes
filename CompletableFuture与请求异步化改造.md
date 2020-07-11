# CompletableFuture与请求异步化改造

web层的业务逻辑总体比较轻量，绝大部分的请求都会是调用后端子系统的接口的等待中。那么自然而然的我们就会想到通过并发和异步来提高整体的性能。

所以如何更简单的能获取异步的能力就称为一个非常重要的命题，mobile-core的异步目前具备两种模式，

- 第一种是通过获取线程池向线程池中递交VoidTask，该种方式需要写一些模版代码，把每一个并发任务都封装进VoidTask任务中，同时这样的方式使用的门槛也比较高。
- 第二种方式是通过@Async注解的形式来实现异步，这种方式的使用难度相对会更简单

那么在最耗时的请求调用中目前mobile-core暂时并没有原生支持，或者使用异步的方式必须通过上述两种方式来实现。

所以基于此，开发了一套接口调用原生的支持异步的模式，在AutoRemoteFacade模式下，使用异步Future只需要在接口声明中声明即可

```java
@AutoRemotefacade("cms")
public interface CmsAppAutoRemoteFacade extends IRemoteFacade {
  
  // 这个方式就老的方式，需要同步阻塞获取cms配置结果
  @FacadeGetMapping("/service/config/query-app-by-cfg")
  CmsAppConfigDTO getCmsConfig();
  
  // 该种方式直接返回一个Future
  @FacadeGetMapping("/service/config/query-app-by-cfg")
  Future<CmsAppConfigDTO> getCmsConfig2();

  // 该种方式返回一个Java8的CompletableFuture
  @FacadeGetMapping("/service/config/query-app-by-cfg")
  CompletableFuture<CmsAppConfigDTO> getCmsConfig3();

}
```

在新版本的mobile-core，实现异步非常简单，只需要在接口后返回的类型处声明自己是一个Future（要注意范型应当准确）即可，新版本（V1.7.4）的所有请求全部都是通过异步的方式实现，如果声明的类型不是Future，就手动get一下，如果是Future就直接返回。

##CompletableFuture

CompletableFuture是Java8中引入的实现，引入这个类的目的是因为Java之前的异步Future并不支持callback回调，所以就有了Guava的ListenableFuture等，所以在Java8中迫切的需要一套自己的可以执行注册回调函数的Future实现作为Java系的通用Future实现的标准。

### 线程池

首先，CompletableFuture是可以自己指定线程池的，可以用子类继承通过重写 defaultExecutor() 方法来完成。或者在每一个大类的API中，都提供了一个可以传入一个线程池作为参数的方法，也可以用这个方法。传入线程池的好处就在于，自己实现的线程池可能添加了诸如熔断，监控，自定义的拒绝策略等各种相关的方法。更适合企业级应用。

### 进行变换(Map)

```java
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn,Executor executor);
```

async结尾的就是可以继续异步执行的，传入的参数依然是函数式接口

### 进行转换(FlatMap)

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor)
```

Compose就是组合的函数本身返回的也是一个CompletionStage，如果不用这个最后返回的结果是CompletableFuture<CompletableFuture<?>>类型，如果用了thenCompose系列，就会拍扁。

### 进行消耗

```java
public CompletionStage<Void> thenAccept(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```

使用Consumer，不返回任何东西，和变换的主要还是函数式接口的语义不同

### 对上一步的计算结果不关心，执行下一个操作。

```java
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```

传入的参事是Runnable，依然是函数式接口

### 结合两个CompletionStage的结果，进行转化后返回

```java
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

将两个CompletionStage结合，并且返回一个新的CompletionStage

### 结合两个CompletionStage的结果，进行消耗

```java
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action,     Executor executor);
```

还是可以通过BiConsumer这个函数式接口看出来

### 在两个CompletionStage都运行完执行

```java
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

不关心结果如何，只关心在两个执行完毕之后进行执行。

### 两个CompletionStage，谁计算的快，我就用那个CompletionStage的结果进行下一步的转化操作。

```java
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```

两个哪一个算好了就进行下一步的计算，不等待另外一个

### 两个CompletionStage，谁计算的快，我就用那个CompletionStage的结果进行下一步的消耗操作。

```java
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

区别主要还是体现在接口上面，Consumer接口的参数

### 两个CompletionStage，任何一个完成了都会执行下一步的操作（Runnable）。

```java
public CompletionStage<Void> runAfterEither(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

任何一个完成就可以

### 当运行完成时，对结果的记录。这里的完成时有两种情况，一种是正常执行，返回值。另外一种是遇到异常抛出造成程序的中断。这里为什么要说成记录，因为这几个方法都会返回CompletableFuture，当Action执行完毕后它的结果返回原始的CompletableFuture的计算结果或者返回异常。所以不会对结果产生任何的作用。

```java
public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action,Executor executor);
```

相当于结束之后打log这样的功能

### 运行完成时，对结果的处理。这里的完成时有两种情况，一种是正常执行，返回值。另外一种是遇到异常抛出造成程序的中断。

```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

和上一个方法的区别就在于，这三个方法只处理异常，而上面三个方法则是不论是否正常完成还是产生了异常，都会进行处理

## 实现原理

```java
// 这个result可能是正常的result，或者是AltResult类。
// AltResult就是封装了null和Throwable，可以认为是一种特殊的Result
volatile Object result;
// Treiber stacks的top节点
volatile Completion stack;
```

一个CompletableFuture有可能有多个依赖的操作，这些以来的操作会被收集在一个Linked Stack中，这些以来用CAS原子的更新result字段，然后会从Stack中pop出下一个操作。这些操作可能包含常规操作、异常处理；同步进行、异步进行等各种类型。

如果result字段不为null，就可以认为是这个CompletableFuture对应的操作已经完成了。同时使用AltResult类来表示当前的结果是null或者发生了异常。使用了这样一个字段可以让CompletableFuture非常易于追踪。

一个CompletableFuture的以来使用了Treiber Stack来管理。

> **Treiber stacks**
>
> Treiber Stack Algorithm是一个可扩展的无锁栈，利用细粒度的并发原语CAS来实现的。
>
> 只有当要添加的item是自开始操作以来唯一添加的item时，才会添加新的item。 这是通过使用CAS完成的。 在添加新item时使用堆栈，将堆栈的顶部放在新item之后。 然后，将这个新构造的头元素（旧头）的第二个item与当前item进行比较。 如果两者匹配，那么你可以将旧头换成新头，否则就意味着另一个线程已经向堆栈添加了另一个项目，在这种情况下，你必须再试一次。
>
> 当从堆栈中弹出一个item时，在返回项目之前，您必须检查另一个线程自操作开始以来没有添加其他item。

栈中主要包含的操作有

- 单个输入 (UniCompletion),
- 两个输入 (BiCompletion),
- 聚合 (BiCompletions using exactly one of two inputs),
- 共享 (CoCompletion, used by the second of two sources),
- 没有输入 source actions,
- 唤醒其他没有被阻塞的waiter

Completion类是ForkJoinTask的子类，可以支持异步操作。同时也实现了Runnable接口用于普通的使用。

- Completion各种子类所包含的内容和作用大致相同，只不过一些细节不一样。封装了这么多的目的就是我们使用的时候可以不用关注太多的细节。
- 一些返回boolean的方法的作用就是检查所有的传入参数，然后执行对应的action或者异步执行相关操作。方法返回true如果操作成功。
- tryFile方法调用相关的方法（返回boolean的那些方法）并持有他的变量，如果调用成功就会清理。这方法需要一个mode参数表示同步或者异步执行。调度执行的过程中发生了异常就会记录，也有可能被一个task调用。cliam()回调禁止调用一个已经被其他线程执行的task。
- 有一些类（比如UniApply）有不一样的处理代码，当处理非常明确的thread-confined（线程局限，就是只会局限在当前线程的操作）
- xStage方法会在public状态下调用，他会记录下调用的参数并且调用and/or方法创建一个stage对象，如果不是异步并且已经可以触发，那么就会立即执行。不然一个Completion对象会被创建，如果可以被执行那么就会交给Executor来执行，或者push到stack中。Completion的操作是从tryFile方法开始。我们会在Completion被push到source对象的stack后再次检查，以便能提升，因为source对象有可能在push的过程中已经完成了。
  那些有两个input的对象也会在push的过程中尝试提升性能。第二个Completion是一个CoCompletion对象，同时会指向第一个Completion，这回共享以便确保至多只有一个action被执行。allOf方法会把Completions对象当作一棵树。anyOf方法会在任何一个Completion操作结束后清空stack，anyOf Completion对象能获取到所有操作的一个共享数组。

postComplete方法会被调用，除非这个target被确保不会被观测到（没有被Linked也不会被return），postComplete可以被多线程调用，这个方法会原子的弹出各自的Completion，并且会使用tryFile来触发这些Completion。

在执行的过程中，CompletableFuture会尽可能的让一些不用的field，已经被cancel的Stack中的Node设置为null，以便帮助尽可能快的gc。Completion对象中的字段都是单线程访问，所以不需要做任何同步的工作。

