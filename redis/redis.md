> 基于Redis 2.9，适用于Redis 2.6至Redis 3.0

## 数据结构

### SDS
simple dynamic string，简单动态字符串，Redis的默认字符串表示。  
字符串、哈希表、列表、集合等数据类型的键值对底层都是由SDS 实现的，因为Redis 需要表示一个可以被修改的字符串值。  
SDS还被用作缓冲区（buffer）：AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区。

在Redis里，C语言的字符串只会作为字符串字面量（string literal）用在一些无须对字符串值进行修改的地方，比如打印日志。

每个sds.h/sdshdr结构表示一个SDS值：

```
struct sdshdr {
    //记录buf 数组中已使用字节的数量，等于SDS 所保存字符串的长度
    int len;
    //记录buf 数组中未使用字节的数量
    int free;
    //字节数组，用于保存字符串
    char buf[];
};
```

SDS 图示：  
![SDS示例](../images/redis/2024-02-16_SDS示例.png ':size=30%')  
> free属性的值为0，表示这个SDS没有分配任何未使用空间  
len属性的值为5，表示这个SDS保存了一个5字节长的字符串  
buf属性是一个char类型的数组，数组的前五个字节分别保存了'R'、'e'、'd'、'i'、's'五个字符，而最后一个字节则保存了空字符'\0'

SDS 遵循C字符串以空字符结尾的惯例，好处是SDS 可以直接重用一部分C字符串函数库里面的函数。

##### Ⅰ.常数复杂度获取字符串长度
C语言使用长度为N+1的字符数组来表示长度为N的字符串，并且字符数组的最后一个元素总是空字符'\0'。  
C字符串并不记录自身的长度信息，为了获取一个C字符串的长度，程序必须遍历整个字符串，直到遇到代表字符串结尾的空字符为止，这个操作的复杂度为O(N)。  
SDS在len属性中记录了字符数组的长度，所以获取一个SDS长度的复杂度仅为O(1)。  
因为采用SDS这种数据结构，Redis即使反复获取一个非常长的字符串的长度，也不会对系统性能造成影响。

##### Ⅱ.杜绝缓冲区溢出
C字符串不记录自身长度带来的另一个问题是容易造成缓冲区溢出（buffer overflow）。举个例子，<string.h>/strcat函数可以将src字符串中的内容拼接到dest字符串的末尾：  
```
char *strcat(char *dest, const char *src);
```
在执行strcat函数前，如果没有为dest分配足够的空间去容纳src字符串的内容，就会覆盖掉相邻内存的内容，这就产生了缓冲区溢出。

当对SDS进行修改时，SDS API会先检查SDS的空间是否满足修改所需的要求，如果不满足，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作。  
所以使用SDS既不需要手动修改SDS的空间大小，也不会出现C字符串的缓冲区溢出问题。

##### Ⅲ.减少修改字符串时带来的内存重分配次数
每次增长或者缩短一个C字符串，程序总要对保存这个C字符串的数组进行一次内存重分配操作：

    ※如果程序执行的是增长字符串的操作，比如拼接操作（append），那么在执行这个操作之前，程序需要先通过内存重分配来扩展底层数组的空间大小——如果忘了这一步就会产生缓冲区溢出。  
    ※如果程序执行的是缩短字符串的操作，比如截断操作（trim），那么在执行这个操作之后，程序需要通过内存重分配来释放字符串不再使用的那部分空间——如果忘了这一步就会产生内存泄漏。

因为内存重分配涉及复杂的算法，并且可能需要执行系统调用，所以它通常是一个比较耗时的操作。  
如果修改字符串长度的情况不太常出现，那么每次修改都执行一次内存重分配是可以接受的。  
但是Redis经常被用于速度要求严苛、数据被频繁修改的场合，如果每次修改字符串的长度都需要执行一次内存重分配的话，那么光是执行内存重分配的时间就会占去修改字符串所用时间的一大部分，如果这种修改频繁地发生的话，可能还会对性能造成影响。

