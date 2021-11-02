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

首先用 ``malloc`` 函数分配内存空间， 如果 p 为空， 则说明内存分配失败了， 因此会执行 \
`_dictPanic`_ 函数打印错误信息。 

.. _`_dictPanic`: #_dictPanic-func

如果内存分配成功， 直接返回分配的内存的地址。

.. _`_dictPanic-func`:
.. `_dictPanic-func`

17 _dictPanic 函数
===============================================================================

.. code-block:: C 

    static void _dictPanic(const char *fmt, ...)
    {
        va_list ap;

        va_start(ap, fmt);
        fprintf(stderr, "\nDICT LIBRARY PANIC: ");
        vfprintf(stderr, fmt, ap);
        fprintf(stderr, "\n\n");
        va_end(ap);
    }

该函数是一个可变参数函数， 有一个固定参数 fmt， 表示的是格式； 然后将 \
"\nDICT LIBRARY PANIC: " 字符串传输到标准错误输出 stderr， 然后对可变参数列表进行格\
式化输出， 最后换行。 总而言之就是用来打印 dict 模块错误信息的函数。

.. _`_dictInit-func`:
.. `_dictInit-func`

18 _dictInit 函数
===============================================================================

.. code-block:: C 

    #define DICT_OK 0

    /* Initialize the hash table */
    int _dictInit(dict *ht, dictType *type, void *privDataPtr)
    {
        _dictReset(ht);
        ht->type = type;
        ht->privdata = privDataPtr;
        return DICT_OK;
    }

初始化 dict 哈希表的函数拥有 3 个参数， 分别是需要初始化的哈希表 ht， 初始化的类型 \
type 以及私有数据 privDataPtr。 

首先会执行 `_dictReset`_ 函数将哈希表重置， 然后将重置后的哈希表 ht 的 type 字段设置\
为参数 type， privdata 字段设置为 privDataPtr 参数。 一切 OK 的话， 返回 DICT_OK， \
也就是 0。

.. _`_dictReset`: #_dictReset-func

.. _`_dictReset-func`:
.. `_dictReset-func`

19 _dictReset 函数
===============================================================================

.. code-block:: C 

    /* Reset an hashtable already initialized with ht_init().
    * NOTE: This function should only called by ht_destroy(). */
    static void _dictReset(dict *ht)
    {
        ht->table = NULL;
        ht->size = 0;
        ht->sizemask = 0;
        ht->used = 0;
    }

顾名思义， 重置哈希表， 但是根据代码注释， 这个方法只能被 ``ht_destroy`` 调用。

将 table 字段置为 NULL， 其他字段被置为 0。

.. _`aeCreateTimeEvent-func`:
.. `aeCreateTimeEvent-func`

20 aeCreateTimeEvent 函数
===============================================================================

.. code-block:: C 

    #define AE_ERR -1

    long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
            aeTimeProc *proc, void *clientData,
            aeEventFinalizerProc *finalizerProc)
    {
        long long id = eventLoop->timeEventNextId++;
        aeTimeEvent *te;

        te = malloc(sizeof(*te));
        if (te == NULL) return AE_ERR;
        te->id = id;
        aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
        te->timeProc = proc;
        te->finalizerProc = finalizerProc;
        te->clientData = clientData;
        te->next = eventLoop->timeEventHead;
        eventLoop->timeEventHead = te;
        return id;
    }

该函数用于创建定时器， 首先将当前事件循环的下一个定时器的 ID 自增加一存到 id 里面， \
te 是一个指向定时器 aeTimeEvent_ 的指针。

.. _aeTimeEvent: beta-1-structures.rst#aeTimeEvent-struct

然后对定时器分配内存， 并将内存地址赋值给 te， 如果 te 为 NULL， 说明内存分配失败了， \
直接返回 ``AE_ERR`` 即 -1。 

然后将 id 赋值个定时的 id 字段； 然后对当前定时器的时间进行操作， 实际上就是修改定时\
器的 when_sec 字段和 when_ms 字段， 这个过程执行的是 aeAddMillisecondsToNow_ 函数。 

.. _aeAddMillisecondsToNow: #aeAddMillisecondsToNow-func

