###############################################################################
Redis Beta 1 源码阅读笔记
###############################################################################

.. contents::

*******************************************************************************
第 1 部分  源码阅读环境 
*******************************************************************************

我的源码阅读环境为 ： WSL2 + CLion on Windows 10

Redis 在 beta 1 版本时使用的单进程单线程的事件驱动技术 (event based)， 又称为 I/O \
多路复用技术， 复用的是同一个线程。 多路复用是指使用一个线程来检查多个文件描述符 \
（Socket） 的就绪状态。

可编译代码已包含在此仓库中。

*******************************************************************************
第 2 部分  开始阅读源码
*******************************************************************************

redis 项目是一个纯 C 项目， 我们从 main 函数开始看起。

.. _main-func:
.. main-func

2.1 main 函数
===============================================================================

main 函数的代码如下：

.. code-block:: C 

    // redis.c
    int main(int argc, char **argv) {
        initServerConfig();
        initServer();
        if (argc == 2) {
            ResetServerSaveParams();
            loadServerConfig(argv[1]);
            redisLog(REDIS_NOTICE,"Configuration loaded");
        } else if (argc > 2) {
            fprintf(stderr,"Usage: ./redis-server [/path/to/redis.conf]\n");
            exit(1);
        }
        redisLog(REDIS_NOTICE,"Server started");
        if (loadDb("dump.rdb") == REDIS_OK)
            redisLog(REDIS_NOTICE,"DB loaded from disk");
        if (aeCreateFileEvent(server.el, server.fd, AE_READABLE,
            acceptHandler, NULL, NULL) == AE_ERR) oom("creating file event");
        redisLog(REDIS_NOTICE,"The server is now ready to accept connections");
        aeMain(server.el);
        aeDeleteEventLoop(server.el);
        return 0;
    }

main 函数的流程图可以参考下图： 

.. image:: https://planttext.com/api/plantuml/img/VP7DJWCn38JlVWeVjrUEkq9KTE5K94IV8EnEYaL-LecxWhSdIQbGLOb397io_ZHnjbbDqfDtH2hgmDv88A8c4_KIH0z8Az8k1Yl7WUbARRrOxamwJdpFTmyRrWy4xhwHDyJSlo7ZrtmmArvDCZuFzSP5Cr-ngvWmIzx7qi1bS1TYezWbIL3RBFWIhGN2JEM8BOd-nbgQYXxVEP-c2JdVPBguUNpaQiNCDaNFHVqSBipsAkmIZE9P79vM16LhIZdV46Fq_qJg3LxANi_L20Szq_OnBaDTTbo8jcMmVCGF
    :align: center
    :alt: main-flow
    :name: main-flow
    :target: none

此图参考 UML 代码： redis-main.puml_

.. _redis-main.puml: uml/redis-main.puml

在继续分析之前， 需要先看一下 ``server`` 这个全局变量。 

.. code-block:: C 

    static struct redisServer server;

也就是说 server 就是 redisServer_ 结构体

.. _redisServer: beta-1-structures.rst#redisServer-struct

分析 redisServer_ 结构体发现其内部含有 4 个结构体， 分别是 dict_， list_， \
aeEventLoop_ 和 saveparam_。

.. _dict: beta-1-structures.rst#dict-struct
.. _list: beta-1-structures.rst#list-struct
.. _aeEventLoop: beta-1-structures.rst#aeEventLoop-struct
.. _saveparam: beta-1-structures.rst#saveparam-struct

顾名思义， dict_ 就是字典 (哈希表)， list_ 是 (双向) 链表， aeEventLoop_ 是事件循\
环， saveparam_ 是保存参数， 其内容是变更次数及做变更时的时间戳。

.. _initServerConfig-func:
.. initServerConfig-func

2.2 initServerConfig 函数
===============================================================================

在上一节中了解了 redisServer_ 相关的内容， 现在正式进入 main_ 函数内的第一个函数: \
``initServerConfig`` 函数。 

.. _main: #main-func

.. code-block:: c 

    static void initServerConfig() {
        server.dbnum = REDIS_DEFAULT_DBNUM;
        server.port = REDIS_SERVERPORT;
        server.verbosity = REDIS_DEBUG;
        server.maxidletime = REDIS_MAXIDLETIME;
        server.saveparams = NULL;
        server.logfile = NULL; /* NULL = log on standard output */
        ResetServerSaveParams();

        appendServerSaveParams(60*60,1);  /* save after 1 hour and 1 change */
        appendServerSaveParams(300,100);  /* save after 5 minutes and 100 changes */
        appendServerSaveParams(60,10000); /* save after 1 minute and 10000 changes */
    }

