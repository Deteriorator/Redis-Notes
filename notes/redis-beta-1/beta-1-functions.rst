##############################################################################
Redis Beta 1 源码阅读笔记 - Functions
##############################################################################

.. contents::

******************************************************************************
Functions
******************************************************************************

.. _ResetServerSaveParams-func:
.. ResetServerSaveParams-func

01 ResetServerSaveParams 函数
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

02 appendServerSaveParams 函数
==============================================================================

.. code-block:: c

    static void appendServerSaveParams(time_t seconds, int changes) {
        server.saveparams = realloc(server.saveparams,sizeof(struct saveparam)*(server.saveparamslen+1));
        if (server.saveparams == NULL) oom("appendServerSaveParams");
        server.saveparams[server.saveparamslen].seconds = seconds;
        server.saveparams[server.saveparamslen].changes = changes;
        server.saveparamslen++;
    }

该函数用于 redis 的持久化功能。 ``server.saveparamslen`` 初始为 0， \
initServerConfig_ 函数中连续执行了 3 次 ``appendServerSaveParams`` 函数， 注册了 \
3 次 redis 持久化检查任务， 分别是一小时内有 1 次改变、 5 分钟内有 100 次改变和 1 \
分钟内 10000 次改变。 

.. _initServerConfig: beta-1-main-flow.rst#initServerConfig-func

``appendServerSaveParams`` 函数每次执行， 都会先分配内存， 然后将 saveparams 字段\
填上， 例如 ``appendServerSaveParams(60*60,1);`` 步骤会将 3600 添加到 \
server.saveparams[0].seconds， 将 1 填到 server.saveparams[0].changes， 同时将 \
``server.saveparamslen`` 字段进行自增。

这个函数会为后来的数据文件保存做铺垫。

.. _listCreate-func:
.. listCreate-func

02 listCreate 函数
==============================================================================

.. code-block:: c

    list *listCreate(void)
    {
        struct list *list;

        if ((list = malloc(sizeof(*list))) == NULL)
            return NULL;
        list->head = list->tail = NULL;
        list->len = 0;
        list->dup = NULL;
        list->free = NULL;
        list->match = NULL;
        return list;
    }

该函数用于新建一个空的双端链表， 分配好内存后， 将值置为 NULL， 长度置为 0， 最终返\
回这个新建的链表。