然后设置定时器的处理函数， timeProc 字段被设置为参数 proc； finalizerProc 字段被设\
置为参数 finalizerProc； clientData 字段被设置为参数 clientData。

再然后这个新建的定时器的下一个定时器被设置为当前事件循环的定时器链表的头指针， 同时当\
前事件循环的定时器头指针被设置为这个新建的定时器。 实际上就是创建完就作为第一个监听的\
定时器。

最终将定时器的 id 返回。

.. _`aeAddMillisecondsToNow-func`:
.. `aeAddMillisecondsToNow-func`

21 aeAddMillisecondsToNow 函数
===============================================================================

.. code-block:: C 

    static void aeAddMillisecondsToNow(long long milliseconds, long *sec, long *ms) {
        long cur_sec, cur_ms, when_sec, when_ms;

        aeGetTime(&cur_sec, &cur_ms);
        when_sec = cur_sec + milliseconds/1000;
        when_ms = cur_ms + milliseconds%1000;
        if (when_ms >= 1000) {
            when_sec ++;
            when_ms -= 1000;
        }
        *sec = when_sec;
        *ms = when_ms;
    }

这个函数的功能很简单， 对时间进行换算， 当前的时间加上需要间隔的毫秒数， 最终返回超时\
时间， 也就是时间到了那个点， 就会执行一些操作。

aeGetTime_ 函数用于获取当前的秒和毫秒。

.. _aeGetTime: #aeGetTime-func

``milliseconds/1000`` 用于获取 milliseconds 包含有多少秒， 如果 milliseconds 大于\
或等于 1000， 则取整， 否则为 0。 然后用当前的毫秒加上上一步剩余的毫秒， 如果 when_ms \
大于等于 1000， 可以对秒进行加一， 同时将毫秒减去 1000， 最终将计算后的秒和毫秒赋值给\
参数 sec 和参数 ms。

.. _`aeGetTime-func`:
.. `aeGetTime-func`

22 aeGetTime 函数
===============================================================================

.. code-block:: C 

    static void aeGetTime(long *seconds, long *milliseconds)
    {
        struct timeval tv;

        gettimeofday(&tv, NULL);
        *seconds = tv.tv_sec;
        *milliseconds = tv.tv_usec/1000;
    }

该函数调用 gettimeofday 函数获取当前的时间， tv_sec 表示的是秒， tv_usec 表示的是微\
秒， 因此将其除以 1000 转换为毫秒。

.. _`serverCron-func`:
.. `serverCron-func`

23 serverCron 函数
===============================================================================

.. code-block:: C 

    #define REDIS_DEBUG 0
    #define REDIS_NOTICE 1
    #define REDIS_WARNING 2

    /* Hash table parameters */
    #define REDIS_HT_MINFILL        10      /* Minimal hash table fill 10% */
    #define REDIS_HT_MINSLOTS       16384   /* Never resize the HT under this */

    int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
        // 1
        int j, size, used, loops = server.cronloops++;
        REDIS_NOTUSED(eventLoop);
        REDIS_NOTUSED(id);
        REDIS_NOTUSED(clientData);

        // 2
        /* If the percentage of used slots in the HT reaches REDIS_HT_MINFILL
        * we resize the hash table to save memory */
        for (j = 0; j < server.dbnum; j++) {
            size = dictGetHashTableSize(server.dict[j]);
            used = dictGetHashTableUsed(server.dict[j]);
            if (!(loops % 5) && used > 0) {
                redisLog(REDIS_DEBUG,"DB %d: %d keys in %d slots HT.",j,used,size);
                // dictPrintStats(server.dict);
            }
            if (size && used && size > REDIS_HT_MINSLOTS &&
                (used*100/size < REDIS_HT_MINFILL)) {
                redisLog(REDIS_NOTICE,"The hash table %d is too sparse, resize it...",j);
                dictResize(server.dict[j]);
                redisLog(REDIS_NOTICE,"Hash table %d resized.",j);
            }
        }

        // 3
        /* Show information about connected clients */
        if (!(loops % 5)) redisLog(REDIS_DEBUG,"%d clients connected",listLength(server.clients));

        // 4
        /* Close connections of timedout clients */
        if (!(loops % 10))
            closeTimedoutClients();

        // 5
        /* Check if a background saving in progress terminated */
        if (server.bgsaveinprogress) {
            int statloc;
            if (wait4(-1,&statloc,WNOHANG,NULL)) {
                int exitcode = WEXITSTATUS(statloc);
                if (exitcode == 0) {
                    redisLog(REDIS_NOTICE,
                        "Background saving terminated with success");
                    server.dirty = 0;
                    server.lastsave = time(NULL);
                } else {
                    redisLog(REDIS_WARNING,
                        "Background saving error");
                }
                server.bgsaveinprogress = 0;
            }
        } else {
            /* If there is not a background saving in progress check if
            * we have to save now */
            time_t now = time(NULL);
            for (j = 0; j < server.saveparamslen; j++) {
                struct saveparam *sp = server.saveparams+j;

                if (server.dirty >= sp->changes &&
                    now-server.lastsave > sp->seconds) {
                    redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                        sp->changes, sp->seconds);
                    saveDbBackground("dump.rdb");
                    break;
                }
            }
        }
        return 1000;
    }

