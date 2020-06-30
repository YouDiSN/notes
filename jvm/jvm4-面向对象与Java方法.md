#面向对象与Java方法

在了解了什么是OOP/Klass之后，我们需要进一步了解透过OOP/Klass的机制，虚拟机是如何实现Java的面向对象的特性。

## 继承

Java的继承分为Field的继承和Method的继承，即子类会继承父类的字段和方法。那么这个继承在Java中是如何实现的呢？

### Field

***Klass***层面的继承，Klass层面主要通过几个字段来表达继承（包括实现接口），就是Klass中的

```c++
Array<Method*>* _methods;
Klass*      _super;
Array<Klass*>* _secondary_supers;
```

Klass只会包含我们在本类中书写的那些Fields，父类的Fields**不会拷贝**一份存在自己的Klass中。

***OOP***层面，OOP是一个实例化后的Java类在虚拟机中的实现，OOP层面实现Field的继承，就是会把父类的所有字段都在自己的OOP去拷贝一份，也就是说我们实例化了一个子类，虚拟机中只会存在这个子类一个OOP（不会同时存在子类OOP和父类OOP），也就是实例化只会实例化一个对象，不会实例化两个对象。

那么既然只有一个OOP，父类的实例字段会放在什么地方呢？

在一个OOP实例中，会把所有父类的字段全部拷贝一份到这个OOP中。

那么虚拟机中是如何做到的呢？其实就是在classFileParser中生成Klass的时候，会获取父类相关字段的大小，然后再加上自己的实例字段的大小，统统加起来，这个大小再分配内存的时候，就会根据这个大小来分配。

```c++
// classFileParser.cpp（3851）
void ClassFileParser::layout_fields(ConstantPool* cp,
                                    const FieldAllocationCount* fac,
                                    const ClassAnnotationCollector* parsed_annotations,
                                    FieldLayoutInfo* info,
                                    TRAPS) {
  // 省略一大坨逻辑
}
```

该方法主要需要理解的问题如下。

- **static字段分配在哪里？**

  static字段的初始偏移是写在instanceMirrorKlass（就是Java的一个Class在虚拟机中的Klass实现）中的，默认是0。由于每一个JavaClass（就是比如obj.getClass()返回的那个）对象本身也是一个OOP，那么这个类的static字段就是存在这个OOP上的。这个OOP对应的Klass是instanceMirrorKlass，也就是源码中经常出现的_java_mirror。

- **非static字段分配在哪里？继承如何体现？**

  非static的实例字段都是分配在该new出来的OOP实例上的，父类的字段在前，子类的字段在后（不考虑压缩等优化手段）。其实所谓的继承，就是子类会把所有父类的字段全部都分配一遍。我们思考如下代码

  ```java
  public class Father {
    public String name = "Dad";
  }
  
  public class Son extends Father {
    public String name = "Son";
  }
  
  public class Main {
    public static void main(String... args) {
      Son son = new Son();
      String sonName = son.name; // Son
      String fatherName = ((Father) son).name; // Dad
    }
  }
  ```

  上面这个示例代码中的name，虽然都是name，为什么我用的是同一个Son对象，但是当我强制类型转换过之后获取到的name值会不一样呢？原因就是子类也会把父类的字段分配一遍，当要获取字段的时候，虚拟机第一步是获取到对应的InstanceKlass对象，当我们强制类型转换之后类型从Son也变成了Father，所以在虚拟机中的偏移就从Son的name字段偏移变成Father的name字段的偏移，这也是为什么获取到的值不一样的原因。
  
