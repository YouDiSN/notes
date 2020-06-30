# Java异常处理和invokedynamic

上次和大家分享了Java的方法调用（主要是Zero解释器），还有两块很重要的内容，一个是Java的异常处理，以及Java7加入的invokedynamic。这次就主要分享一下这两个

## 异常处理

Java的异常处理主要分成两类，第一种**主动throw异常**，第二种是**被动throw**，这两种方式的区别其实就是创建异常的方式不同，一个是我们自己主动new一个异常，另一个是虚拟机为我们创建一个异常，两者殊途同归。

### 虚拟机创建异常

这种方式的异常抛出主要比如数组越界，强制类型转换错误等，属于我们自己写的时候是不认为这里会出来异常的。我们以数组越界为例

```java
public class Test {
  public void static main(String... args) {
    int[] array = new int[]{};
    int a = array[0];
  }
}
```

编译后的字节码如下：

```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_0 // 栈上先压入一个0，下面newarray需要一个参数count，代表创建一个长度为n的数组
         1: newarray       int
         3: astore_1 // 放在位置为1的地方
         4: aload_1 // load位置为1的变量，就是创建出来的数组
         5: iconst_0 // 栈上再压一个0，因为我们要获取第0位置的元素
         6: iaload // 从数组上获取
         7: istore_2 // 获取出来的东西放在变量表下标为2的地方
         8: return // return
      LineNumberTable:
        line 11: 0
        line 12: 4
        line 13: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
            4       5     1 array   [I
            8       1     2     i   I
}
```

那么我们可以看到，从数组中根据下标获取元素的字节码指令是iaload，找到Zero解释器中具体的实现，

```c++
{
  		// 从栈上拿到iaload字节码前两位的那个对象（就上面aload_1指令load到的数据），并强制类型转换为arrayOop
  		arrayOop arrObj = (arrayOop)STACK_OBJECT(-2);
  		// 获取iaload指令前一位的指令作为数组下标
      jint     index  = STACK_INT(-1);
      char message[jintAsStringSize];
      CHECK_NULL(arrObj);
		  // 如果获取到的数组下标，>=该数组的实际长度，就抛出异常
      if ((uint32_t)index >= (uint32_t)arrObj->length()) {
          sprintf(message, "%d", index);
	        // 这个就是数组越界抛出异常的地方，
	        // vmSymbols::java_lang_ArrayIndexOutOfBoundsException()，就是抛出的异常类型是数组越界异常
          VM_JAVA_ERROR(vmSymbols::java_lang_ArrayIndexOutOfBoundsException(),
                        message, note_rangeCheck_trap);
      }
	  	// 如果没有问题，就从数组中获取元素，并且压入栈中
      SET_STACK_INT(*(jint *)(((address) arrObj->base(T_INT)) + index * sizeof(T2)), -2);
  		UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
}
```

这个VM_JAVA_ERROR的具体实现如下

```c++
#define VM_JAVA_ERROR_NO_JUMP(name, msg, note_a_trap)                             \
    DECACHE_STATE();                                                              \
    SET_LAST_JAVA_FRAME();                                                        \
    {                                                                             \
       InterpreterRuntime::note_a_trap(THREAD, istate->method(), BCI());          \
       ThreadInVMfromJava trans(THREAD);                                          \
       Exceptions::_throw_msg(THREAD, __FILE__, __LINE__, name, msg);             \
    }                                                                             \
    RESET_LAST_JAVA_FRAME();                                                      \
    CACHE_STATE();

```

其中比较重要的是 Exceptions::_throw_msg(THREAD, __FILE__, __LINE__, name, msg); 方法

```c++
void Exceptions::_throw_msg(Thread* thread, const char* file, int line, Symbol* name, const char* message,
                            Handle h_loader, Handle h_protection_domain) {
  // Check for special boot-strapping/vm-thread handling
  if (special_exception(thread, file, line, name, message)) return;
  // Create and throw exception
  Handle h_cause(thread, NULL);
  Handle h_exception = new_exception(thread, name, message, h_cause, h_loader, h_protection_domain);
  _throw(thread, file, line, h_exception, message);
}

Handle Exceptions::new_exception(Thread* thread, Symbol* name,
                                 const char* message, Handle h_cause,
                                 Handle h_loader, Handle h_protection_domain,
                                 ExceptionMsgToUtf8Mode to_utf8_safe) {
  JavaCallArguments args;
  Symbol* signature = NULL;
  // 省略的部分就是看当前线程是否已经有一个要抛出的异常了，如果已经有了就直接返回那个异常，否则就创建一个新的
  // 看是否有异常的方式是thread->has_pending_exception()方法
  // 这里还会构造 error_message，并且放入args中，如果有error_message就调用有一个String的构造函数，否则就调用空构造函数
  return new_exception(thread, name, signature, &args, h_cause, h_loader, h_protection_domain);
}
```