server 的 cronloops 字端根据我目前的理解应该是自动检测循环的次数， 初始的时候为 0。 \
将这个大函数根据注释分成 6 部分。

#. 新建局部变量 j， size， used 和 loops， 其中 loops 被初始化为 server.cronloops \
   + 1； 同时将三个参数 eventLoop， id 和 clientData 的类型强制转换为 void， 因为\
   在这个函数中， 这三个参数并没有使用。
#. 当哈希表中已经使用的空间达到 redis 哈希表最小填充， 即 REDIS_HT_MINFILL， 重新设\
   置哈希表的尺寸以达到节省内存的目的。 首先会用 dictGetHashTableSize_ 宏和 \
   dictGetHashTableUsed_ 宏来获取哈希表的大小以及以及使用的大小； 然后每 5 次定时检\
   测记录一次日志， 因为 ``loops % 5`` 只有在 loops 为 5 的整数倍的时候， 这个表达式\
   才能为 0， 才会执行第一个 if 语句中的 redisLog_ 函数； 然后当 ``size``， \
   ``used``， ``size > REDIS_HT_MINSLOTS`` 和 \
   ``(used*100/size < REDIS_HT_MINFILL)`` 都为真值的时候， 也就是当哈希表的大小大\
   于 16384， 且已使用的比率小于 10% 时， 就需要执行 if 内部的缩小哈希表大小的操作， \
   因为哈希表的大小比较大， 但是使用率低， 因此缩小以节省内存， 重置哈希表大小的函数是 \
   dictResize_
#. 每 5 次定时检测记录一次有多少个 client 在连接着 server， 这个数量是通过 \
   listLength_ 宏定义获取 server.clients 的长度拿到的。
#. 每 10 次检测， 断开连接超时的 clients， 执行的函数是 closeTimedoutClients_
#. 然后检测 redis 是否有后台进程用于持久化数据， 也就是保存数据。 当 \
   server.bgsaveinprogress 为真值非 0 时会执行 if 语句的内容， 否则执行 else 的内\
   容。 当为真值时， 说明有后台进程在进行数据的保存， 因此会执行 wait4 函数等待说有的\
   子进程， wait4 函数的第一个参数 -1 表示等待的是所有的子进程； 第二个参数 &statloc \
   表示的是存储的等待结果， 第 3 个参数 WNOHANG 表示非阻塞， 如果没有子进程退出就立刻\
   返回结果。 然后宏 WEXITSTATUS(statloc) 将等待的结果转换为 exitcode， 当 \
   exitcode 为 0 时记录 REDIS_NOTICE 级别的日志， 同时将 server.dirty 置为 0， \
   server.lastsave 置为当前时间； 否则的话记录 REDIS_WARNING 级别日志， 信息是后台\
   保存错误最终将 server.bgsaveinprogress 置为 0。 当没有后台保存进程的时候， 就需要\
   检测是否需要保存， 先获取当前时间， 然后判断修改的数量是否大于等于设定的数量， 同时\
   上次保存成功的时间与当前时间的间隔是否大于或等于设定的时间间隔， 如果是就记录日志， \
   同时执行 saveDbBackground_ 函数生成备份数据， 文件名为 dump.rdb
