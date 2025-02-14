# SDB_Relationships

## DBR_One(sdb_relationships_dbr_one.h)

DBR_One 是一对一关系中的 Child，有一个父节点的指针

当 garbage 或者 loading file 的时候可以修改父节点的地址

当数据修改时可以提醒父节点

## DBR_One2One(sdb_relationships_dbr_one2one.h)

DBR_One2One 是一对一关系中的 Parent，有一个子节点的指针

当 garbage 或者 loading file 的时候可以修改子节点的地址

**RecordModifyObject  NotifyDataChanged ？**

## DBR_Several(SDB_Relationships_Template.h)

DBR_Sevreal 是一对 C[N] 关系中的 Children，有一个父节点指针

当 garbage 或者 loading file 的时候可以修改父节点的地址

当数据修改时可以提醒父节点

## DBR_One2Several(SDB_Relationships_Template.h)

DBR_Sevreal 是一对 C[N] 关系中的 parent，有一个子节点数组指针 **child， *children

当 garbage 或者 loading file 的时候可以修改子节点的地址

## DBR_Variant(SDB_Relationships_Template.h)

DBR_Variant是一对 v(C) 关系中的 Children，有一个父节点指针

当 garbage 或者 loading file 的时候可以修改父节点的地址

## DBR_StrongOne2Variant(SDB_Relationships_Template.h)

P -> v(C)

 Exclude（parent）解除父子联系

Include 建立父子联系

UpdatePointers 在 read file 时更新地址

## DBR_Many(sdb_relationships_dbr_many.h)

是一个链表子节点的链表