```c++
Handle Exceptions::new_exception(Thread *thread, Symbol* name,
                                 Symbol* signature, JavaCallArguments *args,
                                 Handle h_cause,
                                 Handle h_loader, Handle h_protection_domain) {
  // 获取一个新的异常，实现在下面
  Handle h_exception = new_exception(thread, name, signature, args, h_loader, h_protection_domain);

  // 如果这个异常的cause不为空，就调用initCause的方法（见Throwable.java）
  if (h_cause.not_null()) {
    assert(h_cause->is_a(SystemDictionary::Throwable_klass()),
        "exception cause is not a subclass of java/lang/Throwable");
    JavaValue result1(T_OBJECT);
    JavaCallArguments args1;
    args1.set_receiver(h_exception);
    args1.push_oop(h_cause);
    JavaCalls::call_virtual(&result1, h_exception->klass(),
                                      vmSymbols::initCause_name(),
                                      vmSymbols::throwable_throwable_signature(),
                                      &args1,
                                      thread);
  }

  // 是否有待抛出异常
  if (thread->has_pending_exception()) {
    h_exception = Handle(thread, thread->pending_exception());
    thread->clear_pending_exception();
  }
  return h_exception;
}

Handle Exceptions::new_exception(Thread *thread, Symbol* name,
                                 Symbol* signature, JavaCallArguments *args,
                                 Handle h_loader, Handle h_protection_domain) {
  Handle h_exception;

  // 先通过SystemDicitonary进行类加载
  Klass* klass = SystemDictionary::resolve_or_fail(name, h_loader, h_protection_domain, true, thread);

  // 如果当前线程没有待处理异常，就创建一个新的
  if (!thread->has_pending_exception()) {
    assert(klass != NULL, "klass must exist");
    // 这个就是通过 klass->allocate_instance_handle(CHECK_NH); 分配一块内存，然后再调用初始化的方法，由于上面是否构造出error_message来决定调用哪个构造函数
    h_exception = JavaCalls::construct_new_instance(InstanceKlass::cast(klass),
                                signature,
                                args,
                                thread);
  }

  // 是否有待抛出异常
  if (thread->has_pending_exception()) {
    h_exception = Handle(thread, thread->pending_exception());
    thread->clear_pending_exception();
  }
  return h_exception;
}
```

到这里就是一个异常的正常创建和处理，接下来就要看throw的逻辑

```c++
void Exceptions::_throw(Thread* thread, const char* file, int line, Handle h_exception, const char* message) {
  ResourceMark rm;
  assert(h_exception() != NULL, "exception should not be NULL");

  // 记录一些log
  log_info(exceptions)("Exception <%s%s%s> (" INTPTR_FORMAT ") \n"
                       "thrown [%s, line %d]\nfor thread " INTPTR_FORMAT,
                       h_exception->print_value_string(),
                       message ? ": " : "", message ? message : "",
                       p2i(h_exception()), file, line, p2i(thread));
  // 看是不是AbortVMOnException异常
  Exceptions::debug_check_abort(h_exception, message);

  // 是否是不能调用java代码的虚拟机线程抛出的异常
  if (special_exception(thread, file, line, h_exception)) {
    return;
  }

  // 是否是内存溢出
  if (h_exception->is_a(SystemDictionary::OutOfMemoryError_klass())) {
    count_out_of_memory_exceptions(h_exception);
  }

  // 是否是LinkageError，这个Error常见于jar包冲突后，同一个类有两个版本
  if (h_exception->is_a(SystemDictionary::LinkageError_klass())) {
    Atomic::inc(&_linkage_errors);
  }

  assert(h_exception->is_a(SystemDictionary::Throwable_klass()), "exception is not a subclass of java/lang/Throwable");

  // 设置pending exception
  thread->set_pending_exception(h_exception(), file, line);

  // 日志
  if (LogEvents){
    Events::log_exception(thread, "Exception <%s%s%s> (" INTPTR_FORMAT ") thrown at [%s, line %d]",
                          h_exception->print_value_string(), message ? ": " : "", message ? message : "",
                          p2i(h_exception()), file, line);
  }
}
```

可以看到整个抛出异常的流程其实就是创建异常，并且做一些相关的校验和处理的工作，然后将该异常set到当前线程的pending_exception之中。

到了这里为止我们有了异常，也在当前线程中设置了异常，下面就是后续的处理。当创建并设置玩pending_exception之后会通过goto命令跳转到这里。

