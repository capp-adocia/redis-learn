## Redis版本6--源码分析

### 1. Redis的编码与数据结构
#### 1.1 编码
|  数据类型   | 说明  | 编码  | 使用的数据结构  |
|  ----  | ----  | ----  | ----  |
|OBJ_STRING| 字符串| OBJ_ENCODING_INT  | long long、long |
|           |       | OBJ_ENCODING_ENBSTR  |    string |
|           |       | OBJ_ENCODING_RAW  |    string |
|OBJ_LIST | 列表 | OBJ_ENCODING_QUICKLIST  | quicklist |
|OBJ_SET | 集合 | OBJ_ENCODING_HT  | dict |
| |   | OBJ_ENCODING_INT  | intset |
|OBJ_ZSET | 有序集合 | OBJ_ENCODING_ZIPLIST  | dict |
| |   | OBJ_ENCODING_SKIPLIST | skiplist |
|OBJ_HASH | 哈希表 | OBJ_ENCODING_HT  | dict |
|  |   | OBJ_ENCODING_ZIPLIST  | ziplist |
|OBJ_STREAM | 消息流 | OBJ_ENCODING_STREAM  | rax |
| OBJ_Module | Module自定义类型  | OBJ_ENCODING_RAW  | Module自定义 |

#### 1.2 字符串
```cpp
typedef char *sds;
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
......
```

- 字符串是 Redis 中最基础的数据类型之一，它可以存储各种类型的数据，包括文本、数字、二进制数据等。
- Redis 中的 字符串（String） 是二进制安全的，可以包含任何形式的数据（例如：文本、整数、浮点数、二进制数据等）。字符串是 Redis 中最常用的数据类型之一，支持很多操作，如设置、获取、修改、删除等。
#### 1.3 列表
##### 1.3.1 ziplist
- Ziplist 主要目的是为了节省内存，尤其是在存储大量小数据时。它是一种压缩数据结构，通过紧凑的方式存储数据，以提高内存使用效率。
- Ziplist 是一个紧凑的、按顺序排列的字节数组，其中存储着多个小元素。Ziplist 采用了 连续内存块 的方式，将多个元素存储在同一内存区域，从而减少了内存的碎片化。它的设计目的是优化存储和访问效率，尤其是当数据量小且比较均匀时，Ziplist 比其他数据结构（如哈希表）更加节省内存。
- Ziplist 是一种压缩数据结构，它由一个固定的头部（header）和多个条目（entry）组成。每个条目可以存储一个值，且每个条目的大小是可变的。Ziplist 通过以下方式优化存储：
  ```
  1. 无头部数据存储：Ziplist 没有每个元素一个额外的指针或头部，而是将元素压缩存储为一个紧凑的字节流。
  2. 字段压缩：Ziplist 中的数据是按顺序存储的，这样便于压缩和减少内存开销。
  3. 小元素优化：Ziplist 特别适合存储小的、简单的元素。对于大元素（例如长字符串或大整数），Ziplist 不适用，因为它并没有为大元素提供足够的灵活性。
  ```
