#jvm补充 --- 反射（补充内容）

反射是我们一直使用的Java特性，但是更多的我们可能集中在如何使用反射以及如何通过反射更快捷的实现一些功能。

所以反射虽然不全是jvm相关的内容，但是由于反射的重要性以及相关java实现中大量使用了native代码，因此我们也有必要深入探究一下反射的原理，以便能更好的掌握。

由于Class上通过反射可以获取的内容相当多，包括interface、field、method、constructor、annotation等，但是虽然种类非常多，他们背后的原理大致都相同，所以我们仅以一个Method为例来了解背后的原理，其他的原理基本类似。

假设我们有一个待反射的类TargetClass如下

```java
public class TargetClass {

    private String name;

    private int age;

    public TargetClass(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

如果我们希望从这个类上通过反射获取一些相关的信息，通常我们的代码是这样写的：

```java
// 以Method为例，其他信息基本类似
Method[] methods = TargetClass.class.getDeclaredMethods();
```

反射的基本套路就是先获取类Class对象，然后从Class对象上通过提供的各种方法获取对应的内容。

那么我们详细看一下getDeclaredMethods的具体实现

```java
public Method[] getDeclaredMethods() throws SecurityException {
  // 相关的安全检查，阅读源码时可以忽略
  SecurityManager sm = System.getSecurityManager();
  if (sm != null) {
    checkMemberAccess(sm, Member.DECLARED, Reflection.getCallerClass(), true);
  }
  // 重点是这里的privateGetDeclaredMethods方法，false是publicOnly，代表所有的方法
  return copyMethods(privateGetDeclaredMethods(false));
}
```

```java
private Method[] privateGetDeclaredMethods(boolean publicOnly) {
  Method[] res;
  // 这个东西可以理解成是一个缓存DTO，所有反射相关的信息获取一次之后就放到这个对象上缓存起来
  ReflectionData<T> rd = reflectionData();
  // 这里的逻辑基本上就是cache的逻辑
  if (rd != null) {
    res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
    if (res != null) return res;
  }
  // 这里就是找对应的方法
  res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
  // 这里是找到之后更新缓存
  if (rd != null) {
    if (publicOnly) {
      rd.declaredPublicMethods = res;
    } else {
      rd.declaredMethods = res;
    }
  }
  return res;
}

