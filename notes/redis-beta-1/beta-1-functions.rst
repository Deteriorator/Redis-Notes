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

03 listCreate 函数
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

.. _createSharedObjects-func:
.. createSharedObjects-func

04 createSharedObjects 函数
==============================================================================

.. code-block:: c

    #define REDIS_STRING 0

    static void createSharedObjects(void) {
        shared.crlf = createObject(REDIS_STRING,sdsnew("\r\n"));
        shared.ok = createObject(REDIS_STRING,sdsnew("+OK\r\n"));
        shared.err = createObject(REDIS_STRING,sdsnew("-ERR\r\n"));
        shared.zerobulk = createObject(REDIS_STRING,sdsnew("0\r\n\r\n"));
        shared.nil = createObject(REDIS_STRING,sdsnew("nil\r\n"));
        shared.zero = createObject(REDIS_STRING,sdsnew("0\r\n"));
        shared.one = createObject(REDIS_STRING,sdsnew("1\r\n"));
        shared.pong = createObject(REDIS_STRING,sdsnew("+PONG\r\n"));
    }

这个函数主要是创建一些共享的全局对象， 我们平时在跟 redis 服务交互的时候， 如果有遇到\
错误， 会收到一些固定的错误信息或者字符串比如： -ERR syntax error， -ERR no such \
key。 这些字符串对象都是在这个函数里面进行初始化的。 

shared 全局变量是一个 sharedObjectsStruct_ 结构体。 

.. _sharedObjectsStruct: beta-1-structures.rst#sharedObjectsStruct-structure

``REDIS_STRING`` 常量被设置为 0， sdsnew_ 函数是字符串对象创建函数， 最终会返回字\
符串的地址

.. _sdsnew: #sdsnew-func

.. _createObject-func:
.. createObject-func

05 createObject 函数
==============================================================================

.. code-block:: c

    static robj *createObject(int type, void *ptr) {
        robj *o;

        if (listLength(server.objfreelist)) {
            listNode *head = listFirst(server.objfreelist);
            o = listNodeValue(head);
            listDelNode(server.objfreelist,head);
        } else {
            o = malloc(sizeof(*o));
        }
        if (!o) oom("createObject");
        o->type = type;
        o->ptr = ptr;
        o->refcount = 1;
        return o;
    }

在 createSharedObjects_ 函数中有使用到 createObject_ 函数， createObject_ 函数用\
于创建 redis 对象， 其参数有两个： ``type`` 为 redis 对象的类型； ``ptr`` 为 redis \
对象的地址指针。

.. _createSharedObjects: #createSharedObjects-func
.. _createObject: #createObject-func

当 ``server`` 的 ``objfreelist`` 字段不为 0 时， 说明当前的 server 中有可以释放的 \
redis 对象， 那么直接从 ``objfreelist`` 链表中拿一个对象作为新建的 redis 对象， 否\
则就需要重新分配内存来新建 redis 对象。 此举是为了节省内存。

最终将创建的 redis 对象地址返回。 

