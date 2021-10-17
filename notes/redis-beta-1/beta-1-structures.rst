##############################################################################
Redis Beta 1 源码阅读笔记 - Structures
##############################################################################

.. contents::

******************************************************************************
第 1 部分  Structures
******************************************************************************

.. _redisServer-structure:
.. redisServer-structure

1.1 redisServer 结构体
==============================================================================

redisServer 结构体一共由 17 个子元素构成

.. code-block:: c

    /* Global server state structure */
    struct redisServer {
        int port;                     // 端口
        int fd;                       // 文件描述符
        dict **dict;                  // 哈希表
        long long dirty;              /* changes to DB from the last save */
        list *clients;                // 客户端列表 
        char neterr[ANET_ERR_LEN];    // 
        aeEventLoop *el;              // 事件循环
        int verbosity;                // 
        int cronloops;                // 
        int maxidletime;              // 超时时间
        int dbnum;                    // 数据库数量
        list *objfreelist;            /* A list of freed objects to avoid malloc() */
        int bgsaveinprogress;         //
        time_t lastsave;              // 最新保存时间
        struct saveparam *saveparams; //
        int saveparamslen;            //
        char *logfile;                // 日志文件
    };

.. _dict-structure:
.. dict-structure

1.2 dict 结构体
==============================================================================

.. code-block:: c 

    typedef struct dict {
        dictEntry **table;     // 哈希表结点指针数组
        dictType *type;        // 类型
        unsigned int size;     // 指针数组的大小
        unsigned int sizemask; // 长度掩码， 当使用下标访问数据时，确保下标不越界
        unsigned int used;     // 哈希表现有的节点数量
        void *privdata;        // 字典的私有数据
    } dict;

sizemask 字段的作用是当使用下标访问数据时， 确保下标不越界。

例如当前 size 为 8 时， sizemask 为 7 (0x111)。 当给定一个下标N时， 将 N 与 \
sizemask 进行与操作后得出下标才是最终使用的下标， 这是一个绝对不会越界的下标。 

.. _dictEntry-structure:
.. dictEntry-structure

1.3 dictEntry 结构体
==============================================================================

.. code-block:: c 

    typedef struct dictEntry {
        void *key;              // 键
        void *val;              // 值
        struct dictEntry *next; // 下一个结点
    } dictEntry;

dictEntry 就是 Dict (Hash Table) 的节点或条目， 每个条目都有 key， value 和下一个\
条目的地址

.. _dictType-structure:
.. dictType-structure

1.4 dictType 结构体
==============================================================================

.. code-block:: c

    typedef struct dictType {
        unsigned int (*hashFunction)(const void *key);
        void *(*keyDup)(void *privdata, const void *key);
        void *(*valDup)(void *privdata, const void *obj);
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
        void (*keyDestructor)(void *privdata, void *key);
        void (*valDestructor)(void *privdata, void *obj);
    } dictType;

dictType 结构包含若干函数指针， 用于 dict 的调用者对涉及 key 和 value 的各种操作进\
行自定义。 这些操作包含：

- hashFunction， 对 key 进行哈希值计算的哈希算法。
- keyDup 和 valDup， 分别定义 key 和 value 的拷贝函数， 用于在需要的时候对 key 和 \
  value 进行深拷贝， 而不仅仅是传递对象指针。
- keyCompare， 定义两个 key 的比较操作， 在根据 key 进行查找时会用到。
- keyDestructor 和 valDestructor， 分别定义对 key 和 value 的销毁函数。 私有数据\
  指针 （privdata） 就是在 dictType 的某些操作被调用时会传回给调用者。

.. _list-structure:
.. list-structure

1.5 list 结构体
==============================================================================

.. code-block:: c 

    typedef struct list {
        listNode *head; // 头节点
        listNode *tail; // 尾节点
        void *(*dup)(void *ptr);
        void (*free)(void *ptr);
        int (*match)(void *ptr, void *key);
        int len;
    } list;

list 是一个双向链表， 含有头节点和尾节点及链表的长度， 另外还有 3 个函数指针， 分别是 \
dup 、 free 和 match ：

- dup: 节点拷贝函数， 用于在需要的时候对节点进行深拷贝
- free: 节点释放函数
- match: 节点匹配函数

.. _listNode-structure:
.. listNode-structure

1.6 listNode 结构体
==============================================================================

.. code-block:: c 

    typedef struct listNode {
        struct listNode *prev; // 上一个节点地址
        struct listNode *next; // 下一个节点地址
        void *value;           // 当前结点的值
    } listNode;

双向链表的节点， 含有 3 个元素， 分别是上一个节点地址， 下一个节点地址以及当前结点的\
值。 

.. _aeEventLoop-structure:
.. aeEventLoop-structure

1.7 aeEventLoop 结构体
==============================================================================

.. code-block:: c 

    /* State of an event based program */
    typedef struct aeEventLoop {
        long long timeEventNextId;
        aeFileEvent *fileEventHead;
        aeTimeEvent *timeEventHead;
        int stop;
    } aeEventLoop;

事件循环结构体

- ``timeEventNextId``: 用于生成时间事件的唯一标识 id
- ``fileEventHead``:  注册的事件链表头指针
- ``timeEventHead``: 注册的时间事件链表头指针
- ``stop``: 停止标志， 1 表示停止

.. _aeFileEvent-structure:
.. aeFileEvent-structure

1.8 aeFileEvent 结构体
==============================================================================

.. code-block:: c 

    /* File event structure */
    typedef struct aeFileEvent {
        int fd;
        int mask; /* one of AE_(READABLE|WRITABLE|EXCEPTION) */
        aeFileProc *fileProc;
        aeEventFinalizerProc *finalizerProc;
        void *clientData;
        struct aeFileEvent *next;
    } aeFileEvent;

aeFileEvent 文件事件结构体， 实际上是一个链表

- ``fd``: 文件描述符
- ``mask``: 标识这是一个读事件或写事件还是一个异常
- ``fileProc``: 事件处理函数
- ``finalizerProc``: 事件从链表中删除是执行的函数
- ``clientData``: 传递给事件处理函数的数据
- ``next``: 下一个事件的地址