```c++
// An exception exists in the thread state see whether this activation can handle it
  handle_exception: {

    HandleMarkCleaner __hmc(THREAD);
    Handle except_oop(THREAD, THREAD->pending_exception());
    // Prevent any subsequent HandleMarkCleaner in the VM
    // from freeing the except_oop handle.
    HandleMark __hm(THREAD);

    THREAD->clear_pending_exception();
    assert(except_oop() != NULL, "No exception to process");
    intptr_t continuation_bci;
    // expression stack is emptied
    topOfStack = istate->stack_base() - Interpreter::stackElementWords;
    // 找到当前方法上是否可以处理这个异常
    CALL_VM(continuation_bci = (intptr_t)InterpreterRuntime::exception_handler_for_exception(THREAD, except_oop()),
            handle_exception);

    except_oop = Handle(THREAD, THREAD->vm_result());
    THREAD->set_vm_result(NULL);
    // 如果在当前方法中找到可以处理该异常的代码，就从哪个位置开始执行
    if (continuation_bci >= 0) {
      // 重组整个方法栈
      SET_STACK_OBJECT(except_oop(), 0);
      MORE_STACK(1);
      // 更新接下去要执行的pc寄存器值
      pc = METHOD->code_base() + continuation_bci;
      if (log_is_enabled(Info, exceptions)) {
        ResourceMark rm(THREAD);
        stringStream tempst;
        tempst.print("interpreter method <%s>\n"
                     " at bci %d, continuing at %d for thread " INTPTR_FORMAT,
                     METHOD->print_value_string(),
                     (int)(istate->bcp() - METHOD->code_base()),
                     (int)continuation_bci, p2i(THREAD));
        Exceptions::log_exception(except_oop, tempst);
      }
      // for AbortVMOnException flag
      Exceptions::debug_check_abort(except_oop);

      // Update profiling data.
      BI_PROFILE_ALIGN_TO_CURRENT_BCI();
      // 继续执行
      goto run;
    }
    if (log_is_enabled(Info, exceptions)) {
      ResourceMark rm;
      stringStream tempst;
      tempst.print("interpreter method <%s>\n"
             " at bci %d, unwinding for thread " INTPTR_FORMAT,
             METHOD->print_value_string(),
             (int)(istate->bcp() - METHOD->code_base()),
             p2i(THREAD));
      Exceptions::log_exception(except_oop, tempst);
    }
    // for AbortVMOnException flag
    Exceptions::debug_check_abort(except_oop);

    // 如果当前方法没有找到可以处理异常的方式，就把pending_exception重新设置回去，然后方法return，继续到caller方法，每次方法return，caller处都有一个method_resume机制，如果发现当前有pending_exception，同样会goto handle_exception处继续处理（在caller方法的栈帧下处理）
    THREAD->set_pending_exception(except_oop(), NULL, 0);
    goto handle_return;
  }
```

```c++
int Method::fast_exception_handler_bci_for(const methodHandle& mh, Klass* ex_klass, int throw_bci, TRAPS) {
	// 找到当前方法的异常表
  ExceptionTable table(mh());
  int length = table.length();
  constantPoolHandle pool(THREAD, mh->constants());
  for (int i = 0; i < length; i ++) {
    // 重新获取一下，因为这个时候是有可能发生GC的
    ExceptionTable table(mh());
    int beg_bci = table.start_pc(i);
    int end_bci = table.end_pc(i);
    // 找到的这个看是不是在catch的范围内
    if (beg_bci <= throw_bci && throw_bci < end_bci) {
      int handler_bci = table.handler_pc(i);
      int klass_index = table.catch_type_index(i);
      if (klass_index == 0) {
        return handler_bci;
      } else if (ex_klass == NULL) {
        return handler_bci;
      } else {
        // 看抛出的异常是否是声明的异常的子类
        Klass* k = pool->klass_at(klass_index, CHECK_(handler_bci));
        assert(k != NULL, "klass not loaded");
        // 如果是子类，就说明handler_bci位置的这条字节码就是catch的逻辑，直接跳转到这条指令继续执行
        if (ex_klass->is_subtype_of(k)) {
          return handler_bci;
        }
      }
    }
  }

  return -1;
}
```

以上这段就是我们方法如何抛出异常，同时如果当前方法不能处理异常，就会一层一层的往上抛的具体实现。那么如果有一个方法可以处理异常，也就是我们如果代码中有try…catch…finally代码块，对于我们的异常处理机制虚拟机会如何处理呢？

###catch…finnaly代码块

我们以一段Java代码为例

```java
public class ThrowExceptionTest {

    public static void main(String[] args) {
        try {
            int[] array = new int[]{};
            int i = array[0];
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            int abc = 123;
        }
    }

}
```

字节码：

