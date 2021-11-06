###############################################################################
Redis Beta 1 源码阅读笔记 - Macros
###############################################################################

.. contents::

*******************************************************************************
Macros
*******************************************************************************

.. _listLength-macro:
.. listLength-macro

01 listLength 宏定义
===============================================================================

.. code-block:: c 

    #define listLength(l) ((l)->len)

此宏定义是获取 list_ 的 len 属性， 即长度。 代码中的 l 的类型是 list_。

.. _list: beta-1-structures.rst#list-structure

.. _listFirst-macro:
.. listFirst-macro

02 listFirst 宏定义
===============================================================================

.. code-block:: c 

    #define listFirst(l) ((l)->head)

此宏定义是获取 list_ 的 head 属性， 即头节点指针。 代码中的 l 的类型是 list_。

.. _listNodeValue-macro:
.. listNodeValue-macro

03 listNodeValue 宏定义
===============================================================================

.. code-block:: c 

    #define listNodeValue(n) ((n)->value)

此宏定义是获取 listNode_ 的 value 属性， 即节点值的地址。 代码中的 n 的类型是 \
listNode_。

.. _listNode: beta-1-structures.rst#listNode-struct

.. _`REDIS_NOTUSED-macro`:
.. REDIS_NOTUSED-macro

04 REDIS_NOTUSED 宏定义
===============================================================================

.. code-block:: c 

    /* Anti-warning macro... */
    #define REDIS_NOTUSED(V) ((void) V)

为了避免警告， 将 V 的类型强制转换为 void。 

.. _`dictGetHashTableSize-macro`:
.. dictGetHashTableSize-macro

05 dictGetHashTableSize 宏定义
===============================================================================

.. code-block:: c

    #define dictGetHashTableSize(ht) ((ht)->size)

获取哈希表的大小。

.. _`dictGetHashTableUsed-macro`:
.. dictGetHashTableUsed-macro

06 dictGetHashTableUsed 宏定义
===============================================================================

.. code-block:: c

    #define dictGetHashTableUsed(ht) ((ht)->used)

获取哈希表已经使用的大小。

.. _`dictHashKey-macro`:
.. dictHashKey-macro

07 dictHashKey 宏定义
===============================================================================

.. code-block:: c

    #define dictHashKey(ht, key) (ht)->type->hashFunction(key)

用于获取不同类型的 hashFunction 函数指针。

.. _`dictGetEntryKey-macro`:
.. dictGetEntryKey-macro

08 dictGetEntryKey 宏定义
===============================================================================

.. code-block:: c

    #define dictGetEntryKey(he) ((he)->key)

用于获取哈希表条目的 key， he 就是 hashtable entry 的 缩写， 是一个 dictEntry_ 结构\
体， 直接获取其 key 字段

.. _dictEntry: beta-1-structures.rst#dictEntry-struct

.. _`dictGetEntryVal-macro`:
.. dictGetEntryVal-macro

09 dictGetEntryVal 宏定义
===============================================================================

.. code-block:: c

    #define dictGetEntryVal(he) ((he)->val)

用于获取哈希表条目的 val， he 就是 hashtable entry 的 缩写， 是一个 dictEntry_ 结构\
体， 直接获取其 val 字段

