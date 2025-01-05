# redis字典

字典，又称符号表(symbol table)、关联数组(associative array)或映射(map)，是一种保存键值对的抽象数据结构

在字典中，一个键(key)可以和一个值(value)进行关联，这些关联的键和值就称为键值对。

字典中的每个键都是独一无二的，程序可以在字典中根据键查找与之关联的值，或者通过键来更新值，又或者根据键来删除整个键值对。

## 字典的实现

Redis字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

### 哈希表

```c
typedef struct dictht {
	// 哈希数组
  	dictEntry **table;
  	// 哈希表大小
  	unsigned long size;
  	// 哈希表大小掩码，用于计算索引，总是等于size-1
  	unsigned long sizemask;
  	// 哈希表已有节点的数量
  	unsigned long used;
} dictht;
```

table属性是一个节点数组

### 哈希表节点

```c
typedef struct dictEntry {
    // 键
  	void *key;
  	// 值
  	union {
        void *val;
      	uint64 tu64;
      	int64 ts64;
    } v;
  	// 指向下个哈希表节点，形成链表
  	struct dictEntry *next;
} dictEntry;
```

### 字典

```c
typedef struct dict {
    // 类型特定函数
  	dictType *type;
  	// 私有数据
  	void *privdata;
  	// 哈希表
  	dictht ht[2];
  	// rehash索引，当rehash不在进行时，值为-1
  	int rehashidx;
} dict;
```

```c
typedef struct dictType {
    // 计算哈希值的函数
  	unsigned int (*hashFunction)(const void *key);
  	// 复制键的函数
  	void* (*keyDup)(void *privData, const void *key);
  	// 复制值的函数
  	void* (*valDup)(void *privdata, const void *obj);
  	// 对比键的函数
  	int (*keyCompare)(void *privdata, const void *key1, const void* key2);
  	// 销毁键的函数
  	void (*keyDestructor)(void *privdata, void *key1);
  	// 销毁值的函数
  	void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下只是用ht\[0]哈希表，ht\[1]哈希表只会对ht\[0]进行rehash时使用。

除了ht\[1]之外，另一个和rehash有关的属性就是rehashidx，它记录了rehash的目前进度，如果没有在进行rehash，那么它的值为-1。

## 哈希算法

当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。

```c
// 使用字典设置的哈希函数，计算key的哈希值
hash = dict->type->hashFunction(key);
// 使用哈希表的sizemask属性和哈希值，计算出索引
根据情况不同，ht[x]可以是ht[0]或ht[1]
index = hash & dict->ht[x].sizemask
```

当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值

MurmurHash算法最初有Austin Appleby于2008年发明，这种算法的有点在于，即使输入的键是有规律的，算法仍能给出一个很好的随机分布性，并且算法的计算速度也很快。

## 阶段键冲突

当有两个或者以上的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突(collision)。

Redis的哈希表使用链地址法(separate chaining)来解决冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单链表，被分配到同一个索引上的多个节点可以用这个单向链表l连接起来，这就解决了键冲突的问题。

## rehash

随着操作的不断执行，哈希表保存的键值对会逐渐的增多或者减少，为了让哈希表d额负载因子(load factor)维持在一个合理的范围之内，当哈希表保存的j键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。

扩展或者搜索哈希表的工作可以通过执行rehash操作来完成，步骤如下

1. 为字典的ht\[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht\[0]当前包含的键值对数量（ht\[0].used属性的值）
   * 如果执行的是扩展操作，那么ht\[1]的大小为第一个大于等于ht\[0].used\*2的的2^n
   * 如果执行的是收缩操作，那么ht\[1]的大小为第一个大于等于ht\[0].used\*2的2^n
2. 将保存在ht\[0]的所有键值对rehash到ht\[1]上；rehash值的是重新计算键的哈希值和索引值，然后将键值对放置到ht\[1]的指定位置上
3. 当ht\[2]包含的所有键值对都迁移到ht\[1]之后，释放ht\[0]，将ht\[1]设置成ht\[0]，并为ht\[1]新创建空哈希表，为下次rehash做准备

### 哈希表的扩展与收缩

当以下条件中的任意一个呗满足是，程序会自动对哈希表执行扩展操作

1. 服务器目前没有正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。
2. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。

负载因子计算公式

```c
load_factor = ht[0].used / ht[0].size
```

另一方面，当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。

## 渐进式rehash

上一节说过，扩展或收缩哈希表需要将ht\[0]里面的所有键值对rehash到ht\[1]里面，但是，这个rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。

渐进式rehash的步骤

1. 为ht\[1]分配空间，让字典同时持有ht\[0]和ht\[1]两个hash表。
2. 更新rehashidx的值为0，表示rehash正式开始。
3. rehash期间，每次对字典执行添加、删除、查找或者更新时，程序处理执行指定的操作以外，还会顺带将ht\[0]哈希表在rehashidx索引上的所有键值对rehash到ht\[1]，当rehash工作完成之后，程序将rehashidx加1。
4. 随着字典操作的不断执行，最终在某一时刻，ht\[0]的所有键值对都会被rehash到ht\[1]上，这时更新rehashidx的值为-1。

## 字典API

* dictCreate：创建一个新的字典，0(1)
* dictAdd：将给定的键值对添加到字典里面，O(1)
* dictReplace：将给定的键值对添加到字典里面，如果键已经存在则覆盖，O(1)
* dictFetchValue：返回给定键的值，O(1)
* dictGetRandomKey：从字典中随机返回一个键值对，O(1)
* dictDelete：从字典中删除给定键所对应的键值对，O(1)
* dictRelease：释放给定字典，以及字典中包含的所有键值对，O(N)