首先对 server 全局变量进行设置。 然后执行 ``ResetServerSaveParams`` 函数和 \
``appendServerSaveParams`` 函数。 

ResetServerSaveParams_ 清空了 server 全局变量中的 ``saveparams`` 字段和 \
``saveparamslen`` 字段； appendServerSaveParams_ 则为 redis 持久化功能做铺垫， \
后续的 serverCron_ 函数将会使用 appendServerSaveParams_ 函数所做的设置。

.. _ResetServerSaveParams: beta-1-functions.rst#ResetServerSaveParams-func
.. _appendServerSaveParams: beta-1-functions.rst#appendServerSaveParams-func
.. _serverCron: beta-1-functions.rst#serverCron-func

总而言之就是对 redis server 进行设置， 为后续运行做出铺垫作用。 但并不牵扯到运行服务\
器。

.. _initServer-func:
.. initServer-func

2.3 initServer 函数
===============================================================================

.. code-block:: c 

    static void initServer() {
        int j;

        signal(SIGHUP, SIG_IGN);
        signal(SIGPIPE, SIG_IGN);

        server.clients = listCreate();
        server.objfreelist = listCreate();
        createSharedObjects();
        server.el = aeCreateEventLoop();
        server.dict = malloc(sizeof(dict*)*server.dbnum);
        if (!server.dict || !server.clients || !server.el || !server.objfreelist)
            oom("server initialization"); /* Fatal OOM */
        server.fd = anetTcpServer(server.neterr, server.port, NULL);
        if (server.fd == -1) {
            redisLog(REDIS_WARNING, "Opening TCP port: %s", server.neterr);
            exit(1);
        }
        for (j = 0; j < server.dbnum; j++) {
            server.dict[j] = dictCreate(&sdsDictType,NULL);
            if (!server.dict[j])
                oom("server initialization"); /* Fatal OOM */
        }
        server.cronloops = 0;
        server.bgsaveinprogress = 0;
        server.lastsave = time(NULL);
        server.dirty = 0;
        aeCreateTimeEvent(server.el, 1000, serverCron, NULL, NULL);
    }

signal 信号函数， 第一个参数表示需要处理的信号值 （SIGHUP）， 第二个参数为处理函数或\
者是一个标识， 这里 SIG_IGN 表示忽略 SIGHUP 那个注册的信号。

SIGHUP 和控制台操作有关， 当控制台被关闭时系统会向拥有控制台 sessionID 的所有进程发\
送 HUP 信号， 默认 HUP 信号的 action 是 exit， 如果远程登陆启动某个服务进程并在程序\
运行时关闭连接的话会导致服务进程退出， 所以一般服务进程都会用 nohup 工具启动或写成一\
个 daemon。

TCP 是全双工的信道， 可以看作两条单工信道， TCP 连接两端的两个端点各负责一条。 当对\
端调用 close 时， 虽然本意是关闭整个两条信道， 但本端只是收到 FIN 包。 按照 TCP 协\
议的语义， 表示对端只是关闭了其所负责的那一条单工信道， 仍然可以继续接收数据。 也就是\
说， 因为 TCP 协议的限制， 一个端点无法获知对端的 socket 是调用了 close 还是 \
shutdown。

对一个已经收到 FIN 包的 socket 调用 read 方法， 如果接收缓冲已空， 则返回 0， 这就\
是常说的表示连接关闭。 但第一次对其调用 write 方法时， 如果发送缓冲没问题， 会返回正\
确写入(发送)。 但发送的报文会导致对端发送 RST 报文， 因为对端的 socket 已经调用了 \
close， 完全关闭， 既不发送， 也不接收数据。 所以， 第二次调用 write 方法(假设在收\
到 RST 之后)， 会生成 SIGPIPE 信号， 导致进程退出。 

为了避免进程退出， 可以捕获 SIGPIPE 信号， 或者忽略它， 给它设置 SIG_IGN 信号处理函\
数: ``signal(SIGPIPE, SIG_IGN);`` 这样第二次调用 write 方法时， 会返回 -1， 同时 \
errno 置为 SIGPIPE。 程序便能知道对端已经关闭。

然后将 server 的 ``clients`` 字段和 ``objfreelist`` 字段通过 listCreate_ 函数初始\
为空的双端链表。

.. _listCreate: beta-1-functions.rst#listCreate-func

然后使用 createSharedObjects_ 函数创建共享对象

.. _createSharedObjects: beta-1-functions.rst#createSharedObjects-func

实际上就创建了一下字符串相关的共享对象。

然后将 ``server.el`` 置为 aeCreateEventLoop_， aeCreateEventLoop_ 函数用于创建事\
件循环。 

.. _aeCreateEventLoop: beta-1-functions.rst#aeCreateEventLoop-func