```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=5, args_size=1
         0: iconst_0
         1: newarray       int
         3: astore_1
         4: aload_1
         5: iconst_0
         6: iaload
         7: istore_2
         8: bipush        123
        10: istore_1
        11: goto          32
        14: astore_1
        15: aload_1
        16: invokevirtual #3                  // Method java/lang/Exception.printStackTrace:()V
        19: bipush        123
        21: istore_1
        22: goto          32
        25: astore_3
        26: bipush        123
        28: istore        4
        30: aload_3
        31: athrow
        32: return
      Exception table:
         from    to  target type
             0     8    14   Class java/lang/Exception
             0     8    25   any
            14    19    25   any
			// 省略无用信息
}
```

关于这段代码编译之后的结果有以下几点注意：

1. 当需要处理异常的时候，通过Exception table来表达对异常的捕获，from和to就是捕获这些行数中抛出的异常，target是如果捕获就从target位置继续执行，type就是捕获的异常类型。
2. finnaly的代码块中的字节码被写了三遍。
3. 如果代码中有finally，那么catch的异常类型需要多一个any，因为无论是否捕获异常，都需要执行finally中的代码
4. 如果同时有catch和finally，还需要额外增加一个捕获，就是catch中的代码可能会继续抛出异常，所以也要增加一个catch的类型确保即使catch中抛出异常，finally的代码也能执行。

##invokedynamic

### 普通方法调用总结

1. 编译时要用**invokespecial**、**invokevirtual**、**invokestatic**、**invokeinterface**

2. 这四个字节码后都会对应一个常量池中的**ConstantPoolCacheEntry**下标，如：invokevirtual #2 

   > 注意这个下标编译时和运行时可能不一样的，因为类加载的过程中需要替换编译时的静态常量池，使用的替换方式就是字节码重写。

3. 先看这个**ConstantPoolCacheEntry**是否已经链接，如果没有链接就通过运行时链接

   这个运行时链接的过程中就是将符号引用（方法签名）转换为方法的指针，具体的做法就是先拿到类，然后在类上找方法（类加载时方法重排序就是为了这里做优化），这不是link_time_resolve_method，之后还会进行runtime_resolve_method，主要就是从vtable和itable中寻找方法（invokevirtual和invokeinterface），找到之后更新常量池的指针。

4. 找到方法，准备好参数，就新建一个栈帧，默认从解释执行（模版解释的解释执行）的方式开始执行（如果有编译优化过的方法就会从编译优化之后的方法开始执行）。

可以看到在这些方法的执行过程中，所有的内容都是编译时期可以确定的，比如调用哪个方法、这个方法在哪里都是编译时期就已经确定好的，唯一算得上一点点动态的就是vtable和itable两个表，但是这两个表也是类加载的时候生成的，所以也不能算是动态。

那么接下来我们就来看一下invokedynamic，为什么叫invokedynamic，他到底动态在什么地方？

###MethodHandle

在理解invokedynamic之前，我们先要理解一下方法具柄。方法具柄实际上是和Java反射类似的一个东西，只不过它比反射更底层。

方法句柄除了可以指向普通方法之外，一些静态方法、构造方法，甚至字段Field也是可以指向的（会变成Getter和Setter方法）。

方法句柄通过MethodHandles.Lookup类来完成，但是寻找的时候是要根据不同的方法类型来寻找（比如是findStatic，findVirtual，findInterface等）。但是有一个需要注意，方法句柄的访问权限不取决于方法句柄创建的位置，而是取决于Lookup对象的创建位置（其实就是Lookup只能看到自己这个作用域能看到的内容，并不是开了天眼所有作用域都能拿到）。

>  目前关于MethodHandle的使用只在Spring里面关于interface的default方法中看到过，应用的还不是很多。

###invokedynamic

那么invokedynamic和方法具柄其实设计在Java中的目的都是为了解决方法的绑定在编写完代码之后就已经确定的缺点。Java为了使用方法具柄和invokeDynamic的本质目的还是希望可以在代码运行时动态的去绑定方法。因此就出现了上面的方法具柄，同时为了支持invokeDynamic，Java还增加了一个动态调用点（Call Site）的概念，就是当我需要调用一个lamda函数时，我就在此处加入一个动态调用点，虚拟机执行的就是动态调用点，具体这个动态调用点对应的是哪个方法，其实虚拟机并不关心。

###CallSite

既然虚拟机关心的是动态调用点CallSite，那么自然就会有什么是动态调用点，动态调用点是如何创建的等问题。

- 什么是CallSite

  CallSite其实就是一个MethodHandleWrapper，本质上还是一个方法句柄。
  
- 虚拟机的CallSite有哪些典型的实现

  ConstantCallSite，VolatileCallSite，MutableCallSite

