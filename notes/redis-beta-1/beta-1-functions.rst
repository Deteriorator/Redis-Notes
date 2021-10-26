###############################################################################
Redis Beta 1 源码阅读笔记 - Functions
###############################################################################

.. contents::

*******************************************************************************
Functions
*******************************************************************************

.. _ResetServerSaveParams-func:
.. ResetServerSaveParams-func

01 ResetServerSaveParams 函数
===============================================================================

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
===============================================================================

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
===============================================================================

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
===============================================================================

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
===============================================================================

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

.. _listNode: beta-1-structures.rst#listNode-struct
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
===============================================================================

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
===============================================================================

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
===============================================================================

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

.. _sdsOomAbort-func:
.. sdsOomAbort-func

09 sdsOomAbort 函数
===============================================================================

.. code-block:: C 

    static void sdsOomAbort(void) {
        fprintf(stderr,"SDS: Out Of Memory (SDS_ABORT_ON_OOM defined)\n");
        abort();
    }

执行这个函数的原因是内存不足了， 将错误信息向标准错误 stderr 传输， 同时中止程序执行。 

.. _aeCreateEventLoop-func:
.. aeCreateEventLoop-func

10 aeCreateEventLoop 函数
===============================================================================

.. code-block:: C 

    aeEventLoop *aeCreateEventLoop(void) {
        aeEventLoop *eventLoop;

        eventLoop = malloc(sizeof(*eventLoop));
        if (!eventLoop) return NULL;
        eventLoop->fileEventHead = NULL;
        eventLoop->timeEventHead = NULL;
        eventLoop->timeEventNextId = 0;
        eventLoop->stop = 0;
        return eventLoop;
    }

aeEventLoop_ 类型之前已经解析过了。

.. _aeEventLoop: beta-1-structures.rst#aeEventLoop-struct

先分配内存， 当 eventLoop 不为 NULL 时， 初始化 eventLoop 各个字段的值， 最终返回 \
eventLoop。 

.. _oom-func:
.. oom-func

11 oom 函数
===============================================================================

.. code-block:: C 

    static void oom(const char *msg) {
        fprintf(stderr, "%s: Out of memory\n",msg);
        fflush(stderr);
        sleep(1);
        abort();
    }

与之前的 sdsOomAbort_ 函数类似， 将内存不足的信息传输到 stderr 打印之后， 清除 \
stderr 缓存， 休息 1 秒钟后中止程序执行

.. _sdsOomAbort: #sdsOomAbort-func

.. _anetTcpServer-func:
.. anetTcpServer-func

12 anetTcpServer 函数
===============================================================================

.. code-block:: C 

    int anetTcpServer(char *err, int port, char *bindaddr)
    {
        int s, on = 1;
        struct sockaddr_in sa;
        
        // 1
        if ((s = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
            anetSetError(err, "socket: %s\n", strerror(errno));
            return ANET_ERR;
        }

        // 2
        if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) == -1) {
            anetSetError(err, "setsockopt SO_REUSEADDR: %s\n", strerror(errno));
            close(s);
            return ANET_ERR;
        }
        sa.sin_family = AF_INET;
        sa.sin_port = htons(port);
        sa.sin_addr.s_addr = htonl(INADDR_ANY);
        
        // 3
        if (bindaddr) inet_aton(bindaddr, &sa.sin_addr);

        // 4
        if (bind(s, (struct sockaddr*)&sa, sizeof(sa)) == -1) {
            anetSetError(err, "bind: %s\n", strerror(errno));
            close(s);
            return ANET_ERR;
        }

        // 5
        if (listen(s, 5) == -1) {
            anetSetError(err, "listen: %s\n", strerror(errno));
            close(s);
            return ANET_ERR;
        }
        return s;
    }

此函数的核心代码就是调用系统 socket 库的 ``listen`` 函数建立起了一个 TCP Server。 

此函数可以拆分成 5 个主要步骤：

#. ``socket`` 函数用于创建一个新的通信端 (socket)， 如果创建成功将返回一个新的文件\
   描述符， 否则返回 -1， 同时将错误代码写入 errno。 如果等于 -1， 说明创建失败， 然\
   后执行 anetSetError_ 函数并返回错误信息

#. ``setsockopt`` 函数用于操作文件描述符引用的 socket， 如果操作成功返回 0， 否则返\
   回 -1， 同时设置相应的 errno； 然后执行 anetSetError_ 函数， 关闭 socket 并返回\
   错误信息； 然后设置 socket 的相关信息， ``htons`` 用于将无符号的 short 整型主机\
   字节序转换为网络字节序； ``htonl`` 则用于将无符号的整型主机字节序转换为网络字节序。

#. 当指定了地址 ``bindaddr``， ``inet_aton`` 函数则会将 ``bindaddr`` 从数字与点构\
   成的 IPv4 转换为网络字节序的二进制数据， 并存储到 ``&sa.sin_addr``， 如果地址是\
   有效的则返回非零， 否则返回 0

