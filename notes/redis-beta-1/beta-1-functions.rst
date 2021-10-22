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

.. _sharedObjectsStruct: beta-1-structures.rst#sharedObjectsStruct-struct

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

listLength_ 宏定义的作用是返回 list_ 的 len 的值， 即链表的长度。

.. _listLength: beta-1-macros.rst#listLength-macro
.. _list: beta-1-structures.rst#list-struct

listFirst_ 宏定义的作用是返回 list_ 的 head 的值， 即链表的头节点的指针。

.. _listFirst: beta-1-macros.rst#listFirst-macro

listNodeValue_ 宏定义的作用是返回 listNode_ 的 value 的值， 即链表节点的值指针。

.. _listNodeValue: beta-1-macros.rst#listNodeValue-macro

listDelNode_ 函数用于删除链表中指定的节点。 在此处就是删除链表的头节点， 因为释放的\
是头节点。

.. _listDelNode: #listDelNode-func

当 ``server`` 的 ``objfreelist`` 字段不为 0 时， 说明当前的 server 中有可以释放的 \
redis 对象， 那么直接从 ``objfreelist`` 链表中拿第一个对象作为新建的 redis 对象， \
否则就需要重新分配内存来新建 redis 对象。 此举是为了节省内存。 这就是第一个 if 语句的\
作用。 

最终将创建的 redis 对象地址返回。 

.. _listDelNode-func:
.. listDelNode-func

06 listDelNode 函数
==============================================================================

.. code-block:: c

    void listDelNode(list *list, listNode *node)
    {
        if (node->prev)
            node->prev->next = node->next;
        else
            list->head = node->next;
        if (node->next)
            node->next->prev = node->prev;
        else
            list->tail = node->prev;
        if (list->free) list->free(node->value);
        free(node);
        list->len--;
    }

删除节点函数有两个参数： ``list`` 是需要删除节点的链表； ``node`` 是被删的节点。

当当前节点 node 有前节点时， 说明不是链表的头节点， 删除节点时需要将前节点的 next 节\
点指向 node 的 next 节点， 略过自己； 否则的话说明 node 是头节点， 只需将头节点指向 \
node 的 next 节点。

当当前节点 node 有 next 节点时， 说明不是链表的尾节点， 删除节点时需要将 next 节点的 \
prev 节点指向当前节点 node 的 prev 节点， 也是要略过自己， 毕竟当前节点 node 是要删\
除的； 否则的话说明 node 是尾节点， 只需要将尾节点指向当前节点的 prev 节点。

如果 list 的 free 设置了某个函数， 将会对这个 node 执行该函数。

然后释放 node 的内存， 同时将 list 的 len 长度进行减 1。

.. _sdsnew-func:
.. sdsnew-func

07 sdsnew 函数
==============================================================================

.. code-block:: C 

    sds sdsnew(const char *init) {
        size_t initlen = (init == NULL) ? 0 : strlen(init);
        return sdsnewlen(init, initlen);
    }

sds_ 类型实际上是字符指针类型， redis 中实现了 sds_， 实际上可以看做 simple \
dynamic strings 简单动态字符串的缩写

.. _sds: beta-1-typedefs.rst#sds-typedef

当字符指针 (也可以看做是字符串) ``init`` 为 NULL 时， initlen 取 0， 否则取字符串 \
``init`` 的长度； 然后执行 sdsnewlen_ 函数创建一个给定长度的字符串。

.. _sdsnewlen: #sdsnewlen-func

.. _sdsnewlen-func:
.. sdsnewlen-func

08 sdsnewlen 函数
==============================================================================

.. code-block:: C 

    sds sdsnewlen(const void *init, size_t initlen) {
        struct sdshdr *sh;

        sh = malloc(sizeof(struct sdshdr)+initlen+1);
    #ifdef SDS_ABORT_ON_OOM
        if (sh == NULL) sdsOomAbort();
    #else
        if (sh == NULL) return NULL;
    #endif
        sh->len = initlen;
        sh->free = 0;
        if (initlen) {
            if (init) memcpy(sh->buf, init, initlen);
            else memset(sh->buf,0,initlen);
        }
        sh->buf[initlen] = '\0';
        return (char*)sh->buf;
    }

在这个函数中首先遇到了 sdshdr_ 结构体， 它的全称是 Simple Dynamic Strings Header。 \
这个结构体包含了字符串的长度、 剩余空间和字符串本身。

.. _sdshdr: beta-1-structures.rst#sdshar-struct

然后根据指定的字符串长度 ``initlen`` 分配内存大小， 首先是字符串头部大小 sdshdr 大\
小加上指定的长度 ``initlen``， 用于存放字符串， 而最后的 1 则表示字符串结束符 ``\0`` \
。 

如果定义了 ``SDS_ABORT_ON_OOM``， 当 ``sh`` 为 NULL 时， 执行 sdsOomAbort_ 函数， \
打印内存不足信息并中止程序执行， 直接从调用的地方跳出。 如果没有定义， 则直接返回 \
NULL。 

.. _sdsOomAbort: #sdsOomAbort-func

然后将字符串头部的 len 置为要创建的字符串的长度 initlen， 将 free 置为 0； 当 \
initlen 不为 0 时， 且字符串 init 不为空时， 将字符串 init 复制到 sh->buf 指向的地\
址中， 长度为 initlen， 如果字符串 init 为空， 则将字符 0 复制到 sh->buf 指向的地址\
中， 长度也是 initlen。 最后在向字符串结尾添加结束符 ``\0``。 

最终返回创建的字符串的地址。