为了避免C字符串的这种缺陷，SDS通过“未使用空间”解除了字符串长度与底层数组长度之间的关联：在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含未使用的字节，而这些字节的数量就由SDS的free属性记录。  
通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

1. **空间预分配**

空间预分配用于优化SDS的字符串增长操作：当SDS的API对一个SDS进行修改，并且需要对SDS进行空间扩展的时候，程序不仅会为SDS分配修改所必须要的空间，还会为SDS分配额外的未使用空间。  
在扩展SDS空间之前，SDS API会先检查未使用空间是否足够，如果足够的话，API就会直接使用未使用空间，而无须执行内存重分配。

额外分配的未使用空间数量的规则：
> - 如果对SDS进行修改之后，SDS的长度（也就是len属性的值）将小于1MB，那么程序分配和len属性同样大小的未使用空间，这时SDS len属性的值将和free属性的值相同。  
例如，如果进行修改之后，SDS的len将变成13字节，那么程序也会分配13字节的未使用空间，SDS的buf数组的实际长度将变成13+13+1=27字节（额外的一字节用于空字符）。  
> - 如果对SDS进行修改之后，SDS的长度将大于等于1MB，那么程序会分配1MB的未使用空间。  
例如，如果进行修改之后，SDS的len将变成30MB，那么程序会分配1MB的未使用空间，SDS的buf数组的实际长度将为30MB+1MB+1byte。

通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数。SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次。

2. **惰性空间释放**

惰性空间释放用于优化SDS的字符串缩短操作：当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。

通过惰性空间释放策略，SDS避免了缩短字符串时所需的内存重分配操作，并为将来可能有的增长操作提供了优化。  
SDS提供了相应的API，可以在有需要时，真正地释放SDS的未使用空间。

##### Ⅳ.二进制安全
C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些限制使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

SDS的API都是二进制安全的（binary-safe），所有SDS API都会以处理二进制的方式来处理SDS存放在buf数组里的数据，程序不会对其中的数据做任何限制、过滤、或者假设，数据在写入时是什么样的，它被读取时就是什么样。  
这也是我们将SDS的buf属性称为字节数组的原因——Redis不是用这个数组来保存字符，而是用它来保存一系列二进制数据。  
当遇到使用'\0'空字符来分割单词的特殊数据格式，SDS也没有问题，因为SDS使用len属性的值而不是空字符来判断字符串是否结束。

##### Ⅴ.兼容部分C字符串函数
通过遵循C字符串以空字符结尾的惯例，SDS可以在有需要时重用<string.h>函数库，从而避免了不必要的代码重复。

例如，重用<string.h>/strcasecmp函数来对比SDS保存的字符串和另一个C字符串；重用<string.h>/strcat函数来将SDS保存的字符串追加到一个C字符串的后面。

#### C字符串和SDS之间的区别

C字符串 | SDS
 :---- | :----
获取字符串长度的复杂度为O(N) | 获取字符串长度的复杂度为O(1)
API不安全，可能造成缓冲区溢出 | API安全，不会造成缓冲区溢出
修改字符串长度N次必然执行N次内存重分配 | 修改字符串长度N次最多执行N次内存重分配
只能保存文本数据 | 可以保存文本或者二进制数据
可以使用所有<string.h>库中的函数 | 可以使用部分<string.h>库中的函数

#### SDS API
SDS的主要操作API：

函数 | 作用 | 时间复杂度
 ---- | :---- | :----
