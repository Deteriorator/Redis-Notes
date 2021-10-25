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

