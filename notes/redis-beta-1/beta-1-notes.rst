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

此图参考 UML 代码： redis-main.puml 