#. 如果一切 OK， 则该函数返回 1000。

..

  wait3 等待所有的子进程； wait4 可以像 waitpid 一样指定要等待的子进程： pid>0 表示\
  子进程ID； pid=0 表示当前进程组中的子进程； pid=-1 表示等待所有子进程； pid<-1 表\
  示进程组ID为pid绝对值的子进程。

.. _dictGetHashTableSize: beta-1-macros.rst#dictGetHashTableSize-macro
.. _dictGetHashTableUsed: beta-1-macros.rst#dictGetHashTableUsed-macro
.. _redisLog: beta-1-functions.rst#redisLog-func
.. _dictResize: beta-1-functions.rst#dictResize-func
.. _closeTimedoutClients: beta-1-functions.rst#closeTimedoutClients-func
.. _saveDbBackground: beta-1-functions.rst#saveDbBackground-func

.. _`dictResize-func`:
.. `dictResize-func`

24 dictResize 函数
===============================================================================

.. code-block:: C 

    /* This is the initial size of every hash table */
    #define DICT_HT_INITIAL_SIZE     16
    
    int dictResize(dict *ht)
    {
        int minimal = ht->used;

        if (minimal < DICT_HT_INITIAL_SIZE)
            minimal = DICT_HT_INITIAL_SIZE;
        return dictExpand(ht, minimal);
    }

重置字典哈希表的最小 size， 使其最小能容纳所有的节点， 且满足不等式 used/buckets 接\
近 <= 1。 

``DICT_HT_INITIAL_SIZE`` 为默认的哈希表大小， 其值为 16， 当已经使用的大小小于 16 \
的时候， 将 minimal 最小值设为 16， 否则就是哈希表已经使用的大小， 然后使用 \
dictExpand_ 函数进行字典大小的修改。

.. _dictExpand: #dictExpand-func

.. _`dictExpand-func`:
.. `dictExpand-func`

25 dictExpand 函数
===============================================================================

.. code-block:: C 

    /* Expand or create the hashtable */
    int dictExpand(dict *ht, unsigned int size)
    {
        // 1
        dict n; /* the new hashtable */
        unsigned int realsize = _dictNextPower(size), i;

        /* the size is invalid if it is smaller than the number of
        * elements already inside the hashtable */
        if (ht->used > size)
            return DICT_ERR;

        // 2
        _dictInit(&n, ht->type, ht->privdata);
        n.size = realsize;
        n.sizemask = realsize-1;
        n.table = _dictAlloc(realsize*sizeof(dictEntry*));

        // 3
        /* Initialize all the pointers to NULL */
        memset(n.table, 0, realsize*sizeof(dictEntry*));

        // 4
        /* Copy all the elements from the old to the new table:
        * note that if the old hash table is empty ht->size is zero,
        * so dictExpand just creates an hash table. */
        n.used = ht->used;
        for (i = 0; i < ht->size && ht->used > 0; i++) {
            dictEntry *he, *nextHe;

            if (ht->table[i] == NULL) continue;
            
            /* For each hash entry on this slot... */
            he = ht->table[i];
            while(he) {
                unsigned int h;

                nextHe = he->next;
                /* Get the new element index */
                h = dictHashKey(ht, he->key) & n.sizemask;
                he->next = n.table[h];
                n.table[h] = he;
                ht->used--;
                /* Pass to the next element */
                he = nextHe;
            }
        }

        // 5
        assert(ht->used == 0);
        _dictFree(ht->table);

        // 6
        /* Remap the new hashtable in the old */
        *ht = n;
        return DICT_OK;
    }

该函数用于扩展或创建哈希表。 按照代码注释， 大致分成 6 部分解析。

#. realsize 是 `_dictNextPower`_ 函数结果， 用于判断当前的 size 是否是在 2 的某一\
   次方内， 如果不在就将乘以 2； 然后判断哈希表已使用的大小是否大于哈希表的大小， 若是\
   返回 ``DICT_ERR`` 即 1
