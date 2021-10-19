##############################################################################
Redis Beta 1 源码阅读笔记
##############################################################################

.. contents::

******************************************************************************
第 1 部分  源码阅读环境 
******************************************************************************

我的源码阅读环境为 ： WSL2 + CLion on Windows 10

Redis 在 beta 1 版本时使用的单进程单线程的事件驱动技术 (event based)， 又称为 I/O \
多路复用技术， 复用的是同一个线程。 多路复用是指使用一个线程来检查多个文件描述符 \
（Socket） 的就绪状态。

可编译代码已包含在此仓库中。

******************************************************************************
第 2 部分  开始阅读源码
******************************************************************************

redis 项目是一个纯 C 项目， 我们从 main 函数开始看起。

.. _main-func:
.. main-func

2.1 main 函数
==============================================================================

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
    :alt: main
    :name: main
    :target: none

此图参考 UML 代码： redis-main.puml_

.. _redis-main.puml: uml/redis-main.puml

在继续分析之前， 需要先看一下 ``server`` 这个全局变量。 

.. code-block:: C 

    static struct redisServer server;

也就是说 server 就是 redisServer_ 结构体

.. _redisServer: beta-1-structures.rst#redisServer-structure

分析 redisServer_ 结构体发现其内部含有 4 个结构体， 分别是 dict_， list_， \
aeEventLoop_ 和 saveparam_。

.. _dict: beta-1-structures.rst#dict-structure
.. _list: beta-1-structures.rst#list-structure
.. _aeEventLoop: beta-1-structures.rst#aeEventLoop-structure
.. _saveparam: beta-1-structures.rst#saveparam-structure

顾名思义， dict_ 就是字典 (哈希表)， list_ 是 (双向) 链表， aeEventLoop_ 是事件循\
环， saveparam_ 是保存参数， 其内容是变更次数及做变更时的时间戳。

.. _initServerConfig-func:
.. initServerConfig-func

2.2 initServerConfig 函数
==============================================================================

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
``appendServerSaveParams`` 函数。 总而言之就是对 redis server 进行设置， 为后续运\
行做出铺垫作用。 

.. _ResetServerSaveParams-func:
.. ResetServerSaveParams-func

2.3 ResetServerSaveParams 函数
==============================================================================

.. code-block:: c

    static void ResetServerSaveParams() {
        free(server.saveparams);
        server.saveparams = NULL;
        server.saveparamslen = 0;
    }

static 关键字表示该函数只能在本文件中使用。 ``ResetServerSaveParams`` 函数的功能是\
清空 server 全局变量中的 ``saveparams`` 字段和 ``saveparamslen`` 字段。 

首先释放掉 ``server.saveparams`` 字段的内存， 然后将该字段置为 NULL， 同时将 \
``saveparamslen`` 置为 0， ``saveparamslen`` 顾名思义就是 ``server.saveparams`` \
的长度。

.. _appendServerSaveParams-func:
.. appendServerSaveParams-func

2.4 appendServerSaveParams 函数
==============================================================================

.. code-block:: c

    static void appendServerSaveParams(time_t seconds, int changes) {
        server.saveparams = realloc(server.saveparams,sizeof(struct saveparam)*(server.saveparamslen+1));
        if (server.saveparams == NULL) oom("appendServerSaveParams");
        server.saveparams[server.saveparamslen].seconds = seconds;
        server.saveparams[server.saveparamslen].changes = changes;
        server.saveparamslen++;
    }

该函数用于 redis 持久化功能。 ``server.saveparamslen`` 初始为 0， \
initServerConfig_ 函数中连续执行了 3 次 ``appendServerSaveParams`` 函数， 注册了 \
3 次 redis 持久化检查任务， 分别是一小时内有 1 次改变、 5 分钟内有 100 次改变和 1 \
分钟内 10000 次改变。 

.. _initServerConfig: #initServerConfig-func

``appendServerSaveParams`` 函数每次执行， 都会先分配内存， 然后将 saveparams 字段\
填上， 例如 ``appendServerSaveParams(60*60,1);`` 步骤会将 3600 添加到 \
server.saveparams[0].seconds， 将 1 填到 server.saveparams[0].changes， 同时将 \
``server.saveparamslen`` 字段进行自增。

这个函数会为后来的数据文件保存做铺垫。

回到 initServerConfig_ 函数中， 到此 initServerConfig_ 函数是完成了分析。