###invokeDynamic的执行流程

1. 在ConstantPool中找到BootstrapMethod（这个BootstrapMethod是一个方法句柄），由于此时找到的这个BootstrapMethod还是常量池中的符号链接，所以需要进行运行时链接。

   虚拟机中会构建一个MemberName类型（就是一个方法的符号链接）；然后通过SystemDictionary把MemberName对应的符号链接解析成对方法的引用；虚拟机会调用**MethodHandles.linkMethodHandleConstant（一个Java方法）**把前面解析到的符号引用变成一个MethodHandle类。

   到这里为止，BootstrapMethod的MethodHandle解析完成，虚拟机已经获得了Bootstrap方法的具体引用。

2. 有了BootstrapMethodHandle之后，虚拟机会继续准备一些参数，然后调用**MethodHandleNatives.linkCallSite**方法，由于bootstrap方法是**LamdaMetaFactory.metafactory**方法，那么调用bootstrap方法其实就是调用该方法，**LamdaMetaFactory.metafactory**方法会返回一个CallSite，这个CallSite就会被放在运行时常量池中，供后续调用方法的时候使用。

> 大家在看编译结果的时候要注意一点，就是不同的jdk版本对编译出来的字节码是有很大不同的，高版本的jdk对于invokeDynamic会有更多的优化。

###代码示例

上面这段描述比较抽象，第一次接触invokedynamic的原理最大的感觉就是头晕，需要结合jvm源码和jdk的源码进一步深入理解才行。下面结合一小段代码来更加实际的感受一下invokedynamic的原理。

下面的这个例子基于jdk8，因为jdk9+之后的版本又做了很多优化，会更绕一点。所以基于一个低版本来了解一下基本的流程。

```java
public class Dynamic {
    public static void main(String[] args) throws InterruptedException {
        String a = "123";
        Function<String, String> supplier2 = str -> str + a.toUpperCase();
        callLamda(supplier2);
    }

    private static void callLamda(Function<String, String> function) {
        function.apply("!23");
    }
}
```

下面是字节码