#. 使用 ``bind`` 函数将 IP 地址与 socket 进行绑定； ``socket`` 函数创建套接字的时\
   候， 这个套接字就存在地址簇中了， 但是没有 IP 地址分配给它， ``bind`` 函数将指定\
   的地址分配给套接字， 如果执行成功返回 0， 否则返回 -1 并设置相应的 errno。

#. 这一步是核心步骤， ``listen`` 函数将文件描述符代表的套接字标记为一个被动的套接字， \
   可以使用 ``accept`` 函数接收进入的网络请求； 而那个 5 表示的是队列的长度为 5。 \
   执行成功返回 0， 失败返回 -1 同时设置相应的 errno。

#. 如果以上步骤都没有问题， 将返回这个可以正常接收数据的套接字文件描述符。

.. _anetSetError: #anetSetError-func

.. _anetSetError-func:
.. anetSetError-func

13 anetSetError 函数
===============================================================================

.. code-block:: C 

    #define ANET_ERR_LEN 256

    static void anetSetError(char *err, const char *fmt, ...)
    {
        va_list ap;

        if (!err) return;
        va_start(ap, fmt);
        vsnprintf(err, ANET_ERR_LEN, fmt, ap);
        va_end(ap);
    }

该函数使用了可变参数， ``void va_start(va_list ap, last);`` 从该函数的的声明可以看\
出: 最后一个确定参数是 last， 可变参数是从 last 开始的， 一直到最后， 一旦 va_end \
函数执行， ap 将变成 undefined 状态；  

.. code-block:: C 

    int vsnprintf(char *str, size_t size, const char *format, va_list ap);

格式化字符串， 最多写入 size 字节 (包含字符串结束符 "\\0") 到 str 中。

此函数中的 size 被设定为 ``ANET_ERR_LEN`` 也就是 256。

.. _redisLog-func:
.. redisLog-func

14 redisLog 函数
===============================================================================

.. code-block:: C 

    void redisLog(int level, const char *fmt, ...)
    {
        va_list ap;
        FILE *fp;

        fp = (server.logfile == NULL) ? stdout : fopen(server.logfile,"a");
        if (!fp) return;

        va_start(ap, fmt);
        if (level >= server.verbosity) {
            char *c = ".-*";
            fprintf(fp,"%c ",c[level]);
            vfprintf(fp, fmt, ap);
            fprintf(fp,"\n");
            fflush(fp);
        }
        va_end(ap);

        if (server.logfile) fclose(fp);
    }

redis 日志记录函数， 参数是可变参数， 有两个固定参数： 

#. level： 表示的是日志等级
#. fmt： 日志格式
#. 其他： 为可变参数

可变参数是从 fmt 开始的， 之后都是可变参数。 

首先判断 server.logfile 是否为 NULL， 若是将 fp 置为 stdout， 否则以追加的形式打\
开文件流， 然后判断文件流是否正常， 不正常直接返回空

当 level 大于或等于 ``server.verbosity``， 即 server 的信息复杂度， 也就是日志级\
别了， 在 initServerConfig_ 函数中被定义为 ``REDIS_DEBUG``

.. code-block:: c

    ...
    server.verbosity = REDIS_DEBUG;
    ...

    /* Log levels */
    #define REDIS_DEBUG 0
    #define REDIS_NOTICE 1
    #define REDIS_WARNING 2

因此函数中的 ``c[level]`` 为 ``.``

然后将可变参数以 fmt 格式写入到 fp 中， 最后换行。 函数的结尾判断是否有日志文件， 如\
果有， 还需要关闭 fp 文件流。

.. _dictCreate-func:
.. dictCreate-func

15 dictCreate 函数
===============================================================================

.. code-block:: C 

    /* Create a new hash table */
    dict *dictCreate(dictType *type, void *privDataPtr)
    {
        dict *ht = _dictAlloc(sizeof(*ht));

        _dictInit(ht,type,privDataPtr);
        return ht;
    }

该函数用于创建一个新的 dict 哈希表， type 是类型指针， privDataPtr 是私有数据指针。

首先先分配内存空间， 即执行 `_dictAlloc`_ 函数， 大小就是 dict_ 结构体的大小， 然后对\
这个对象进行初始化， 执行 `_dictInit`_ 函数。 

.. _dict: beta-1-structures.rst#dict-struct
.. _`_dictAlloc`: #_dictAlloc-func
.. _`_dictInit`: #_dictInit-func

最后返回这个新建的哈希表。 函数中的 ht 就是 hash table 的首字母缩写。

.. _`_dictAlloc-func`:
.. `_dictAlloc-func`

16 _dictAlloc 函数
===============================================================================

.. code-block:: C 

    static void *_dictAlloc(int size)
    {
        void *p = malloc(size);
        if (p == NULL)
            _dictPanic("Out of memory");
        return p;
    }