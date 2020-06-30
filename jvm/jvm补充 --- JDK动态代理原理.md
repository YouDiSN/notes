# jvm补充 --- 动态代理

我们经常会看到说Spring的动态代理有两种方式，一种是Java自己的基于借口的动态代理，还有一种是CGLIB的子类继承。但是我们自己很少会主动去写动态代理的代码，所以本章内容希望能简单展示如何使用动态代理，以及jdk基于接口动态代理的实现原理。

## 为什么需要动态代理

代理本身是设计模式模式的一种，通过对某个类或某些类的代理，可以在不改变原有代码的情况下，做一些增强，比如日志的记录、缓存，以及上次讲到反射的基本原理时，提到的DelegatingMethodAccessorImpl。但是静态代理，往往一个静态代理类只能代理一种特定的类，不具备通用型，通过动态代理，无需为每一种被代理类都写一遍代理逻辑的实现就可以实现。

比如在方法调用前后记录日志和cat打点，就可以使用一种通用的动态代理，给需要的方法自动添加，而不必每一个类都写一个代理类去做这件事情。

## 如何使用动态代理

- 需要一个被代理的接口，需要一个InvocationHandler的实现
- 通过Proxy.newProxyInstance的方式生成代理类
- 使用该代理类即可

```java
public interface Something {
  void doSth();
}

public static void main(String[] args) throws ClassNotFoundException {
  Class<Something> sc = Something.class;
  // 生成代理类
  Something target = (Something) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{sc}, getHandler());
  // 调用代理类的方法
  target.doSth();
}

private static InvocationHandler getHandler() {
  return (proxy, method, args) -> {
    // 被代理类每一次方法调用都会进入到这里，目前的实现就是简单的打印一些数据，实际上可以根据自己的需要修改不同的代码实现
    System.out.println("proxy class: " + proxy.getClass().getSimpleName());
    System.out.println("proxy method: " + method.getName());
    return null;
  };
}
```

以上这段代码就已经可以生成一个简单的动态代理了。代理类target的使用和普通我们自己new出来的类没有区别。

## JDK动态代理的原理

首先因为是动态代理，所以具体的被代理类是动态生成的，那么根据上面的例子，我们首先看一下这个被动态生成的target到底jdk给我们生成了什么东西。

还是通过arthas对类进行反编译后，有一下结果：

```java
/*
 * Decompiled with CFR.
 *
 * Could not load the following classes:
 *  com.youdisn.module.main.Something
 */
package com.sun.proxy.jdk.proxy1;

import com.youdisn.module.main.Something;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

// 首先这个类是继承自Proxy，同时实现了我们自己定义的Something接口
public final class $Proxy0 extends Proxy implements Something {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler invocationHandler) {
	      // 调用父类的构造函数
        super(invocationHandler);
    }

    static {
        try {
          	// 获取equals方法
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
          	// tostring
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
          	// 接口上的doSth
            m3 = Class.forName("com.youdisn.module.main.Something").getMethod("doSth", new Class[0]);
	          // hashcode
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
            return;
        }
        catch (NoSuchMethodException noSuchMethodException) {
            throw new NoSuchMethodError(noSuchMethodException.getMessage());
        }
        catch (ClassNotFoundException classNotFoundException) {
            throw new NoClassDefFoundError(classNotFoundException.getMessage());
        }
    }

    public final boolean equals(Object object) {
        try {
	          // 这个this.h就是InvocationHandler，m1就是equals方法
            return (Boolean)this.h.invoke(this, m1, new Object[]{object});
        }
        catch (Error | RuntimeException throwable) {
            throw throwable;
        }
        catch (Throwable throwable) {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final String toString() {
        try {
	          // 同上，略
            return (String)this.h.invoke(this, m2, null);
        }
        catch (Error | RuntimeException throwable) {
            throw throwable;
        }
        catch (Throwable throwable) {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final int hashCode() {
        try {
          	// 同上，略
            return (Integer)this.h.invoke(this, m0, null);
        }
        catch (Error | RuntimeException throwable) {
            throw throwable;
        }
        catch (Throwable throwable) {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final void doSth() {
        try {
          	// 这个就是我们接口上的方法，注意m3是类初始化的时候找到的接口上的方法
            this.h.invoke(this, m3, null);
            return;
        }
        catch (Error | RuntimeException throwable) {
            throw throwable;
        }
        catch (Throwable throwable) {
            throw new UndeclaredThrowableException(throwable);
        }
    }
}
```

通过分析代理生成的类，其实大致的原理我们已经很清楚了，就是生成一个Proxy类的子类，同时实现对应接口（可以是一个，也可以是多个，都没关系），然后每一个方法都调用InvocationHandler去实现具体的逻辑，这个生成类的作用如下

1. 初始化Method
2. 调用时把相关参数和调用的方法，委托给InvocationHandler，自己仅处理异常情况，其他一概不管

## 进一步深究原理

### Java部分

#### Proxy类的结构

```java
// 这个类很大，就不把所有的代码贴过来，仅选取有用的部分
public class Proxy implements java.io.Serializable {
  protected InvocationHandler h;
  
  protected Proxy(InvocationHandler h) {
    Objects.requireNonNull(h);
    this.h = h;
  }
  
  private static final class ProxyBuilder {
    // 这个builder里面实现了比较多的逻辑
  }
}
```

Proxy里面有一个ProxyBuilder的子类，这个子类实现了比较多的生成代理的相关逻辑，大致做的事情如下