```
Classfile /Users/xiwang/Documents/course/out/production/course/invokedynamic/Dynamic.class
  Last modified 2019年7月10日; size 1903 bytes
  MD5 checksum 85fbfb8678daf8fcabe26ceecf49cc0d
  Compiled from "Dynamic.java"
public class invokedynamic.Dynamic
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #11                         // invokedynamic/Dynamic
  super_class: #12                        // java/lang/Object
  interfaces: 0, fields: 0, methods: 4, attributes: 3
Constant pool:
   #1 = Methodref          #12.#42        // java/lang/Object."<init>":()V
   #2 = String             #43            // 123
   #3 = InvokeDynamic      #0:#49         // #0:apply:(Ljava/lang/String;)Ljava/util/function/Function;
   #4 = Methodref          #11.#50        // invokedynamic/Dynamic.callLamda:(Ljava/util/function/Function;)V
   #5 = String             #51            // !23
   #6 = InterfaceMethodref #52.#53        // java/util/function/Function.apply:(Ljava/lang/Object;)Ljava/lang/Object;
   #7 = Class              #54            // java/lang/StringBuilder
   #8 = Methodref          #7.#42         // java/lang/StringBuilder."<init>":()V
   #9 = Methodref          #7.#55         // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #10 = Methodref          #7.#56         // java/lang/StringBuilder.toString:()Ljava/lang/String;
  #11 = Class              #57            // invokedynamic/Dynamic
  #12 = Class              #58            // java/lang/Object
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Linvokedynamic/Dynamic;
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               args
  #23 = Utf8               [Ljava/lang/String;
  #24 = Utf8               a
  #25 = Utf8               Ljava/lang/String;
  #26 = Utf8               supplier2
  #27 = Utf8               Ljava/util/function/Function;
  #28 = Utf8               LocalVariableTypeTable
  #29 = Utf8               Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;
  #30 = Utf8               Exceptions
  #31 = Class              #59            // java/lang/InterruptedException
  #32 = Utf8               callLamda
  #33 = Utf8               (Ljava/util/function/Function;)V
  #34 = Utf8               function
  #35 = Utf8               Signature
  #36 = Utf8               (Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;)V
  #37 = Utf8               lambda$main$0
  #38 = Utf8               (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #39 = Utf8               str
  #40 = Utf8               SourceFile
  #41 = Utf8               Dynamic.java
  #42 = NameAndType        #13:#14        // "<init>":()V
  #43 = Utf8               123
  #44 = Utf8               BootstrapMethods
  #45 = MethodHandle       6:#60          // REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #46 = MethodType         #61            //  (Ljava/lang/Object;)Ljava/lang/Object;
  #47 = MethodHandle       6:#62          // REF_invokeStatic invokedynamic/Dynamic.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #48 = MethodType         #63            //  (Ljava/lang/String;)Ljava/lang/String;
  #49 = NameAndType        #64:#65        // apply:(Ljava/lang/String;)Ljava/util/function/Function;
  #50 = NameAndType        #32:#33        // callLamda:(Ljava/util/function/Function;)V
  #51 = Utf8               !23
  #52 = Class              #66            // java/util/function/Function
  #53 = NameAndType        #64:#61        // apply:(Ljava/lang/Object;)Ljava/lang/Object;
  #54 = Utf8               java/lang/StringBuilder
  #55 = NameAndType        #67:#68        // append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #56 = NameAndType        #69:#70        // toString:()Ljava/lang/String;
  #57 = Utf8               invokedynamic/Dynamic
  #58 = Utf8               java/lang/Object
  #59 = Utf8               java/lang/InterruptedException
  #60 = Methodref          #71.#72        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #61 = Utf8               (Ljava/lang/Object;)Ljava/lang/Object;
  #62 = Methodref          #11.#73        // invokedynamic/Dynamic.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #63 = Utf8               (Ljava/lang/String;)Ljava/lang/String;
  #64 = Utf8               apply
  #65 = Utf8               (Ljava/lang/String;)Ljava/util/function/Function;
  #66 = Utf8               java/util/function/Function
  #67 = Utf8               append
  #68 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #69 = Utf8               toString
  #70 = Utf8               ()Ljava/lang/String;
  #71 = Class              #74            // java/lang/invoke/LambdaMetafactory
  #72 = NameAndType        #75:#79        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #73 = NameAndType        #37:#38        // lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #74 = Utf8               java/lang/invoke/LambdaMetafactory
  #75 = Utf8               metafactory
  #76 = Class              #81            // java/lang/invoke/MethodHandles$Lookup
  #77 = Utf8               Lookup
  #78 = Utf8               InnerClasses
  #79 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #80 = Class              #82            // java/lang/invoke/MethodHandles
  #81 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #82 = Utf8               java/lang/invoke/MethodHandles
{
  // 省略构造函数

  public static void main(java.lang.String[]) throws java.lang.InterruptedException;
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: ldc           #2                  // String 123
         2: astore_1
         3: aload_1
         4: invokedynamic #3,  0              // InvokeDynamic #0:apply:(Ljava/lang/String;)Ljava/util/function/Function;
         9: astore_2
        10: aload_2
        11: invokestatic  #4                  // Method callLamda:(Ljava/util/function/Function;)V
        14: return
      LineNumberTable:
        line 15: 0
        line 16: 3
        line 17: 10
        line 18: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  args   [Ljava/lang/String;
            3      12     1     a   Ljava/lang/String;
           10       5     2 supplier2   Ljava/util/function/Function;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
           10       5     2 supplier2   Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;
    Exceptions:
      throws java.lang.InterruptedException

  private static void callLamda(java.util.function.Function<java.lang.String, java.lang.String>);
    descriptor: (Ljava/util/function/Function;)V
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: ldc           #5                  // String !23
         3: invokeinterface #6,  2            // InterfaceMethod java/util/function/Function.apply:(Ljava/lang/Object;)Ljava/lang/Object;
         8: pop
         9: return
      LineNumberTable:
        line 21: 0
        line 22: 9
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0 function   Ljava/util/function/Function;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0      10     0 function   Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;
    Signature: #36                          // (Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;)V

  private static java.lang.String lambda$main$0(java.lang.String, java.lang.String);
    descriptor: (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    flags: (0x100a) ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: new           #7                  // class java/lang/StringBuilder
         3: dup
         4: invokespecial #8                  // Method java/lang/StringBuilder."<init>":()V
         7: aload_1
         8: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        11: aload_0
        12: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        15: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        18: areturn
      LineNumberTable:
        line 16: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      19     0     a   Ljava/lang/String;
            0      19     1   str   Ljava/lang/String;
}
SourceFile: "Dynamic.java"
InnerClasses:
  public static final #77= #76 of #80;    // Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #45 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #46 (Ljava/lang/Object;)Ljava/lang/Object;
      #47 REF_invokeStatic invokedynamic/Dynamic.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
      #48 (Ljava/lang/String;)Ljava/lang/String;

```

可以看到上面就是一小段的代码，但是编译出来的java字节码非常的长，尤其是常量池的长度变得非常夸张。

1. 当执行到invokedynamic指令时，会根据指令上的参数找到对应的bootstrapMethod。

   invokedynamic #3,  0              // InvokeDynamic #0:apply:(Ljava/lang/String;)Ljava/util/function/Function;

   这个指令的含义就是找到第零个BootstrapMethod

