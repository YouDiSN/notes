#REDIS源码1-(基于5.0版本)启动

## 启动源码分析

redis和大多数的c程序一样，有一个main函数作为启动方法，这个方法在server.c中。

下面仅选取核心的一部分代码，无关的都删除了。

```c
int main(int argc, char **argv) {
    struct timeval tv;
    int j;

	  // 这个方法是为了改进程名称
	  // 原理是通过修改调用c传入的启动参数argv[0]，就可以修改进程名称
    // 但是因为这是一块连续的内存，如果修改了argv[0]，新名称的在内存中的大小几乎不可能和原来一致，所以基本的做法就是把这些参数拷贝一份出来，然后再修改argv[0]，因为后面内存的数据已经被拷贝了一份，所以这样修改即便污染了后面的数据也没有关系
    spt_init(argc, argv);

	  // 设置语言
    setlocale(LC_COLLATE,"");
  	// 设置时区
    tzset();
	  // 内存溢出处理
    zmalloc_set_oom_handler(redisOutOfMemoryHandler);
	  // 获取时间
    gettimeofday(&tv,NULL);

	  // 这个就是获取一个随机的hash种子，然后后面的hash函数的种子就用这里随机生成的作为种子
    char hashseed[16];
    getRandomHexChars(hashseed,sizeof(hashseed));
    dictSetHashFunctionSeed((uint8_t*)hashseed);
	  // 检查是否是sentinel，暂时先不关注是否是sentinel
    // 不过判断是不是sentinel的方法就是看启动命令是不是有--sentinel或者redis-sentinel
    server.sentinel_mode = checkForSentinelMode(argc,argv);
	  // 初始化server的配置
	  // redis各种初始化的值都在这个方法里面设置
    initServerConfig();
	  // module是给别人做插件用的，忽略
    moduleInitModulesSystem();

    /* Store the executable path and arguments in a safe place in order
     * to be able to restart the server later. */
    server.executable = getAbsolutePath(argv[0]);
    server.exec_argv = zmalloc(sizeof(char*)*(argc+1));
    server.exec_argv[argc] = NULL;
    for (j = 0; j < argc; j++) server.exec_argv[j] = zstrdup(argv[j]);

    // sentinel模式，不管
    if (server.sentinel_mode) {
        initSentinelConfig();
        initSentinel();
    }

    /* Check if we need to start in redis-check-rdb/aof mode. We just execute
     * the program main. However the program is part of the Redis executable
     * so that we can easily execute an RDB check on loading errors. */
  	// 参数校验
    if (strstr(argv[0],"redis-check-rdb") != NULL)
        redis_check_rdb_main(argc,argv,NULL);
    else if (strstr(argv[0],"redis-check-aof") != NULL)
        redis_check_aof_main(argc,argv);

		// 这里省略一大段，主要是读取配置

    serverLog(LL_WARNING, "oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo");
    serverLog(LL_WARNING,
        "Redis version=%s, bits=%d, commit=%s, modified=%d, pid=%d, just started",
            REDIS_VERSION,
            (sizeof(long) == 8) ? 64 : 32,
            redisGitSHA1(),
            strtol(redisGitDirty(),NULL,10) > 0,
            (int)getpid());

	  // 监督模式？不知道干嘛的
    server.supervised = redisIsSupervised(server.supervised_mode);
    int background = server.daemonize && !server.supervised;
    if (background) daemonize();

  	// 初始化redis server
	  // 最核心的相关逻辑
    initServer();
    if (background || server.pidfile) createPidFile();
	  // 这个就是前面的进程名称
    redisSetProcTitle(argv[0]);
	  // 输出一些日志
    redisAsciiArt();
	  // 不管，没用
    checkTcpBacklogSettings();

  	// 非sentinel模式
    if (!server.sentinel_mode) {
        /* Things not needed when running in Sentinel mode. */
        serverLog(LL_WARNING,"Server initialized");
    #ifdef __linux__
        linuxMemoryWarnings();
    #endif
        moduleLoadFromQueue();
	      // 初始化bio线程
        InitServerLast();
      	// aof和rdb模式读取数据
        loadDataFromDisk();
        if (server.cluster_enabled) {
            if (verifyClusterConfigWithData() == C_ERR) {
                serverLog(LL_WARNING,
                    "You can't have keys in a DB different than DB 0 when in "
                    "Cluster mode. Exiting.");
                exit(1);
            }
        }
        if (server.ipfd_count > 0)
            serverLog(LL_NOTICE,"Ready to accept connections");
        if (server.sofd > 0)
            serverLog(LL_NOTICE,"The server is now ready to accept connections at %s", server.unixsocket);
    } else {
	      // sentinel模式，暂时不管
        InitServerLast();
        sentinelIsRunning();
    }

    /* Warning the user about suspicious maxmemory setting. */
    if (server.maxmemory > 0 && server.maxmemory < 1024*1024) {
        serverLog(LL_WARNING,"WARNING: You specified a maxmemory value that is less than 1MB (current value is %llu bytes). Are you sure this is what you really want?", server.maxmemory);
    }

    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeSetAfterSleepProc(server.el,afterSleep);
  	// main循环，处理逻辑
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
    return 0;
}
```

Redis的整体启动流程非常清楚，其中几个比较重要的点就是initServer处的处理，以及最后aeMain的EventLoop迭代。

后面会先从Redis的基本数据结构开始分析，然后再重新回到Redis本身的架构上。