#. 对哈希表 n 进行初始化， 然后将哈希表的 size 置为 realsize， 同时 sizemask 置为 \
   realsize-1， table 置为哈希表分配 dictEntry 内存的地址
#. 将指向 n.table 的内存全部写成 0
#. 当旧的哈希表的大小不为 0 且有使用的大小时， 循环迭代复制每一个元素到新的哈希表中， \
   需要注意的是， 之前在 initServer_ 函数中使用的 sdsDictType_ 进行的初始化 dict 操\
   作， 因此在 dictHashKey_ 宏中使用的是 hash 函数是 sdsDictHashFunction_， 在此处\
   使用 ``dictHashKey(ht, he->key) & n.sizemask`` 是为了防止数组越界， 因为 \
   sizemask 一直比 size 小 1。 复制完成后将旧的 hash 表已使用大小减 1。 
#. 判断就的 hash 表已使用大小是否为 0， 为 0 说明复制完毕， 因为在复制的时候复制一个\
   就减 1。 然后在将旧的 hash 表释放
#. 然后将旧的 hash 表的指针指向新的拓展后的 hash 表。 之前步骤一切 OK 后， 返回 \
   DICT_OK 即 0

.. _`_dictNextPower`: #_dictNextPower-func
.. _`initServer`: beta-1-main-flow.rst#initServer-func
.. _`sdsDictType`: beta-1-others.rst#sdsDictType-var
.. _`dictHashKey`: beta-1-macros.rst#dictHashKey-macro
.. _`sdsDictHashFunction`: #sdsDictHashFunction-func

.. _`_dictNextPower-func`:
.. `_dictNextPower-func`

26 _dictNextPower 函数
===============================================================================

.. code-block:: C 

    /* Our hash table capability is a power of two */
    static unsigned int _dictNextPower(unsigned int size)
    {
        unsigned int i = DICT_HT_INITIAL_SIZE;

        if (size >= 2147483648U)
            return 2147483648U;
        while(1) {
            if (i >= size)
                return i;
            i *= 2;
        }
    }

redis 中的哈希表的容量都是 2 的整数次幂， 同时初始化的容量是 DICT_HT_INITIAL_SIZE \
即 16。

该函数用于判断一个 hash 表的大小是否应该放大乘以 2。 

- 当传入的参数大小大于等于 2147483648U， 直接返回 2147483648U
- 当哈希表的大小小于或等于初始容量， 返回初始容量表明无须扩大， 否则将 i 乘以 2 继续\
  判断。 直到 i 的值大于等于 hash 表的值， 并返回这个值

.. _`sdsDictHashFunction-func`:
.. `sdsDictHashFunction-func`

27 sdsDictHashFunction 函数
===============================================================================

.. code-block:: C 

    static unsigned int sdsDictHashFunction(const void *key) {
        return dictGenHashFunction(key, sdslen((sds)key));
    }

sdsDictType 类型的 hash 函数就是该函数

在该函数中执行 dictGenHashFunction_ 函数对 key 进行 hash 运算， 最终返回函数值

.. _dictGenHashFunction: #dictGenHashFunction-func

.. _`dictGenHashFunction-func`:
.. `dictGenHashFunction-func`

28 dictGenHashFunction 函数
===============================================================================

.. code-block:: C 

    /* Generic hash function (a popular one from Bernstein).
    * I tested a few and this was the best. */
    unsigned int dictGenHashFunction(const unsigned char *buf, int len) {
        unsigned int hash = 5381;

        while (len--)
            hash = ((hash << 5) + hash) + (*buf++); /* hash * 33 + c */
        return hash;
    }

传入的参数 len 有多少就执行多少次 hash 运算， 最终将运算结果返回。

.. _`closeTimedoutClients-func`:
.. `closeTimedoutClients-func`

29 closeTimedoutClients 函数
===============================================================================