- 确定生成类的包名、以及所属模块（jdk9之后），比如有些接口不一定是public的，可能是package-private的，那么生成类的包名就不能是默认的“com.sun.proxy”
- 通过一个自增的数字确定类名
- 通过字节码生成的方式生成类的字节码，生成具体方式就是先按照jvm字节码的规则生成对应的byte[]，然后通过Files.write把这个byte[]数组写到对应的路径上（路径就是前面决定包名的原因）
- 通过Unsafe调用jvm的能力，把生成的字节码变成一个java运行时的类（和反射的时候做法类似），然后缓存该类
- 然后获取该类的Constructor，反射调用该Constructor的newInstance方法，生成类的实例。

### jvm部分



以上部分都是从Java的视角看到的结果，我通过调用jdk的方法，会动态生成一个类，然后也看到了这个类的具体实现，但是作为一个jvm系列的分享，我们还需要把目光放到jvm中，这个动态类是如何生成的，jdk是如何处理这个动态生成的类的。

其实jdk动态代理涉及jvm的部分只有一个方法，

```java
Class<?> pc = UNSAFE.defineClass(proxyName, proxyClassFile,
                                                 0, proxyClassFile.length,
                                                 loader, null);

// 最后调用的是这个方法
// 这里传入的参数有必要解释一下
// name：类名
// b：类的字节码数组
// off：就是b中的起始偏移；通常是0
// len：就是从off的位置读取多少长度作为类的字节码；通常是b.length
// loader：ClassLoader
// protectionDomain：不用管
public native Class<?> defineClass0(String name, byte[] b, int off, int len,
                                        ClassLoader loader,
                                        ProtectionDomain protectionDomain);
```

```c++
UNSAFE_ENTRY(jclass, Unsafe_DefineClass0(JNIEnv *env, jobject unsafe, jstring name, jbyteArray data, int offset, int length, jobject loader, jobject pd)) {
  ThreadToNativeFromVM ttnfv(thread);

  return Unsafe_DefineClass_impl(env, name, data, offset, length, loader, pd);
} UNSAFE_END
```

具体实现如下

```c++
static jclass Unsafe_DefineClass_impl(JNIEnv *env, jstring name, jbyteArray data, int offset, int length, jobject loader, jobject pd) {
  // 省略其他

  // 这里就是生成的逻辑
  result = JVM_DefineClass(env, utfName, loader, body, length, pd);

  if (utfName && utfName != buf) {
    FREE_C_HEAP_ARRAY(char, utfName);
  }

 free_body:
  FREE_C_HEAP_ARRAY(jbyte, body);
  return result;
}
```

DefineClass的具体实现：

```c++
static jclass jvm_define_class_common(JNIEnv *env, const char *name,
                                      jobject loader, const jbyte *buf,
                                      jsize len, jobject pd, const char *source,
                                      TRAPS) {
  if (source == NULL)  source = "__JVM_DefineClass__";

  JavaThread* jt = (JavaThread*) THREAD;

  PerfClassTraceTime vmtimer(ClassLoader::perf_define_appclass_time(),
                             ClassLoader::perf_define_appclass_selftime(),
                             ClassLoader::perf_define_appclasses(),
                             jt->get_thread_stat()->perf_recursion_counts_addr(),
                             jt->get_thread_stat()->perf_timers_addr(),
                             PerfClassTraceTime::DEFINE_CLASS);

  // 统计信息
  if (UsePerfData) {
    ClassLoader::perf_app_classfile_bytes_read()->inc(len);
  }

  // Since exceptions can be thrown, class initialization can take place
  // if name is NULL no check for class name in .class stream has to be made.
  TempNewSymbol class_name = NULL;
  if (name != NULL) {
    // 检查类名是不是过长，max_length= 1 << 16 - 1，所以一般情况不会过长
    const int str_len = (int)strlen(name);
    if (str_len > Symbol::max_length()) {
      // 如果类名过长就抛出异常
      Exceptions::fthrow(THREAD_AND_LOCATION,
                         vmSymbols::java_lang_NoClassDefFoundError(),
                         "Class name exceeds maximum length of %d: %s",
                         Symbol::max_length(),
                         name);
      return 0;
    }
    // 把类名变成一个symbol
    class_name = SymbolTable::new_symbol(name, str_len, CHECK_NULL);
  }

  ResourceMark rm(THREAD);
  // 根据byte数组变成一个文件流
  ClassFileStream st((u1*)buf, len, source, ClassFileStream::verify);
  // 找到对应的class_loader的具柄对象
  Handle class_loader (THREAD, JNIHandles::resolve(loader));
  // 统计信息
  if (UsePerfData) {
    is_lock_held_by_thread(class_loader,
                           ClassLoader::sync_JVMDefineClassLockFreeCounter(),
                           THREAD);
  }
  Handle protection_domain (THREAD, JNIHandles::resolve(pd));
  // 通过SystemDictionary解析stream信息，和普通的类加载逻辑相同
  // resolve_from_stream可以看之前类加载部分的分析，这里不赘述了。
  Klass* k = SystemDictionary::resolve_from_stream(class_name,
                                                   class_loader,
                                                   protection_domain,
                                                   &st,
                                                   CHECK_NULL);

  // 日志
  if (log_is_enabled(Debug, class, resolve) && k != NULL) {
    trace_class_resolution(k);
  }

  // 把结果放到当前线程的栈上，作为结果返回
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
}
```

综上源码分析，其实jvm做的事情就是获取到字节码之后，通过加载的过程走一遍，然后生成的新类放到当前线程栈上返回即可，和一个普通的类加载其实没有太大的区别。