- 一个 Ziplist 包含以下几个部分：
```
  1. 头部（Header）：用于标识 Ziplist 的长度、最小条目的长度等信息。
  2. 条目（Entry）：存储具体的数据元素。每个条目由一个字节串组成，包含了元素的长度信息和值。
  3. 结束标记：Ziplist 以特殊的标记结束，表示数据结构的结束。
```
##### 1.3.1 quicklist
- 结构：
```cpp
typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; 
    int direction;
} quicklistIter;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        
    unsigned long len;          
    int fill : QL_FILL_BITS;              
    unsigned int compress : QL_COMP_BITS; 
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;          
    unsigned int count : 16;   
    unsigned int encoding : 2;  
    unsigned int container : 2;  
    unsigned int recompress : 1; 
    unsigned int attempted_compress : 1; 
    unsigned int extra : 10; 
} quicklistNode;

typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;
```
- Quicklist 是 Redis 在 LIST 数据类型中，使用的一种存储方式，尤其是在 Redis 3.2 版本之后，逐步取代了原先的双向链表和 ziplist 的混合实现。Quicklist 主要通过减少内存消耗并提高操作效率，使得对列表数据类型的操作（如插入、删除等）更加高效。
- 概念：Quicklist 是由多个 ziplist（压缩列表）和 双向链表节点（通常是 quicklistNode）组成的复合数据结构。每个 quicklistNode 存储一个 ziplist，而每个 ziplist 内部存储列表元素。当一个列表的大小增加时，Quicklist 会动态调整它的存储方式，确保在内存和性能上达到平衡。
- 工作原理：Quicklist 将 Redis 列表的元素划分为多个较小的压缩列表（ziplist）。这些压缩列表被组织在双向链表的节点中。
#### 1.4 散列
```cpp
typedef struct dict {
    dictType* type;
    void* privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

// key value 的哈希表桶
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;
```
- Redis的哈希表数据结构采用SipHash等算法，该算法能够有效防止Hash表碰撞攻击，并且有不错性能。
- 扩容：由于redis是单线程的，如果单纯将ht[0]里面所有数据迁移到ht[1]，那么会导致线程长期阻塞，所以redis的扩容机制是每次操作数据时，都会执行一次扩容单步操作。
- 扩容一般会将size调整为2的n次幂，这样在计算索引时，可以采取位运算，提高效率。

#### 1.5 集合
- Redis的集合是由set和zset组成。
##### 1.5.1 无序集合 set
- 如果一个集合中都是整数，这过于浪费内存，为此Redis考虑设计intset来解决这个问题。
- 内存优化：intset 在存储纯整数集合时比普通的 Set 类型（基于哈希表）更加节省内存。
- 有限的数据类型：intset 仅限于存储整数类型的数据，且只适用于 Set 类型。
- 动态调整：intset 会根据集合中整数的大小动态调整其存储格式，以达到最优的内存利用率。
- 存储格式：intset 的存储格式是根据集合中整数的大小来选择合适的内存格式。具体来说，intset 会根据以下标准决定使用哪种数据格式：
```
    1. 如果集合中的整数都能 fit（适配）为 8-bit 整数（即 -128 到 127 之间），则会使用 8 位的存储格式。
    2. 如果集合中的整数都可以 fit 为 16-bit 整数（即 -32768 到 32767 之间），则会使用 16 位存储格式。
    3. 如果集合中的整数都能 fit 为 32-bit 整数（即 -2147483648 到 2147483647 之间），则使用 32 位存储格式。
```
- 定义如下:
```cpp
// 为了防止存储数字而浪费空间，intset使用不同的编码来存储数字
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
##### 1.5.2 有序集合 zset
- skiplist：跳表是由多层链表组成的，每一层都比上一层包含更少的节点，且每个节点可以同时出现在多个层次上。最底层的链表包含所有的元素，较高层次的链表则包含部分元素。每一层的节点通过 跳跃指针 连接，从而实现跳跃查找的效果。
- 优点：Skiplist 的优点
```
    1. 实现简单：相比平衡树，skiplist 的实现更为简单，不需要复杂的平衡和旋转操作。
    2. 概率性：通过概率决定层次，避免了某些极端情况下的性能瓶颈。
```
- 数据结构定义如下：
```cpp
/* ZSETs use a specialized version of Skiplists */
/* 集数组和链表之优点于大成者----跳表 */

// 跳表节点
typedef struct zskiplistNode {
    sds ele; // 元素
    double score; // 排序分数
    struct zskiplistNode *backward; // 指向前一个节点的指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 指向下一个节点的指针
        unsigned long span; // 记录从当前节点到下一个节点的跨度，比如当前节点是第2个，如果下一个节点是第4个，则跨度为2
    } level[]; // 结构体数组，每层一个元素，记录当前节点指向下一节点的指针和跨度
} zskiplistNode;
// 跳表
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length; // 节点长度
    int level; // 最大层数
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
