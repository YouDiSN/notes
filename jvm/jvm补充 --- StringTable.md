#jvm补充 --- StringTable

在jvm中，所有的String本身是一个Java对象，这些对象存在堆中，这点没有问题。但是String作为一个特殊的对象，所有的Java String对象又是存储在StringTable中的。这个StringTable虚拟机全局只有一个，先看一下StringTable的基本结构。

```c++
class StringTable : public CHeapObj<mtSymbol>{
  // 静态指针
  static StringTable* _the_table;
  static volatile bool _alt_hash;

private:

  // 这个就是内部持有的Map结构
  StringTableHash* _local_table;
  size_t _current_size;
  volatile bool _has_work;
  // Set if one bucket is out of balance due to hash algorithm deficiency
  volatile bool _needs_rehashing;

  OopStorage* _weak_handles;

  volatile size_t _items_count;
}

typedef ConcurrentHashTable<WeakHandle<vm_string_table_data>,
                            StringTableConfig, mtSymbol> StringTableHash;
```

可以看到，StringTable内部主要是维护了一个StringTableHash，这个StringTableHash其实就是一个ConcurrentHashTable。

之前分析过String本身的一些特性，那么接着String后面着重分析一下StringTable的行为。

##StringTable添加

之前讲过，当我们第一次用到String对象的时候，会把这个String对应的Symbol从静态常量池中拿出来解析，然后放到StringTable中，源码如下：

```c++
oop ConstantPool::string_at_impl(const constantPoolHandle& this_cp, int which, int obj_index, TRAPS) {
  // If the string has already been interned, this entry will be non-null
  oop str = this_cp->resolved_references()->obj_at(obj_index);
  assert(!oopDesc::equals(str, Universe::the_null_sentinel()), "");
  if (str != NULL) return str;
  Symbol* sym = this_cp->unresolved_string_at(which);
  // 这里就是放到StringTable中
  str = StringTable::intern(sym, CHECK_(NULL));
  this_cp->string_at_put(which, obj_index, str);
  assert(java_lang_String::is_instance(str), "must be string");
  return str;
}
```

那么今天就来分析一下这个intern方法到底做了些什么事情。

```c++
oop StringTable::intern(Symbol* symbol, TRAPS) {
  if (symbol == NULL) return NULL;
  ResourceMark rm(THREAD);
  int length;
  // 首先先把Symbol变成一个utf-8编码的内容
  jchar* chars = symbol->as_unicode(length);
  Handle string;
  // 把转换后得到的内容再调用intern
  oop result = intern(string, chars, length, CHECK_NULL);
  return result;
}

jchar* Symbol::as_unicode(int& length) const {
  Symbol* this_ptr = (Symbol*)this;
  // 通过UTF-8编码找到长度
  length = UTF8::unicode_length((char*)this_ptr->bytes(), utf8_length());
  // 分配一块连续的char类型内存，注意这块内存是在ThreadLocal上的ResourceArea，可以理解成是一个当前线程的临时内存池，主要放一些中间变量做中转
  jchar* result = NEW_RESOURCE_ARRAY(jchar, length);
  if (length > 0) {
    // 把Symble变成一个unicode编码字符串
    UTF8::convert_to_unicode((char*)this_ptr->bytes(), result, length);
  }
  // 返回unicode编码内容
  return result;
}
```

上面这段的主要作用就是Symbol因为是在常量池中的，现在把这个常量转化为unicode编码的一块连续内存，也可以理解成数组。然后得到的内容继续调用intern方法。

```c++

oop StringTable::intern(Handle string_or_null_h, const jchar* name, int len, TRAPS) {
  // 因为这是一个String，所以调用hash_code方法来获取字符串的hash。
  // 注意！！
  // 这里的这个java_lang_String::hash_code的实现方式和Java中String对象的hashcode方法实现是一样的，但是是c++的代码。我个人认为这两个方法的实现必须确保一样。
  unsigned int hash = java_lang_String::hash_code(name, len);
  // 上面有了hash，就在StringTable中找找看是不是已经存在过同样的String了
  oop found_string = StringTable::the_table()->lookup_shared(name, len, hash);
  if (found_string != NULL) {
    return found_string;
  }
  if (StringTable::_alt_hash) {
    hash = hash_string(name, len, true);
  }
  // 如果没有找到，就主动插入到StringTable中。
  return StringTable::the_table()->do_intern(string_or_null_h, name, len,
                                             hash, CHECK_NULL);
}
```

这里就是我们经常说的，Java的String，在jvm级别，同样的String只会存在一份，因为在StringTable的层面，每次都会先去找现有的String是不是已经存在，如果存在就直接返回。

下面就是do_intern的具体实现了。

```c++
oop StringTable::do_intern(Handle string_or_null_h, const jchar* name,
                           int len, uintx hash, TRAPS) {
  HandleMark hm(THREAD);  // cleanup strings created
  Handle string_h;

  // 创建一个Handle
  if (!string_or_null_h.is_null()) {
    string_h = string_or_null_h;
  } else {
    string_h = java_lang_String::create_from_unicode(name, len, CHECK_NULL);
  }

  // Deduplicate the string before it is interned. Note that we should never
  // deduplicate a string after it has been interned. Doing so will counteract
  // compiler optimizations done on e.g. interned string literals.
  Universe::heap()->deduplicate_string(string_h());

  StringTableLookupOop lookup(THREAD, hash, string_h);
  StringTableGet stg(THREAD);

  bool rehash_warning;
  do {
    // 这个就是在StringTableHash中寻找这个值
    if (_local_table->get(THREAD, lookup, stg, &rehash_warning)) {
      // 是否需要rehash
      update_needs_rehash(rehash_warning);
      // 既然找到了，就返回对应的oop即可。
      return stg.get_res_oop();
    }
    // 如果没有找到就创建一个WeakHandle
    WeakHandle<vm_string_table_data> wh = WeakHandle<vm_string_table_data>::create(string_h);
    // 尝试插入
    if (_local_table->insert(THREAD, lookup, wh, &rehash_warning)) {
      update_needs_rehash(rehash_warning);
      return wh.resolve();
    }
  } while(true);
}
```

## 其他特性

### StringTable初始化容量

StringTable的大小是可以配置的，默认大小为：

```c++
const size_t defaultStringTableSize = NOT_LP64(1024) LP64_ONLY(65536);

const size_t minimumStringTableSize = 128;

// 2^24 is max size
const size_t END_SIZE = 24;
```

非64位机器大小为1024，64位机器位65536。默认最小为128。最大为2^24^。

###WeakHandle

还有一个点需要注意的是，在StringTableHash中，key是一个WeakHandle。

这个也很好理解，我们肯定不会希望整个jvm的内存里面全部都是各种各样奇奇怪怪的String，所以在GC的时候通过WeakHandle就可以很好的回收那些没用的String。

##静态空字符串

所以综上分析，当我们写一行类似这样的Java代码时，

```java
String str = "str";
```

会把这样的字符串尝试注册到StringTable中，所以这样的代码有可能会返回一个新的对象，也有可能返回的是已经存在的对象。

所以我们之前有些类似这样的代码其实是没有意义的：

```java
public interface StringUtils {
  public final static String EMPTY_STRING = "";
}
```

就是通过静态变量申明一个空字符串，然后再需要使用空字符串的地方使用这个对象。即使你直接写成""，其实在jvm里还是同一个对象，因此没有太多本质的性能差别，如果硬要说，就是第一次执行的时候会走一遍静态常量池的解析。