- **如果子类继承了父类的字段，那么是如何找到的呢？**

  我们思考如下例子

  ```java
  public class Father {
    public String name = "Dad";
  }
  
  public class Son extends Father {
  }
  
  public class Main {
    public static void main(String... args) {
      Son son = new Son();
      String name = son.name; // Dad
    }
  }
  ```

  这是因为虚拟机在寻找对象的时候，优先从local对象中寻找，如果找不到，会从实现的接口中找，如果还是没有，会继续到父类去找。

  ```c++
  Klass* InstanceKlass::find_field(Symbol* name, Symbol* sig, fieldDescriptor* fd) const {
    // 自己身上找
    // 这个其实就是一个for循环，是否也可以用二分法优化？
    if (find_local_field(name, sig, fd)) {
      return const_cast<InstanceKlass*>(this);
    }
  	// 接口
    // 也是for循环
    { Klass* intf = find_interface_field(name, sig, fd);
      if (intf != NULL) return intf;
    }
    // 父类，相当于递归
    { Klass* supr = super();
      if (supr != NULL) return InstanceKlass::cast(supr)->find_field(name, sig, fd);
    }
    // 木有找到
    return NULL;
  }
  ```

  > 如果父类和接口同时有一个字段，按照虚拟机的实现其实是优先先读的接口的，但是实际上编译的时候会报错，所以我们应该避免这样的使用方式。当有继承关系时，每一个field都应该仔细设计。

###Method

其实方法没有做太多的处理，OOP中没有方法这个概念，所有的方法全部都是在Klass中的，每个Klass都只记录了自己的Method。

####VTable ITable

我们知道我们调用一个Java方法，有多态、重写等，那么jvm是如何实现方法的多态的，其原理就是VTable，同时Java接口方法的实现原理使用的则是ITable。

在Java中，一个VTable存放的内容的是自己的实例方法（注意不是静态和final）和父类的虚函数，而ITable则放实现接口的函数列表，在内存中，这两个表都是放在instanceKlass对象的最后面，获取方式和Field这种东西都差不多，也是通过偏移的方式来获取的。

VTable是一个表，这张表是又vtableEntry的元素组成的一个List，每一个vtableEntry都指向一个函数Method（存在Metaspace中）。如果发现子类重写了一个父类的方法，那么子类会覆盖vtable中同顺序的那个vtableEntry，否则就会新增一个。

#### miranda方法

miranda方法是一种特殊的方法。

> Note on Miranda methods: Let's say there is a class C that implements
>
> interface I, and none of C's superclasses implements I.
>
> Let's say there is an abstract method m in I that neither C
>
> nor any of its super classes implement (i.e there is no method of any access,
>
> with the same name and signature as m), then m is a Miranda method which is
>
> entered as a public abstract method in C's vtable.  From then on it should
>
> treated as any other public method in C for method over-ride purposes.

那么最后vtable是如何计算的呢？

```c++
// klassVtable.cpp（66）
void klassVtable::compute_vtable_size_and_num_mirandas(
    int* vtable_length_ret, int* num_new_mirandas,
    GrowableArray<Method*>* all_mirandas, const Klass* super,
    Array<Method*>* methods, AccessFlags class_flags, u2 major_version,
    Handle classloader, Symbol* classname, Array<InstanceKlass*>* local_interfaces,
    TRAPS) {
  NoSafepointVerifier nsv;

  // set up default result values
  int vtable_length = 0;

  // 先看父类的vtable大小
  vtable_length = super == NULL ? 0 : super->vtable_length();

  // 遍历每一个方法
  int len = methods->length();
  for (int i = 0; i < len; i++) {
    methodHandle mh(THREAD, methods->at(i));

    if (needs_new_vtable_entry(mh, super, classloader, classname, class_flags, major_version, THREAD)) {
      vtable_length += vtableEntry::size(); // we need a new entry
    }
  }

  GrowableArray<Method*> new_mirandas(20);
  // 就是看自己实现的接口和那些透传过来的接口（比如父类实现的接口），然后看哪些方法是实现了接口的，就放到miranda_list中。
  get_mirandas(&new_mirandas, all_mirandas, super, methods, NULL, local_interfaces,
               class_flags.is_interface());
  *num_new_mirandas = new_mirandas.length();

  // 如果不是接口，vtable的大小就是vtable + itable
  if (!class_flags.is_interface()) {
     vtable_length += *num_new_mirandas * vtableEntry::size();
  }

  // super == Null有且只有一种情况，就是java.lang.Object，这里的意思就是有人重写了java.lang.Object
  if (super == NULL && vtable_length != Universe::base_vtable_size()) {
    // 检查报错
  }

  // 最后计算的结果
  *vtable_length_ret = vtable_length;
}
```

