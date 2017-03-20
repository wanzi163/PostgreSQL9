## makeNode()函数分析

makeNodede的定义，实际上是一个宏，调用的实际上经过类型转换的newNode()的返回结果。

```c
#define makeNode(_type_)        ((_type_ *) newNode(sizeof(_type_),T_##_type_))
```



实际上newNode也是一个宏，此时已经不会再有前面调用时候涉及到的类型信息（makeNode用的类型信息只是在最后对一块内存做一个类型转换，并且提供newNode做实际内存申请时候的一个内存空间大小的指标），newNode实际上是直接向当前MemoryContext申请的内存，而不是向操作系统申请（由palloc0fast函数来完成）。

需要注意的就是这个Tag数据，实际上是makeNode挟带的类型名称加上一个T_前缀来确定的，这是一个代码上面的约定，所有的node的tag都在src/include/nodes/nodes.h文件中定义了，所以这个约定是前提，否者会在编译的时候报错。

```c
#define newNode(size, tag) \
({  Node   *_result; \
    AssertMacro((size) >= sizeof(Node));        /* need the tag, at least */ \
    _result = (Node *) palloc0fast(size); \
    _result->type = (tag); \
    _result;               \
})
```

newNode实际上做了一个最简单的事情，就是在当前的MemoryContext中申请一个指定大小的空间，然后指定这个空间的第一个type字段为传入的tag，标识这个node的类型是什么。



Node的结构很简单，只是一个容纳tag的字段，注意后面再makeNode之后做类型转化你的类型的第一个字段也必须是NodeTag，否者就会出错。

```c
typedef struct Node
{
    NodeTag     type;
} Node;
```



实际场景中makenode的调用

```c
typedef struct VacuumStmt
{
    NodeTag     type;
    int         options;        /* OR of VacuumOption flags */
    RangeVar   *relation;       /* single table to process, or NULL */
    List       *va_cols;        /* list of column names, or NIL for all */
} VacuumStmt;
VacuumStmt *n = makeNode(VacuumStmt);
```



## list_make1()函数分析

```c
#define list_make1(x1)              lcons(x1, NIL)
```

调用lcons时候，第二个参数设置为NULL，表明这是list的第一个节点。

```c
List *
lcons(void *datum, List *list)
{
    Assert(IsPointerList(list));

    if (list == NIL)
        list = new_list(T_List);
    else
        new_head_cell(list);

    lfirst(list->head) = datum;
    check_list_invariants(list);
    return list;
}
```

lcons()函数的作用是在list列表结构中添加一个节点，在list列表的最前端添加这个新的节点。但是如果要添加的这个list结构是一个NULL值的话，就会创建一个新的list，并且把这个节点到这个新的list里。

```c
static List *
new_list(NodeTag type)
{
    List       *new_list;
    ListCell   *new_head;

    new_head = (ListCell *) palloc(sizeof(*new_head));
    new_head->next = NULL;
    /* new_head->data is left undefined! */

    new_list = (List *) palloc(sizeof(*new_list));
    new_list->type = type;
    new_list->length = 1;
    new_list->head = new_head;
    new_list->tail = new_head;

    return new_list;
}
```

新创建列表的时候，会默认的新创建一个空的节点(ListCell)。

```c
static void
new_head_cell(List *list)
{
    ListCell   *new_head;

    new_head = (ListCell *) palloc(sizeof(*new_head));
    new_head->next = list->head;

    list->head = new_head;
    list->length++;
}
```

创建一个新的节点数据结构，把这个新创建的节点放在整个list的最前面，他的next指针指向之前的最前面的节点。