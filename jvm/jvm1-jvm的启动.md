# 虚拟机的启动

入口文件：

main.c

```c
setEnv;
parseArgs;
return JLI_Launch(margc, margv,
                   jargc, (const char**) jargv,
                   0, NULL,
                   VERSION_STRING,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   jargc > 0,
                   const_cpwildcard, const_javaw, 0);
```

java.c（219）

```c
JNIEXPORT int JNICALL
JLI_Launch(int argc, char ** argv,              /* main argc, argv */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* UNUSED dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* unused */
        ) {
        	SelectVersion
        		// 选择java版本
        	CreateExecutionEnvironment
        		// 创建执行环境
        		// 设置JRE路径、JVM 路径、根据一些参数变量设置一些值
        	LoadJavaVM
        		// 加载JNI几个关键函数，比如JNI_CreateJavaVM，JNI_GetDefaultJavaVMInitArgs等
        	ParseArguments
        		// 解析变量，太多变量，详情可见java.c 1278行
        	设置一些变量
        	JVMInit // java_md_solinux.c
        }
```

java_md_solinux.c
```c
int
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,
        int argc, char **argv,
        int mode, char *what, int ret)
{
    ShowSplashScreen();
    return ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

java.c 2321

```c
int
ContinueInNewThread(InvocationFunctions* ifn, jlong threadStackSize,
                    int argc, char **argv,
                    int mode, char *what, int ret)
{
		// 设置栈深度
	  // 另起一个新线程来启动java虚拟机
    { /* Create a new thread to create JVM and invoke main method */
      JavaMainArgs args;
      int rslt;

      args.argc = argc;
      args.argv = argv;
      args.mode = mode;
      args.what = what;
      args.ifn = *ifn;

      // 这个就是会另起一个新线程去执行JavaMain这个函数
      rslt = ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
      /* If the caller has deemed there is an error we
       * simply return that, otherwise we return the value of
       * the callee
       */
      return (ret != 0) ? ret : rslt;
    }
}
```

java.c

```c
int JNICALL
JavaMain(void * _args)
{
  	// 创建java虚拟机
  	// 记录一些日志，以及一些校验虚拟机是否创建成功的工作
  	// 加载java的main方法所在的类
  	// 创建main方法所需要的参数
  	// 调用main方法
}
```

jni.cpp

```c++
static jint JNI_CreateJavaVM_inner(JavaVM **vm, void **penv, void *args) {
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
}
```

thread.cpp

```c++
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  // ...
  // Initialize global modules
  jint status = init_globals();
  // ... 
}
```

init.cpp

```c++
jint init_globals() {
  HandleMark hm;
  management_init(); // 初始化监控 PerfDataManager；比如虚拟机创建时间、销毁时间、cpu时间；还比如有多少线程、多少守护线程；还比如安全点的监控；还比如加载了多少类、卸载了多少类；
  bytecodes_init(); // jvm支持的所有字节码指令
  classLoader_init1(); // 初始化classloader监控、加载java library，bootstrap classPath维护到一个list中
  compilationPolicy_init(); // 指定使用哪个编译器，编译优化
  codeCache_init(); // 初始化codeCache
  VM_Version_init(); // 主、次版本号
  os_init_globals(); // 不知道干什么，看上去是操作系统初始化
  stubRoutines_init1(); // 平台相关的汇编处理，
  jint status = universe_init();  // dependent on codeCache_init and
                                  // stubRoutines_init1 and metaspace_init.
  																// 比如初始化java堆，加载类什么的
  if (status != JNI_OK)
    return status;

  gc_barrier_stubs_init();   // depends on universe_init, must be before interpreter_init
  													 // gc内存屏障等初始化
  interpreter_init();        // before any methods loaded 初始化解释器
  invocationCounter_init();  // before any methods loaded 调用次数计数器，主要是热点方法编译优化之类的
  accessFlags_init(); // public、private这种关键字
  templateTable_init(); // 模版解释器，字节码根据模版生成汇编
  InterfaceSupport_init();
  SharedRuntime::generate_stubs();
  universe2_init();  // dependent on codeCache_init and stubRoutines_init1
  javaClasses_init();// must happen after vtable initialization, before referenceProcessor_init
  referenceProcessor_init();
  jni_handles_init();
#if INCLUDE_VM_STRUCTS
  vmStructs_init();
#endif // INCLUDE_VM_STRUCTS

  vtableStubs_init();
  InlineCacheBuffer_init();
  compilerOracle_init();
  dependencyContext_init();

  if (!compileBroker_init()) {
    return JNI_EINVAL;
  }
  VMRegImpl::set_regName();

  if (!universe_post_init()) {
    return JNI_ERR;
  }
  stubRoutines_init2(); // note: StubRoutines need 2-phase init
  MethodHandles::generate_adapters();

#if INCLUDE_NMT
  // Solaris stack is walkable only after stubRoutines are set up.
  // On Other platforms, the stack is always walkable.
  NMT_stack_walkable = true;
#endif // INCLUDE_NMT

  // All the flags that get adjusted by VM_Version_init and os::init_2
  // have been set so dump the flags now.
  if (PrintFlagsFinal || PrintFlagsRanges) {
    JVMFlag::printFlags(tty, false, PrintFlagsRanges);
  }

  return JNI_OK;
}
```

universe.cpp

```c++
jint universe_init() {
  // 初始化java堆
  // 初始化metaspace等
}
```

#jvm目录结构

adlc：平台描述文件，差不多就是一个接口，不同的操作系统和cpu架构有不同的实现

aot：静态编译，前端编译（javac）、即时编译（JIT）、静态提前编译（AOT编译）。就是javac运行前生成字节码，即时编译运行时动态生成机器码，AOT在运行前尝试生成机器码。

asm：字节码生成

c1：c1编译器

ci：动态编译

**classfile：class文件解析，类的链接。比如java类的解析ClassFileParser**

code：机器码生成

compiler：动态编译

**gc：垃圾回收**

include：三个接口文件

**intepreter：解释器，解释执行java代码**

jfr：性能记录

jvmci：java编译接口

libadt：数据结构，dict，set，Vector Sets（vector是c++的一个数据结构，向量）

logging：日志

**memory：内存管理**

metaprogramming：元编程

**oops：oop/klass**

opto：c2编译

precompiled：c++文件导入了很多相同得依赖，一个依赖改变要改几百个文件，所以就让这几百个文件依赖同一个文件头

prims：对外接口

**runtime：运行时，比如线程，锁，java方法调用，编译优化和去优化等**

services：JMX，远程监控

utilities：工具类

下次内容会围绕虚拟机如何加载一个类，加载出来的Java类在虚拟机内部对应的数据结构，一个new出来的Java实例是什么样的关系等展开。