2. 找到的第零个BootstrapMethod就是

   0: #45 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles\$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
       Method arguments:
         \#46 (Ljava/lang/Object;)Ljava/lang/Object;
         \#47 REF_invokeStatic invokedynamic/Dynamic.lambda\$main$0:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
         \#48 (Ljava/lang/String;)Ljava/lang/String;

   虚拟机找到这段调用 ***MethodHandleNatives.linkMethodHandleConstant*** 方法来创建指向BootstrapMethod的MethodHandle（简单点可以理解成就是一个函数指针或者是Java反射中的一个Method）。

3. 解析指向方法的MethodHandle

   \#47 REF_invokeStatic invokedynamic/Dynamic.lambda\$main$0 (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;

   上面Bootstrap方法中还有一个MethodHandle，也是通过***MethodHandleNatives.linkMethodHandleConstant***方法来创建对应的MethodHandle。

4. 虚拟机主动调用***MethodHandleNatives.linkCallSite***方法来创建invokedynamic对应的CallSite，

   ```java
       static MemberName linkCallSite(Object callerObj,
                                      Object bootstrapMethodObj,
                                      Object nameObj, Object typeObj,
                                      Object staticArguments,
                                      Object[] appendixResult) {
           MethodHandle bootstrapMethod = (MethodHandle)bootstrapMethodObj;
           Class<?> caller = (Class<?>)callerObj;
           String name = nameObj.toString().intern();
           MethodType type = (MethodType)typeObj;
           if (!TRACE_METHOD_LINKAGE)
               return linkCallSiteImpl(caller, bootstrapMethod, name, type,
                                       staticArguments, appendixResult);
           return linkCallSiteTracing(caller, bootstrapMethod, name, type,
                                      staticArguments, appendixResult);
       }
   
       static MemberName linkCallSiteImpl(Class<?> caller,
                                          MethodHandle bootstrapMethod,
                                          String name, MethodType type,
                                          Object staticArguments,
                                          Object[] appendixResult) {
           CallSite callSite = CallSite.makeSite(bootstrapMethod,
                                                 name,
                                                 type,
                                                 staticArguments,
                                                 caller);
           if (callSite instanceof ConstantCallSite) {
               appendixResult[0] = callSite.dynamicInvoker();
               return Invokers.linkToTargetMethod(type);
           } else {
               appendixResult[0] = callSite;
               return Invokers.linkToCallSiteMethod(type);
           }
       }
   ```

   CallSite对应的方法

   ```java
   static CallSite makeSite(MethodHandle bootstrapMethod,
                            // Callee information:
                            String name, MethodType type,
                            // Extra arguments for BSM, if any:
                            Object info,
                            // Caller information:
                            Class<?> callerClass) {
       MethodHandles.Lookup caller = IMPL_LOOKUP.in(callerClass);
       CallSite site;
       try {
           Object binding;
           info = maybeReBox(info);
           if (info == null) {
               binding = bootstrapMethod.invoke(caller, name, type);
           } else if (!info.getClass().isArray()) {
               binding = bootstrapMethod.invoke(caller, name, type, info);
           } else {
               Object[] argv = (Object[]) info;
               maybeReBoxElements(argv);
             	// 这里就是调用bootstrapMethod，也就是LambdaMetafactory.metafactory方法
               switch (argv.length) {
               case 0:
                   binding = bootstrapMethod.invoke(caller, name, type);
                   break;
               case 1:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0]);
                   break;
               case 2:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0], argv[1]);
                   break;
               case 3:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0], argv[1], argv[2]);
                   break;
               case 4:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0], argv[1], argv[2], argv[3]);
                   break;
               case 5:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0], argv[1], argv[2], argv[3], argv[4]);
                   break;
               case 6:
                   binding = bootstrapMethod.invoke(caller, name, type,
                                                    argv[0], argv[1], argv[2], argv[3], argv[4], argv[5]);
                   break;
               default:
                   final int NON_SPREAD_ARG_COUNT = 3;  // (caller, name, type)
                   if (NON_SPREAD_ARG_COUNT + argv.length > MethodType.MAX_MH_ARITY)
                       throw new BootstrapMethodError("too many bootstrap method arguments");
                   MethodType bsmType = bootstrapMethod.type();
                   MethodType invocationType = MethodType.genericMethodType(NON_SPREAD_ARG_COUNT + argv.length);
                   MethodHandle typedBSM = bootstrapMethod.asType(invocationType);
                   MethodHandle spreader = invocationType.invokers().spreadInvoker(NON_SPREAD_ARG_COUNT);
                   binding = spreader.invokeExact(typedBSM, (Object)caller, (Object)name, (Object)type, argv);
               }
           }
           //System.out.println("BSM for "+name+type+" => "+binding);
           if (binding instanceof CallSite) {
               site = (CallSite) binding;
           }  else {
               throw new ClassCastException("bootstrap method failed to produce a CallSite");
           }
           if (!site.getTarget().type().equals(type))
               throw wrongTargetType(site.getTarget(), type);
       } catch (Throwable ex) {
           BootstrapMethodError bex;
           if (ex instanceof BootstrapMethodError)
               bex = (BootstrapMethodError) ex;
           else
               bex = new BootstrapMethodError("call site initialization exception", ex);
           throw bex;
       }
       return site;
   }
   ```

   LambdaMetafactory

   ```java
       public static CallSite metafactory(MethodHandles.Lookup caller,/*这个就是编译进去的Lookup*/
                                          String invokedName, /*这个是函数接口的那个方法*/
                                          MethodType invokedType,
                                          MethodType samMethodType,
                                          MethodHandle implMethod, /*这个就是对应方法*/
                                          MethodType instantiatedMethodType)
               throws LambdaConversionException {
           AbstractValidatingLambdaMetafactory mf;
           mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                                invokedName, samMethodType,
                                                implMethod, instantiatedMethodType,
                                                false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
           mf.validateMetafactoryArgs();
           return mf.buildCallSite();
       }
   ```

   这里执行完，会通过asm的字节码生成，生成一个动态的Java类，反编译动态生成的Java类如下：

   ```java
   package invokedynamic;
   
   import invokedynamic.Dynamic;
   import java.lang.invoke.LambdaForm;
   import java.util.function.Function;
   
   final class Dynamic$$Lambda$1 implements Function {
       private final String arg$1;
   
       private Dynamic$$Lambda$1(String string) {
           this.arg$1 = string;
       }
   
     	// 有这个方法的原因是我们的lambda方法有额外捕获的参数，如果没有捕获的参数就不需要这个方法。
   	  // 这里的参数string是函数式接口捕获的那个参数
       private static Function get$Lambda(String string) {
           return new Dynamic$$Lambda$1(string);
       }
   
     	// 这个注解的意思是如果抛出一个异常，那么异常的堆栈里面这个方法的栈帧需要被隐藏
       @LambdaForm.Hidden
       public Object apply(Object object) {
   	      // 这个apply方法其实就是函数式接口Function的方法
         	// 该方法的具体实现就是去调用Dynamic类中的lambda$main$0方法，就是上面字节码中动态生成的方法。
           return Dynamic.lambda$main$0((String)this.arg$1, (String)((String)object));
       }
   }
   ```