```c++
// klassVtable.cpp（582）
bool klassVtable::needs_new_vtable_entry(const methodHandle& target_method,
                                         const Klass* super,
                                         Handle classloader,
                                         Symbol* classname,
                                         AccessFlags class_flags,
                                         u2 major_version,
                                         TRAPS) {
    if (class_flags.is_interface()) {
      // 接口并不需要vtables
	    return false;
  	}

  	if (target_method->is_final_method(class_flags) ||
        // final方法不需要新建一个vtable entry，因为final方法本身不会被重写，即使他是重写了父类的方法之后的，也可以简单的重用父类的entry
      (target_method()->is_private()) ||
      // 私有方法也不用进入vatable
      (target_method()->is_static()) ||
      // 静态方法也不用进入vtable
      (target_method()->name()->fast_compare(vmSymbols::object_initializer_name()) == 0)
      // <init> 类的初始化方法也不用
      ) {
	   	 return false;
	  }

  // 接口上的default方法不需要
  if (target_method()->method_holder() != NULL &&
      target_method()->method_holder()->is_interface()  &&
      !target_method()->is_abstract()) {
    assert(target_method()->is_default_method(),
           "unexpected interface method type");
    return false;
  }

  // 没有父类的时候就需要新增一个
  if (super == NULL) {
    return true;
  }

  // 方法作用域是Package private的方法总是需要一个新的entry，（why？）
  if (target_method()->is_package_private()) {
    return true;
  }

  // 查找父类看看我们是否需要新建一个，还是重写一个
  ResourceMark rm;
  Symbol* name = target_method()->name();
  Symbol* signature = target_method()->signature();
  const Klass* k = super;
  Method* super_method = NULL;
  InstanceKlass *holder = NULL;
  Method* recheck_method =  NULL;
  while (k != NULL) {
    // 看看有没有重写父类的方法
    super_method = InstanceKlass::cast(k)->lookup_method(name, signature);
    if (super_method == NULL) {
      break; // 终止，继续搜索miranda方法（下面介绍）
    }
    InstanceKlass* superk = super_method->method_holder();
    // 只会覆盖实例方法，所以需要忽略private的方法因为private和static的方法不支持重写
    if ((!super_method->is_static()) &&
       (!super_method->is_private())) {
      // 确认这个父类的方法确实可以被覆盖，比如是否是public、protected等。
      if (superk->is_override(super_method, classloader, classname, THREAD)) {
        // 因为父类已经有一个Entry了，我们就不需要新建了
        return false;
      }
      // 注意如果这个方法不能被覆盖，这里也不需要报错，因为父类有一个private方法A，子类完全可以有一个public方法A，这并不冲突。
    }

		// 如果>=Java7，就看父类的父类；言外之意就是java7之前不支持找父类的父类
    if (major_version >= VTABLE_TRANSITIVE_OVERRIDE_VERSION) {
      k = superk->super();
    } else {
      break;
    }
  }

  // 查看是否需要覆盖miranda方法
  const InstanceKlass *sk = InstanceKlass::cast(super);
  if (sk->has_miranda_methods()) {
    if (sk->lookup_method_in_all_interfaces(name, signature, Klass::find_defaults) != NULL) 		{
      return false;  // 如果是一个miranda方法，我们就不新建entry
    }
  }
  return true; // found no match; we need a new entry
}
```

上面这段源码只是计算了vtable的大小，算出来了一个size，具体使用则是在要分配instanceKlass对象内存的时候使用到的。