sdsnew | 创建一个包含给定C字符串的SDS | O(N)，N为给定C字符串的长度
sdsempty | 创建一个不包含任何内容的空SDS | O(1)
sdsfree | 释放给定的SDS | O(N)，N为被释放SDS的长度
sdslen | 返回SDS的已使用空间字节数 | O(1)，这个值可以通过读取SDS的len属性获得
sdsavail | 返回SDS的未使用空间字节数 | O(1)，这个值可以通过读取SDS的free属性获得
sdsdup | 创建一个给定的SDS的副本(copy) | O(N)，N为给定SDS的长度
sdsclear | 清空SDS保存的字符串内容 | O(1)，因为惰性空间释放策略
sdscat | 将给定C字符串拼接到SDS字符串末尾 | O(N)，N为被拼接C字符串的长度
sdscatsds | 将给定SDS字符串拼接到另一个SDS字符串末尾 | O(N)，N为被拼接SDS字符串的长度
sdscpy | 将给定C字符串复制到SDS里面，覆盖原字符串 | O(N)，N为被复制C字符串的长度
sdsgrowzero | 用空字符将SDS扩展至给定长度 | O(N)，N为扩展新增的字节数
sdsrange | 保留SDS指定区间内的数据，不在区间内的数据会被覆盖或清除 | O(N)，N为被保留数据的字节数
sdstrim | 接受一个SDS和一个C字符串作为参数，从SDS中移除所有在C字符串中出现过的字符 | O(N²)
sdscmp | 对比两个SDS字符串是否相同 | O(N)，N为两个SDS中较短的那个SDS的长度


### 链表
链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

因为Redis使用的C语言并没有内置这种数据结构，所以Redis构建了自己的链表实现。

当一个列表键包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现。  
发布与订阅、慢查询、监视器等功能也用到了链表，Redis服务器本身还使用链表来保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区（output buffer）。

```
redis> LLEN integers
(integer) 1024
redis> LRANGE integers 0 2
1)"1"
2)"2"
3)"3"
```

integers列表键的底层实现就是一个链表，链表中的每个节点都保存了一个整数值。

每个链表节点使用一个adlist.h/listNode结构来表示：  
```
typedef struct listNode {
    //前置节点
    struct listNode * prev;
    //后置节点
    struct listNode * next;
    //节点的值
    void * value;
}listNode;
```

多个listNode可以通过prev和next指针组成双端链表：  
![double-ended_listNode](../images/redis/2024-02-17_双端链表.png ':size=50%')

用adlist.h/list来持有链表：  
```
typedef struct list {
    //表头指针
    listNode * head;
    //表尾指针
    listNode * tail;
    //链表所包含的节点数量
    unsigned long len;
    //复制链表节点所保存的值
    void *(*dup)(void *ptr);
    //释放链表节点所保存的值
    void (*free)(void *ptr);
    //对比链表节点所保存的值和另一个输入值是否相等
    int (*match)(void *ptr,void *key);
} list;
```

由一个list结构和三个listNode结构组成的链表：  
![list+listNode](../images/redis/2024-02-17_list+listNode.png ':size=50%')

##### Redis的链表实现的特性
- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)
- 带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O(1)
- 多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值

#### 链表和链表节点的API

 函数 | 作用 |时间复杂度
 :----: | :---- | :----
listSetDupMethod | 将给定的函数设置为链表的节点值复制函数 | O(1)，复制函数可以通过链表的dup属性获得
listGetDupMethod | 返回链表当前正在使用的节点值复制函数 | O(1)
listSetFreeMethod | 将给定的函数设置为链表的节点值释放函数 | O(1)，释放函数可以通过链表的free属性获得
listGetFree | 返回链表当前正在使用的节点值释放函数 | O(1)
listSetMatchMethod | 将给定的函数设置为链表的节点值对比函数 | O(1)，对比函数可以通过链表的match属性获得
listGetMatchMethod | 返回链表当前正在使用的节点值对比函数 | O(1)
listLength | 返回链表的长度（包含多少节点）| O(1)，链表长度可以通过链表的len属性获得
listFirst | 返回链表的表头节点 | O(1)，表头节点可以通过链表的head属性获得
listLast | 返回链表的表尾节点 | O(1)，表尾节点可以通过链表的tail属性获得
listPrevNode | 返回给定节点的前置节点 | O(1)，前置节点可以通过节点的prev属性获得
listNextNode | 返回给定节点的后置节点 | O(1)，后置节点可以通过节点的next属性获得
listNodeValue | 返回给定节点目前正在保存的值 | O(1)，节点值可以通过节点的value属性获得
listCreate | 创建一个不包含任何节点的新链表 | O(1)
listAddNodeHead | 将一个包含给定值的新节点添加到给定链表的表头 | O(1)
listAddNodeTail | 将一个包含给定值的新节点添加到给定链表的表尾 | O(1)
listInsertNode | 将一个包含给定值的新节点添加到给定节点的之前或之后 | O(1)
listSearchKey | 查找并返回链表中包含给定值的节点 | O(N)，N为链表长度
listIndex | 返回链表在给定索引上的节点 | O(N)，N为链表长度
listDelNode | 从链表中删除给定节点 | O(N)，N为链表长度
listRotate | 将链表的表尾节点弹出，然后将被弹出的节点插入到链表表头，成为新的表头节点 | O(1)
listDup | 复制一个给定链表的副本 | O(N)，N为链表长度
listRelease | 释放给定链表，以及链表中的所有节点 | O(N)，N为链表长度


