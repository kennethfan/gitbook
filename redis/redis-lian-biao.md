# redis链表

## 每个链表和链表节点的实现

### 链表节点结构

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
  	struct listNode *next;
    // 节点内容
    void *value;
} listNode;
```

### 链表结构

```c
typedef struct list {
    // 头节点
    listNode *head;
    // 尾节点
    listNode *tail;
    // 节点数量
    unsigned long len;
    // 节点复制函数
    void* (*dup) (void *ptr);
    // 节点释放函数
    void (*free) (void *ptr);
    // 节点对比函数
    int (*match) (void *prt, void *key);
} list;
```

dup、free、match成员用于实现多态链表所需的类型特定函数：

* dup函数用于复制链表节点所保存的值。
* free函数用于释放节点所保存的值。
* match函数用户对比链表节点所保存的值和另一个输入值是否相等。

Redis的链表实现特性

* 双端：链表节点带有prev和next指针，获取前置节点和后置节点的复杂度都是O(1)
* 无环：头结点的prev和尾节点的next都指向NULL
* 头节点和尾节点：获取表头和表尾复杂度都是O(1)
* 多态：链表节点都用void\*保存节点值，并且可以通过list结构的dup、free、match三个属性为节点设置类型特定函数，所以链表可以保存不同类型的值。

## API

* listSetDupMethod：将给定的函数设置为链表的节点复制函数，O(1)
* listGetDupMethod：返回链表正在使用的节点复制函数，O(1)
* listSetFreeMethod：将给定的函数设置成链表节点的释放函数，O(1)
* listGetFreeMethod：反馈链表正在使用的节点释放函数，O(1)
* listSetMatchMethod：将给定的函数设置成链表节点的对比函数，O(1)
* listGetMatchMethod：反馈链表正在使用的节点对比函数，O(1)
* listLength：返回链表长度，O(1)
* listFirst：返回链表头结点，O(1)
* listLast：返回链表尾节点，O(1)
* listPrevNode：返回前置节点，O(1)
* listNextNode：返回后置节点，O(1)
* listNodeValue：返回节点保存的值，O(1)
* listCreate：创建一个空链表，O(1)
* listAddNodeHead：添加节点到链表头，O(1)
* listAddNodeTail：添加节点到链表尾，O(1)
* listInsertNode：将新节点添加到给定节点之前或者之后，O(1)
* listSearchKey：反会链表中包含给定值的节点，O(N)
* listIndex：返回链表中给定索引的节点，O(N)
* listDelNode：删除给定节点，O(N)
* listRotate：弹出尾节点，并弹出，插入到表头，O(1)
* listDup：复制一个给链表，O(N)
* listRelease：释放给定链表以及链表中所有节点，O(N)