``server.dict`` 被设置为 ``dbnum * sizeof(dict*)``。 

注意 if 语句， 当 ``server.dict``、 ``server.clients``、 ``server.el`` 和 \
``server.objfreelist`` 其中任意一个为空时， 都会执行 oom_ 函数， 用于打印内存不足\
错误和中止程序运行。 它们是取非之后 ``!`` 又进行或运算 ``||`` 的。 

.. _oom: beta-1-functions.rst#oom-func

``server.fd`` 被用来存放可以正常接收数据的套接字文件描述符， 也就是说如果正常的话， \
TCP server 可以正常使用了。 正常情况下的 fd 为非负整数。 当 fd 为 -1 时， 执行 \
redisLog_ 函数并退出程序。

.. _redisLog: beta-1-functions.rst#redisLog-func

之后循环迭代创建 dict 哈希表， dbnum 为多少就创建多少个 dict。 使用 dictCreate_ 函\
数创建， 创建类型是 sdsDictType_， 私有数据为空 NULL。 创建完成后需要判断创建结果是\
否正常， 不正常的话 oom_ 函数进行报错。

.. _dictCreate: beta-1-functions.rst#dictCreate-func
.. _sdsDictType: beta-1-others.rst#sdsDictType-var

使用 dictCreate_ 函数创建的哈希表都是被初始化的， 内部均没有其他数据， 为 NULL 或 0。

然后 ``cronloops``， ``bgsaveinprogress``， ``dirty`` 三个 server 字段被设置为 0， \
lastsave 字段被设置为当前的时间戳， 因为 ``time(NULL)`` 计算的就是从 1970 年 1 月 \
1 日 00:00:00 到现在为止经过了多少秒。 

最后使用 aeCreateTimeEvent_ 函数创建定时器， 事件循环是当前的 server.el， 时间间隔\
是 1000 毫秒， 定时处理函数是 serverCron_ 函数， 另外两个参数均为 NULL， 不必在意。 \
也就是说 serverCron_ 函数每隔 1000 毫秒执行一次。 

.. _aeCreateTimeEvent: beta-1-functions.rst#aeCreateTimeEvent-func

如此， initServer 执行完毕， 创建了定时器， 每秒钟执行一次 serverCron_ 函数。 

.. _`loadServerConfig-func`:
.. `loadServerConfig-func`

2.4 loadServerConfig 函数
===============================================================================

loadServerConfig 函数是正常情况下必须执行的， 也就是从 conf 文件中加载 redis 的配置， \
非正常情况就是 else 语句中的 redis 执行参数 argc 大于 2， 它会打印正确的用法并退出执\
行。 

还是看正常情况， 也就是 argc 等于 2 的情况， 执行 ResetServerSaveParams_ 函数， 将 \
server 中的 saveparams 字段置为 NULL， saveparamslen 字段被置为 0。 然后执行 \
loadServerConfig 函数， 将 main 函数的第二个参数 ``argv[1]`` 作为 redis 配置文件作\
为参数， 解析其内容。