### 字典
又称为符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对（key-value pair）的抽象数据结构。  
在字典中，一个键（key）可以和一个值（value）进行关联（或者说将键映射为值），这些关联的键和值就称为键值对。

Redis的数据库、哈希键等都是基于字典实现的。
```
redis> SET msg "hello world"
```
在数据库中创建一个键为"msg"，值为"hello world"的键值对时，这个键值对就是保存在代表数据库的字典里面的。

```
redis> HLEN website
(integer) 10086
redis> HGETALL website
1)"Redis"
2)"Redis.io"
3)"MariaDB"
4)"MariaDB.org"
# ...
```
website这个哈希键的底层实现就是一个字典，字典中包含了10086个键值对。例如，键值对的键为"Redis"，值为"Redis.io"。

Redis的字典结构图：  
![dict_structure](../images/redis/2024-02-18_dict_structure.png ':size=50%')

**哈希表节点**使用dictEntry结构定义，每个dictEntry结构都保存着一个键值对：  
```
typedef struct dictEntry {
    //键
    void *key;
    //值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

**哈希表**由dict.h/dictht结构定义：  
```
typedef struct dictht {
    //哈希表数组
    dictEntry **table;
    //哈希表大小，记录table数组的大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

**字典**由dict.h/dict结构定义：  
```
typedef struct dict {
    //类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    //哈希表
    dictht ht[2];
    // rehash索引，当不进行rehash时，值为-1
    int rehashidx;
} dict;
```

> type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的：  
> - type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
> - privdata属性则保存了需要传给那些类型特定函数的可选参数。
> - ht属性是一个包含两个项的数组，数组中的每项都是一个dictht哈希表，一般情况，字典只使用ht[0]哈希表，ht[1]哈希表只会在进行rehash时使用。
> - rehashidx用来记录rehash目前的进度，如果目前没有在进行rehash，那么它的值为-1。

```
typedef struct dictType {
    //计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    //复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    //复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    //对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    //销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

#### 哈希算法
当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。Redis计算哈希值和索引值的方法如下：  

    #使用字典设置的哈希函数，计算键key的哈希值
    hash = dict->type->hashFunction(key);
    #使用哈希表的sizemask属性和哈希值，计算出索引值（根据情况不同，ht[x]可以是ht[0]或者ht[1]）
    index = hash & dict->ht[x].sizemask;

举例，如果将一个键值对k0和v0添加到字典里：  
```
//计算键k0的哈希值，假设计算得出的哈希值为8
hash = dict->type->hashFunction(k0);
//根据hash=8计算出键k0的索引值0
index = hash & dict->ht[0].sizemask
= 8 & 3 = 0;
```

根据公式的计算结果将键值对k0、v0的节点放置到哈希表数组的索引0位置上，如图：  
![hash_algorithm](../images/redis/2024-02-18_hash_algorithm.png ':size=45%')

当字典被用作数据库或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。  
MurmurHash算法最初由Austin Appleby于2008年发明，这种算法的优点在于，即使输入的键是有规律的，算法仍能给出一个很好的随机分布性，并且算法的计算速度也非常快。MurmurHash算法目前的最新版本为MurmurHash3，而Redis使用的是MurmurHash2，关于MurmurHash算法的更多信息可以参考该算法的主页：http://code.google.com/p/smhasher/

##### 解决键冲突
当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突（collision）。  
Redis的哈希表使用链地址法（separate chaining）来解决键冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。  
因为dictEntry节点组成的链表没有指向链表表尾的指针，所以为了速度考虑，程序总是将新节点添加到链表的表头位置（复杂度为O（1）），排在其他已有节点的前面。

##### rehash
随着操作的不断执行，哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。

1. 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是ht[0].used属性的值）：

    如果执行的是扩展操作，那么ht[1]的大小 = 大于等于ht[0].used*2的最小 2ⁿ  
    如果执行的是收缩操作，那么ht[1]的大小 = 大于等于ht[0].used的最小 2ⁿ

2. 将保存在ht[0]中的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对从ht[0]放置到ht[1]哈希表的指定位置上。  
3. 当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

###### 哈希表的扩展与收缩
**哈希表的扩展条件**：
- 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
- 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5

**哈希表的收缩条件**：
- 当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作

```
#负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
```

<span style="color: red;font-weight: bold;">Tips</span>：根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，服务器执行扩展操作所需的负载因子并不相同，这是因为在执行BGSAVE命令或BGREWRITEAOF命令的过程中，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率，所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，从而尽可能地避免在子进程存在期间进行哈希表扩展操作，这可以避免不必要的内存写入操作，最大限度地节约内存。

###### 渐进式rehash
如果哈希表里的键值对数量特别多，要一次性将这些键值对全部rehash到ht[1]的话，庞大的计算量可能会导致服务器在一段时间内停止服务。因此，为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。

哈希表**渐进式rehash**的详细步骤：  
1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表。
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。
3. 在rehash进行期间，程序除了对字典执行增删改查操作外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

渐进式rehash的好处在于它采取分而治之的方式，将rehash键值对所需的计算工作均摊到对字典的增删改查操作上，从而避免了集中式rehash而带来的庞大计算量。

<span style="color: red;font-weight: bold;">Tips</span>：在进行渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。例如，要在字典里面查找一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找。  
另外，在渐进式rehash执行期间，新添加到字典的键值对一律会被保存到ht[1]里面，而ht[0]则不再进行任何添加操作，这一措施保证了ht[0]包含的键值对数量会只减不增，并随着rehash操作的执行而最终变成空表。

#### 字典API
函数 | 作用 | 时间复杂度
 :----: | :---- | :----
dictCreate | 创建一个新的字典 | O(1)
dictAdd | 将给定的键值对添加到字典里 | O(1)
dictReplace | 将给定的键值对添加到字典里，如果键已存在，那么用新值取代原值 | O(1)
dictFetchValue | 返回给定键的值 | O(1)
dictGetRandomKey | 从字典中随机返回一个键值对 | O(1)
dictDelete | 从字典中删除给定键所对应的键值对 | O(1)
dictRelease | 释放给定字典，以及字典中包含的所有键值对 | O(N)，N为字典包含的键值对数量


### 跳跃表
skiplist，是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。  
跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点。  
在大部分情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树要来得更为简单，所以有不少程序都使用跳跃表来代替平衡树。

如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员（member）是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。

Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构。

fruit-price有序集合的130个数据都保存在一个跳跃表里，其中每个跳跃表节点（node）都保存了一款水果的名称和价钱，所有水果按价钱从低到高在跳跃表里面排序。  
例如，跳跃表的第一个元素的成员为"banana"，它的分值为5。
```
redis> ZCARD fruit-price
(integer)130
redis> ZRANGE fruit-price 0 2 WITHSCORES
1)"banana"
2)"5"
3)"cherry"
4)"6.5"
5)"apple"
6)"8"
```

**跳跃表**的结构图示：  
![skiplist](../images/redis/2024-02-18_skiplist.png ':size=50%')

### 整数集合


### 压缩列表





## 对象