.. code-block:: C 

    /* Directions for iterators */
    #define AL_START_HEAD 0
    #define AL_START_TAIL 1

    void closeTimedoutClients(void) {
        redisClient *c;
        listIter *li;
        listNode *ln;
        time_t now = time(NULL);

        li = listGetIterator(server.clients,AL_START_HEAD);
        if (!li) return;
        while ((ln = listNextElement(li)) != NULL) {
            c = listNodeValue(ln);
            if (now - c->lastinteraction > server.maxidletime) {
                redisLog(REDIS_DEBUG,"Closing idle client");
                freeClient(c);
            }
        }
        listReleaseIterator(li);
    }

此处需要先了解一下 redisClient_ 结构体和 listIter_ 结构体。

.. _redisClient: beta-1-structures.rst#redisClient-struct
.. _listIter: beta-1-structures.rst#listIter-struct

先获取当前的时间， 然后使用 listGetIterator_ 函数生成一个访问 List 的迭代器， 其中包\
含了访问方向。 代码中使用的是 AL_START_HEAD 即 0， 表示的是从 List 头节点开始访问。

.. _listGetIterator: #listGetIterator-func

当访问迭代器为空时， 直接返回。 正常时继续向下执行， 然后使用 listNextElement_ 获取下\
一个节点， 节点不为空时， 执行 listNodeValue_ 宏获取结点值。 当现在的时候与上次交互的\
时间间隔大于 server.maxidletime 时， 即大于超时时间， 就记录关闭 client 连接的日志， \
同时使用 freeClient_ 函数释放 client 连接。 

.. _listNextElement: #listNextElement-func
.. _listNodeValue: beta-1-macros.rst#listNodeValue-macro
.. _freeClient: #freeClient-func

最终使用 listReleaseIterator_ 函数释放 List 访问迭代器。

.. _listReleaseIterator: #listReleaseIterator-func

.. _`listGetIterator-func`:
.. `listGetIterator-func`

30 listGetIterator 函数
===============================================================================

.. code-block:: C 

    listIter *listGetIterator(list *list, int direction)
    {
        listIter *iter;
        
        if ((iter = malloc(sizeof(*iter))) == NULL) return NULL;
        if (direction == AL_START_HEAD)
            iter->next = list->head;
        else
            iter->next = list->tail;
        iter->direction = direction;
        return iter;
    }

从给定的 List 和 direction 生成一个 List 访问迭代器。 

如果分配迭代器内存失败， 直接返回 NULL。 当 direction 为 AL_START_HEAD 时， 表明是\
从头节点开始访问， 那么将迭代器 next 字段置为当前 List 的头节点； 否则就是从尾节点开\
始访问， 将 next 字段置为 List 的尾节点； 然后将其方向 direction 字段置为给定的 \
direction， 最终返回这个迭代器。

.. _`listNextElement-func`:
.. `listNextElement-func`

31 listNextElement 函数
===============================================================================

.. code-block:: C 

    listNode *listNextElement(listIter *iter)
    {
        listNode *current = iter->next;

        if (current != NULL) {
            if (iter->direction == AL_START_HEAD)
                iter->next = current->next;
            else
                iter->next = current->prev;
        }
        return current;
    }
    
声明 current 为当前节点， 其值为 List 访问迭代器的 next 指针， 如果 current 非空， \
当 iter 方向为从头节点开始时， 那么 iter->next 就是当前节点的 next 节点， 即 iter->\
next->next， 相当于 iter 向后移动了一个单位。 否则就向前移动。

最终返回 current 节点。 

.. _`freeClient-func`:
.. `freeClient-func`

32 freeClient 函数
===============================================================================

.. code-block:: C 

    #define AE_READABLE 1
    #define AE_WRITABLE 2
    #define AE_EXCEPTION 4

    static void freeClient(redisClient *c) {
        listNode *ln;

        aeDeleteFileEvent(server.el,c->fd,AE_READABLE);
        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
        sdsfree(c->querybuf);
        listRelease(c->reply);
        freeClientArgv(c);
        close(c->fd);
        ln = listSearchKey(server.clients,c);
        assert(ln != NULL);
        listDelNode(server.clients,ln);
        free(c);
    }

释放 client 连接， 需要进行一系列的操作：

#. aeDeleteFileEvent(server.el,c->fd,AE_READABLE); aeDeleteFileEvent_ 函数删除 \
   IO 读
#. aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE); aeDeleteFileEvent_ 函数删除 \
   IO 写
#. sdsfree_ 函数释放 client 查询缓冲区 
#. listRelease_ 函数释放 client reply 
#. freeClientArgv_ 函数释放 client 参数
#. close 关闭 client 连接
#. listSearchKey_ 从 server.clients 中搜索要释放的 client
#. 断言搜索结果是否为空， 为空说明 clients 列表中没有要释放的 client 
#. 正常情况下 ln 是不为空的， 使用 listDelNode_ 从 server.clients 将 client 删除
#. 最后释放 client 占用的内存

.. _`aeDeleteFileEvent`: #aeDeleteFileEvent-func
.. _`sdsfree`: #sdsfree-func
.. _`listRelease`: #listRelease-func
.. _`freeClientArgv`: #freeClientArgv-func
.. _`listSearchKey`: #listSearchKey-func
.. _`listDelNode`: #listDelNode-func

.. _`aeDeleteFileEvent-func`:
.. `aeDeleteFileEvent-func`

33 aeDeleteFileEvent 函数
===============================================================================

.. code-block:: C 

    void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)
    {
        aeFileEvent *fe, *prev = NULL;

        fe = eventLoop->fileEventHead;
        while(fe) {
            if (fe->fd == fd && fe->mask == mask) {
                if (prev == NULL)
                    eventLoop->fileEventHead = fe->next;
                else
                    prev->next = fe->next;
                if (fe->finalizerProc)
                    fe->finalizerProc(eventLoop, fe->clientData);
                free(fe);
                return;
            }
            prev = fe;
            fe = fe->next;
        }
    }

局部变量 fe 指的是当前 FileEvent， prev 指的是上一个 FileEvent。 

然后从第一个 FileEvent， 即 ``fe = eventLoop->fileEventHead`` 开始循环判断， 当当\
前 FileEvent 的 fd 与传递的 fd 相等且当前的 mask 与传递的 mask 相等时， 开始执行删除\
操作：

- 当 prev 为空， 说明是第一个 FileEvent， 那么直接将 fileEventHead 指向当前 \
  FileEvent 的 next； 否则就不是第一个 FileEvent， 直接将当前 FileEvent 的前一个的\
  next 指向当前 FileEvent 的 next， 直接略过当前 FileEvent， 表明删除
- 当当前 FileEvent 的 finalizerProc 指针有值时， 那么执行这个函数。 finalizerProc \
  是一个指向函数的指针。
- 删除后将当前 FileEvent 占用的内存释放， 并返回

如果不满足 if 条件， 则开始进行下一轮判断， 直到 fe 为空。

.. _`sdsfree-func`:
.. `sdsfree-func`

34 sdsfree 函数
===============================================================================

.. code-block:: C 

    void sdsfree(sds s) {
        if (s == NULL) return;
        free(s-sizeof(struct sdshdr));
    }

释放字符串对象内存。 当字符串 s 为空时直接返回； 否则将 sds 的对象释放掉。

``s-sizeof(struct sdshdr)`` 此处的意思是字符串和 sdshdr 整体。

.. code-block::

    |5|0|redis|
    ^   ^
    sh  sh->buf

sizeof(struct sdshdr) 实际上只是 len 和 free 字段的长度， buf 字段是不确定长度， 因\
此在 sizeof 计算时并没有包含在内。 那么 s 就是 buf 所在的指针， 因此此处 free 的时候\
就是连同 sdshdr 一起释放。

.. _`listRelease-func`:
.. `listRelease-func`

35 listRelease 函数
===============================================================================

.. code-block:: C 

    void listRelease(list *list)
    {
        int len;
        listNode *current, *next;

        current = list->head;
        len = list->len;
        while(len--) {
            next = current->next;
            if (list->free) list->free(current->value);
            free(current);
            current = next;
        }
        free(list);
    }

该函数用于释放整个 List， 会从第一个节点开始释放内存， 直到整个 list 完全释放。

current 从头节点开始， 如果指定了 ``list->free``， 那么就执行该函数释放当前结点的值。 \
否则直接释放当前结点， 同时将当前结点指向下一个节点。

最终释放 list 的内存。