```c++
// klassFactory.cpp（182）
InstanceKlass* KlassFactory::create_from_stream(ClassFileStream* stream,
                                                Symbol* name,
                                                ClassLoaderData* loader_data,
                                                Handle protection_domain,
                                                const InstanceKlass* unsafe_anonymous_host,
                                                GrowableArray<Handle>* cp_patches,
                                                TRAPS) {
  // 省略
  ClassFileParser parser(stream,
                         name,
                         loader_data,
                         protection_domain,
                         unsafe_anonymous_host,
                         cp_patches,
                         ClassFileParser::BROADCAST, // publicity level
                         CHECK_NULL);
  // 之前讲的很多都是在ClassFileParser的逻辑中，这个类就是new出来之后去解析读取到的stream文件，然后将相关的信息储存起来，在下面真正去分配一个instanceKlass的时候会把这些储存起来的值用于分配这个instanceKlass对象。
  InstanceKlass* result = parser.create_instance_klass(old_stream != stream, CHECK_NULL);
  // 省略
}
```

```c++
// classFileParser.cpp（5482）
InstanceKlass* ClassFileParser::create_instance_klass(bool changed_by_loadhook, TRAPS) {
  if (_klass != NULL) {
    return _klass;
  }

  // 这里就是分配一个内存
  InstanceKlass* const ik =
    InstanceKlass::allocate_instance_klass(*this, CHECK_NULL);

  fill_instance_klass(ik, changed_by_loadhook, CHECK_NULL);

  if (ik->should_store_fingerprint()) {
    ik->store_fingerprint(_stream->compute_fingerprint());
  }

  ik->set_has_passed_fingerprint_check(false);
	// aot编译优化

  return ik;
}
```

```c++
// instanceKlass.cpp（359）
InstanceKlass* InstanceKlass::allocate_instance_klass(const ClassFileParser& parser, TRAPS) {
  // 这里就用到了前面算出来的各种size
  const int size = InstanceKlass::size(parser.vtable_size(),
                                       parser.itable_size(),
                                       nonstatic_oop_map_size(parser.total_oop_map_count()),
                                       parser.is_interface(),
                                       parser.is_unsafe_anonymous(),

  should_store_fingerprint(parser.is_unsafe_anonymous()));

  const Symbol* const class_name = parser.class_name();
  assert(class_name != NULL, "invariant");
  ClassLoaderData* loader_data = parser.loader_data();
  assert(loader_data != NULL, "invariant");

  InstanceKlass* ik;

  // Allocation
  if (REF_NONE == parser.reference_type()) {
    if (class_name == vmSymbols::java_lang_Class()) {
      // 如果是java.lang.Class类型，那么instanceKlass的类型应当是 InstanceMirrorKlass
      ik = new (loader_data, size, THREAD) InstanceMirrorKlass(parser);
    }
    else if (is_class_loader(class_name, parser)) {
      // 如果是ClassLoader，那么instanceKlass的类型应当是 InstanceClassLoaderKlass
      ik = new (loader_data, size, THREAD) InstanceClassLoaderKlass(parser);
    } else {
      // 普通的就是 InstanceKlass
      ik = new (loader_data, size, THREAD) InstanceKlass(parser, InstanceKlass::_misc_kind_other);
    }
  } else {
    // 不然就是引用类型（Reference），那么instanceKlass的类型应当是 InstanceRefKlass
    ik = new (loader_data, size, THREAD) InstanceRefKlass(parser);
  }
	// 省略检查逻辑

  return ik;
}
```

以上逻辑我们vtable前面计算了大小和分配到了内存，那么什么时候去填充这块数据呢？

就是在一个类链接的时候，

```c++
// instanceKlass.cpp（731）
bool InstanceKlass::link_class_impl(TRAPS) {
  // 843
  ClassLoaderData * loader_data = class_loader_data();
  if (!(is_shared() &&
        loader_data->is_the_null_class_loader_data())) {
    vtable().initialize_vtable(true, CHECK_false);
    itable().initialize_itable(true, CHECK_false);
  }
}
```

> 类链接的时候锁的粒度？
>
> 类初始化的时候锁的粒度？