# SDB_List

## DBR_List

双向链表

双向链表的基本操作

## DBR_RList

环形链表

## DBR_IntList

环形链表

列表对象可以被包含在每个数据结构中，从而将数据结构与所有ODB接口对象都连接到这个数据结构

列表对象的注册需要用到

通过 SetUserData 可以将数据从临时数据附加到永久数据上

管理 DBR_IntNode，在管理时，如果涉及到对 ODB 对象的修改，交给他们自己修改

### DBR_IntNode

在链表中的作用是作为一个节点

作为一个 ODB 对象的作用主要有：

1. 判断属性，通过各种函数来判断 ODB 当前的属性，主要通过 SDB_Types 判断
2. 修改属性，通过 ODB 对象设置属性的方法对属性进行修改，然后对 ODB 对象重新显示

## ODB_Container

ODB_Container 通过 ODB 的基类 DBR_IntNode 中定义的虚函数实现对 ODB 对象的访问

ODB_Container 可以在几何图形中访问特定的 ODB 对象 （例如可以通过剪切 Polyline 来获取一个 ODB_Container，然后调用 `ODB_Container::setselect` 选择对象

ODB_Container 里面包含一个 DBR_IntList 对象，ODB 对象应该通过 ODB_Container 来进行构造和析构

ODB_Container 中对 ODB 对象的修改都交给 ODB_IntNode 解决