> - **上面说的 get$Lambda 函数是谁来调用的？**
>
> 创建出来的CallSite内部持有的MethodHandle就是这个get$Lambda方法
>
>   ```java
>try {
>    UNSAFE.ensureClassInitialized(innerClass);
>    return new ConstantCallSite(
>       MethodHandles.Lookup.IMPL_LOOKUP
>       .findStatic(innerClass, NAME_FACTORY/*这个就是get$Lambda的字符串常量*/, invokedType));
>    }
>    catch (ReflectiveOperationException e) {
>    throw new LambdaConversionException("Exception finding constructor", e);
>   }
>    ```
>   
>   - **什么是变量捕获？**
>
> ```java
>String a = "123";
>   // 这个就是变量捕获，我们的函数式接口捕获了一个参数a
>   Function<String, String> f = str -> str + a;
>   
>   // 这个方法就没有捕获额外的变量，也就是函数式编程中的纯函数
>   Function<String, String> f2 = str -> str;
>   ```
>   
>   - **有变量捕获的函数式方法是不是性能会差一些？**
>
> 讲道理会差一点，因为没有变量捕获的函数式接口生成的动态类其实是可以缓存起来的，因为没有额外的变量，而有捕获的就需要每次通过get$Lambda创建一个新的，但是只要虚拟机的优化开启，这些性能损耗基本可以忽略不计。
>
>   - **invokedynamic性能如何**
>
> 大家可以看到进行一次invokedynamic方法相比原生的方法调用要绕很大一个圈子，根据比较权威的benchmark结果，只要虚拟机开启方法内联和逃逸分析等优化，那么invokedynamic的性能和原生方法的性能基本一致，如果不开启这些优化，那么性能会相差3倍以上。
>
>   所以整体上不应该以性能不佳为理由不使用invokedynamic（良好的设计模式总是优先于性能优化）

由于invokedynamic的实现原理非常的绕，所以虚拟机的源码就没有贴上来了，包括jdk9+的一些让人摸不着头脑的改动也没有放过来，大家有兴趣可以进一步深入学习。

下一章是GC基本原理