// 这个方法是一个native的方法，我们需要看openjdk的实现
private native Method[]      getDeclaredMethods0(boolean publicOnly);
```

##从jvm获取Method信息

下面就是openjdk里面对应的实现

![image-20200204200438649](/Users/xiwang/Desktop/分享/jvm/img/jvm-补充内容反射.png)

```c++
JVM_ENTRY(jobjectArray, JVM_GetClassDeclaredMethods(JNIEnv *env, jclass ofClass, jboolean publicOnly))
{
  JVMWrapper("JVM_GetClassDeclaredMethods");
  return get_class_declared_methods_helper(env, ofClass, publicOnly,
                                           /*want_constructor*/ false,
/*这个reflect_Method_klass就是Method这个类*/              SystemDictionary::reflect_Method_klass(),
                                           THREAD);
}
JVM_END
  
  
```

```c++
static jobjectArray get_class_declared_methods_helper(
                                  JNIEnv *env,
                                  jclass ofClass, jboolean publicOnly,
                                  bool want_constructor,
                                  Klass* klass, TRAPS) {

  JvmtiVMObjectAllocEventCollector oam;

  // 处理primitive类型以及数组类型
  if (java_lang_Class::is_primitive(JNIHandles::resolve_non_null(ofClass))
      || java_lang_Class::as_Klass(JNIHandles::resolve_non_null(ofClass))->is_array_klass()) {
    // Return empty array
    oop res = oopFactory::new_objArray(klass, 0, CHECK_NULL);
    return (jobjectArray) JNIHandles::make_local(env, res);
  }

  // 把传入的类转换成虚拟机内部的InstanceKlass
  // InstanceKlass就是之前说的oop-klass其中的klass
  InstanceKlass* k = InstanceKlass::cast(java_lang_Class::as_Klass(JNIHandles::resolve_non_null(ofClass)));

  // Ensure class is linked
  k->link_class(CHECK_NULL);

  // 从klass上获取所有的method
  Array<Method*>* methods = k->methods();
  int methods_length = methods->length();

  // 这里先保存所有的Method的id
  ResourceMark rm(THREAD);
  GrowableArray<int>* idnums = new GrowableArray<int>(methods_length);
  int num_methods = 0;

  // 这里就是需要return的方法先保存id
  // 这里要先保存id的原因是，java允许动态的redefine方法
  // 所以为了防止返回redefine之前的方法，这里先记录id，然后即便redefine了也可以返回redefine之后的方法
  for (int i = 0; i < methods_length; i++) {
    methodHandle method(THREAD, methods->at(i));
    if (select_method(method, want_constructor)) {
      if (!publicOnly || method->is_public()) {
        idnums->push(method->method_idnum());
        ++num_methods;
      }
    }
  }

  // Allocate result
  objArrayOop r = oopFactory::new_objArray(klass, num_methods, CHECK_NULL);
  objArrayHandle result (THREAD, r);

  // 这里就是根据id重新再找一次
  for (int i = 0; i < num_methods; i++) {
    methodHandle method(THREAD, k->method_with_idnum(idnums->at(i)));
    if (method.is_null()) {
      // Method may have been deleted and seems this API can handle null
      // Otherwise should probably put a method that throws NSME
      result->obj_at_put(i, NULL);
    } else {
      oop m;
      // 这里就是把找到的方法变成一个Java对象
      if (want_constructor) {
        m = Reflection::new_constructor(method, CHECK_NULL);
      } else {
        // 这里Reflection具体做法其实很简单，new一个Method对象，然后set对应的属性
        m = Reflection::new_method(method, false, CHECK_NULL);
      }
      // 把结果放到数组里
      result->obj_at_put(i, m);
    }
  }

  // 这个JNIHandles::make_local就是把返回的结果放到JavaThread的栈帧上，因为Java方法是基于栈的，把jvm的结果放到栈上，后面的java方法就可以直接获取了
  return (jobjectArray) JNIHandles::make_local(env, result());
}
```

到了这里虚拟机已经从运行时的Klass对象中找到对应的c++ Method对象，然后封装成java Method对象并塞到调用线程的栈上，以便后面的java代码可以继续运行。

## 反射方法的调用

通过反射找到对应的Method对象之后，往往我们还需要进行调用，

```java
TargetClass t = new TargetClass("1", 1);
Method[] methods = TargetClass.class.getDeclaredMethods();
Method getName = methods[0];
Object result = getName.invoke(t);
```

上面这一小段代码就是通过反射获取getName方法（例子中的代码写的比较简单，实际应用不要通过数组下标来获取方法，之前讲oop-klass的时候有说过method数组的顺序和书写顺序是不同的）。

那么我们来重点看一下这里的invoke的实现。

```java
public Object invoke(Object obj, Object... args) throws IllegalAccessException, IllegalArgumentException, InvocationTargetException
{
  // 检查，直接忽略
  if (!override) {
    Class<?> caller = Reflection.getCallerClass();
    checkAccess(caller, clazz,
                Modifier.isStatic(modifiers) ? null : obj.getClass(),
                modifiers);
  }
  // 获取MethodAccessor，不论是缓存还是新建一个
  MethodAccessor ma = methodAccessor;             // read volatile
  if (ma == null) {
    ma = acquireMethodAccessor();
  }
  // 然后通过MethodAccessor来调用具体的method方法
  return ma.invoke(obj, args);
}
```

那么这里有两个问题，第一个是MethodAccessor是如何创建的，还有一个是MethodAccessor的invoke方法的具体实现。

### 创建MethodAccessor

```java
private MethodAccessor acquireMethodAccessor() {
  // First check to see if one has been created yet, and take it
  // if so
  MethodAccessor tmp = null;
  // 这里的root我理解只有method copy的时候才会被赋值
  if (root != null) tmp = root.getMethodAccessor();
  if (tmp != null) {
    methodAccessor = tmp;
  } else {
    // 创建一个新的
    tmp = reflectionFactory.newMethodAccessor(this);
    setMethodAccessor(tmp);
  }

  return tmp;
}