.. code-block:: c

    #define REDIS_CONFIGLINE_MAX    1024

    static void loadServerConfig(char *filename) {
        // 1
        FILE *fp = fopen(filename,"r");
        char buf[REDIS_CONFIGLINE_MAX+1], *err = NULL;
        int linenum = 0;
        sds line = NULL;
        
        // 2
        if (!fp) {
            redisLog(REDIS_WARNING,"Fatal error, can't open config file");
            exit(1);
        }

        // 3
        while(fgets(buf,REDIS_CONFIGLINE_MAX+1,fp) != NULL) {

            // 1
            sds *argv;
            int argc;

            linenum++;
            line = sdsnew(buf);
            line = sdstrim(line," \t\r\n");

            // 2
            /* Skip comments and blank lines*/
            if (line[0] == '#' || line[0] == '\0') {
                sdsfree(line);
                continue;
            }

            // 3
            /* Split into arguments */
            argv = sdssplitlen(line,sdslen(line)," ",1,&argc);

            // 4
            /* Execute config directives */
            if (!strcmp(argv[0],"timeout") && argc == 2) {
                server.maxidletime = atoi(argv[1]);
                if (server.maxidletime < 1) {
                    err = "Invalid timeout value"; goto loaderr;
                }
            } else if (!strcmp(argv[0],"save") && argc == 3) {
            // 5
                int seconds = atoi(argv[1]);
                int changes = atoi(argv[2]);
                if (seconds < 1 || changes < 0) {
                    err = "Invalid save parameters"; goto loaderr;
                }
                appendServerSaveParams(seconds,changes);
            } else if (!strcmp(argv[0],"dir") && argc == 2) {
            // 6
                if (chdir(argv[1]) == -1) {
                    redisLog(REDIS_WARNING,"Can't chdir to '%s': %s",
                        argv[1], strerror(errno));
                    exit(1);
                }
            } else if (!strcmp(argv[0],"loglevel") && argc == 2) {
            // 7    
                if (!strcmp(argv[1],"debug")) server.verbosity = REDIS_DEBUG;
                else if (!strcmp(argv[1],"notice")) server.verbosity = REDIS_NOTICE;
                else if (!strcmp(argv[1],"warning")) server.verbosity = REDIS_WARNING;
                else {
                    err = "Invalid log level. Must be one of debug, notice, warning";
                    goto loaderr;
                }
            } else if (!strcmp(argv[0],"logfile") && argc == 2) {
            // 8    
                FILE *fp;

                server.logfile = strdup(argv[1]);
                if (!strcmp(server.logfile,"stdout")) server.logfile = NULL;
                if (server.logfile) {
                    /* Test if we are able to open the file. The server will not
                    * be able to abort just for this problem later... */
                    fp = fopen(server.logfile,"a");
                    if (fp == NULL) {
                        err = sdscatprintf(sdsempty(),
                            "Can't open the log file: %s", strerror(errno));
                        goto loaderr;
                    }
                    fclose(fp);
                }
            } else if (!strcmp(argv[0],"databases") && argc == 2) {
            // 9    
                server.dbnum = atoi(argv[1]);
                if (server.dbnum < 1) {
                    err = "Invalid number of databases"; goto loaderr;
                }
            } else {
            // 10    
                err = "Bad directive or wrong number of arguments"; goto loaderr;
            }
            // 11
            sdsfree(line);
        }
        // 4
        fclose(fp);
        return;

        // 5
    loaderr:
        fprintf(stderr, "\n*** FATAL CONFIG FILE ERROR ***\n");
        fprintf(stderr, "Reading the configuration file, at line %d\n", linenum);
        fprintf(stderr, ">>> '%s'\n", line);
        fprintf(stderr, "%s\n", err);
        exit(1);
    }

这个函数很长， 我将它按照结构大致分成了几部分， 后面会按照这个结构进行解析。

- STEP-1: 将加载的文件以文件流 fp 的方式打开， 并初始化 4 个局部变量。
- STEP-2: 当 fp 为空时， 说明加载文件失败， 记录日志并退出程序
- STEP-3: 然后从 fp 逐行读取配置， fgets 函数的意思是从 fp 一次最多读取 \
  REDIS_CONFIGLINE_MAX+1 的内容， 并存储到 buf 中， 读取到 EOF 或换行符时停止。 执\
  行成功返回 buf， 失败返回 NULL。
    - STEP-1: 开始逐行读取后， 现将 linenum 自增加一， 然后对读取的内容使用 sdsnew_ \
      函数新建一个动态字符串 line， 并使用 sdstrim_ 函数去除 line 首尾的 \
      ``" \t\r\n"`` 字符。 
    - STEP-2: 如果首字符是 # 或 ``\0``， 说明是注释掉的行或空行， 直接使用 sdsfree_ \
      函数释放掉这一行， 并执行下一轮循环。
    - STEP-3: 正常情况下， 开始将你 line 拆分成参数形式。 使用 sdssplitlen_ 函数进\
      行拆分。 分割符是空格， 长度是 1， 分割后的数量存入 argc 中。 
    - STEP-4: 当分割后的字符串数组第一个字符串等于 timeout 且 argc 等于 2， 将 \
      server 的 maxidletime 字段置为第二个字符串； 即 redis 的配置文件是 "配置 值" \
      格式， 配置名称与值之间使用空格进行分割。
    - STEP-5: 当配置名称是 save 且 argc 是 3 时， 执行 appendServerSaveParams_ 函\
      数进行定时器注册。 在 seconds 时间内进行 changes 次修改后将执行数据备份操作。
    - STEP-6: 当配置名称是 dir 且 argc 为 2 时， 进行切换工作目录操作， 如果切换失败\
      记录日志并中止程序执行。
    - STEP-7: 当配置名称是 loglevel 且 argc 为 2 时， 设置 server 的 verbosity 字\
      段， 也就是 redis 日志的级别即信息复杂度。 值分别是 debug， notice 和 warning
    - STEP-8: 当配置名称是 logfile 且 argc 为 2 时， 将 server 的 logfile 字段置为\
      logfile 配置的值， 如果 logfile 字段是 stdout， 就清空 logfile； 否则打开这个\
      文件， 如果打开失败， 




.. _sdsnew: beta-1-functions.rst#sdsnew-func
.. _sdstrim: beta-1-functions.rst#sdstrim-func