public MethodAccessor newMethodAccessor(Method method) {
  // 检查jvm是否初始化完毕
  checkInitted();

  // 检查是否是CallerSensitive，这个东西主要是为了防止利用多重反射找漏洞的，我们一般直接认为false即可
  if (Reflection.isCallerSensitive(method)) {
    Method altMethod = findMethodForReflection(method);
    if (altMethod != null) {
      method = altMethod;
    }
  }

  // use the root Method that will not cache caller class
  Method root = langReflectAccess.getRoot(method);
  if (root != null) {
    method = root;
  }

  // noInflation是是否直接使用字节码生成，默认是false
  // 这个地方也可以设置为true，按照官方的解释是如果直接使用字节码生成，第一次反射的性能会差3-4倍，但是后续频繁运行后可以快20倍，但是考虑很多很多类的反射方法初始化的时候只会调用一次，所以不用反而会让性能更快
  if (noInflation && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
    return new MethodAccessorGenerator().
      generateMethod(method.getDeclaringClass(),
                     method.getName(),
                     method.getParameterTypes(),
                     method.getReturnType(),
                     method.getExceptionTypes(),
                     method.getModifiers());
  } else {
    // 这里就用NativeMethodAccessorImpl
    NativeMethodAccessorImpl acc =
      new NativeMethodAccessorImpl(method);
    // DelegatingMethodAccessorImpl就是一个代理，这里的代理设置的很巧妙，后面解释，这里的设计方式值得我们学习
    DelegatingMethodAccessorImpl res =
      new DelegatingMethodAccessorImpl(acc);
    // native的地方设置parent是这个代理
    acc.setParent(res);
    return res;
  }
}
```

所以对于大部分我们自己写的代码而言，我们通过反射获取的MethodAccessor都是NativeMethodAccessorImpl + DelegatingMethodAccessorImpl的一个组合，我们先来解释为什么不直接返回NativeMethodAccessorImpl而需要外面再包一层Delegating。

我们先看一下NativeMethodAccessorImpl的实现

```java
class NativeMethodAccessorImpl extends MethodAccessorImpl {
    private final Method method;
    private DelegatingMethodAccessorImpl parent;
    private int numInvocations;

    NativeMethodAccessorImpl(Method method) {
        this.method = method;
    }

    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        // 当调用次数到了一定次数之后，ReflectionFactory.inflationThreshold = 15
	      // 会进行if内的逻辑
        if (++numInvocations > ReflectionFactory.inflationThreshold()
                && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            MethodAccessorImpl acc = (MethodAccessorImpl)
                new MethodAccessorGenerator().
                    generateMethod(method.getDeclaringClass(),
                                   method.getName(),
                                   method.getParameterTypes(),
                                   method.getReturnType(),
                                   method.getExceptionTypes(),
                                   method.getModifiers());
            parent.setDelegate(acc);
        }

	      // 假设我们if没有进入，我们这里是直接调用native的方法，是jvm实现的
        return invoke0(method, obj, args);
    }

    void setParent(DelegatingMethodAccessorImpl parent) {
        this.parent = parent;
    }

    private static native Object invoke0(Method m, Object obj, Object[] args);
}
```

可以看到这个了类的具体实现，就是每次通过反射调用，有一个计数器+1，当比15大的时候开启一个优化（我们暂时不管什么优化），否则就调用native的方法。

但是调用native的方法以为着要通过jni的接口来访问，而调用jni的方法是相对性能比较差的，所以才有了上面计数器的优化，计数器内的的优化方式就是如果调用比较频繁，就用字节码生成的方式生成一个类。因为前面说了jni访问次数比较少的情况下性能还行，但是字节码生成的方式访问在之后的性能有20倍的提升。

所以上面if条件里的优化是这样的，由于外层获取到的是DelegatingMethodAccessorImpl，这个代理类代理了NativeMethodAccessorImpl。而NativeMethodAccessorImpl一旦调用次数到达一定次数之后开启优化，会生成一个新的类（字节码生成的方式生成的动态类），然后把DelegatingMethodAccessorImpl的代理从代理NativeMethodAccessorImpl修改为代理到新的动态类。

这就是为什么MethodAccessor要有一层代理的原因，底层做一些优化的时候对上层就可以做到无感。

那么最后，我们要看一下既然是字节码生成，那么到给我们生成了什么东西出来。

通过arthas可以反编译得到这个生成的类。

```java
package jdk.internal.reflect;

import java.lang.reflect.InvocationTargetException;
import jdk.internal.reflect.MethodAccessorImpl;
import learn.reflect.TargetClass;

public class GeneratedMethodAccessor1
extends MethodAccessorImpl {
    /*
     * Loose catch block
     * Enabled aggressive block sorting
     * Enabled unnecessary exception pruning
     * Enabled aggressive exception aggregation
     * Lifted jumps to return sites
     */
    public Object invoke(Object object, Object[] arrobject) throws InvocationTargetException {
	      // 调用空判
        if (object == null) {
            throw new NullPointerException();
        }
        try {
	          // 参数空判
            if (arrobject != null && arrobject.length != 0) {
                throw new IllegalArgumentException();
            }
	          // 直接调用getName方法
            return ((TargetClass)object).getName();
        }
        catch (ClassCastException | NullPointerException runtimeException) {
            throw new IllegalArgumentException(super.toString());
        }
        catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
    }
}
```

上面这个就是通过字节码生成的类反编译之后的结果，其实我们可以看到，其实就是反射调用getName方法，修改为生成一个类，专门调用getName方法，仅此而已。