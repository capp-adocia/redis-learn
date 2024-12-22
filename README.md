## Redis�汾6--Դ�����

### 1. Redis�ı��������ݽṹ
#### 1.1 ����
|  ��������   | ˵��  | ����  | ʹ�õ����ݽṹ  |
|  ----  | ----  | ----  | ----  |
|OBJ_STRING| �ַ���| OBJ_ENCODING_INT  | long long��long |
|           |       | OBJ_ENCODING_ENBSTR  |    string |
|           |       | OBJ_ENCODING_RAW  |    string |
|OBJ_LIST | �б� | OBJ_ENCODING_QUICKLIST  | quicklist |
|OBJ_SET | ���� | OBJ_ENCODING_HT  | dict |
| |   | OBJ_ENCODING_INT  | intset |
|OBJ_ZSET | ���򼯺� | OBJ_ENCODING_ZIPLIST  | dict |
| |   | OBJ_ENCODING_SKIPLIST | skiplist |
|OBJ_HASH | ��ϣ�� | OBJ_ENCODING_HT  | dict |
|  |   | OBJ_ENCODING_ZIPLIST  | ziplist |
|OBJ_STREAM | ��Ϣ�� | OBJ_ENCODING_STREAM  | rax |
| OBJ_Module | Module�Զ�������  | OBJ_ENCODING_RAW  | Module�Զ��� |

#### 1.2 �ַ���
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

- �ַ����� Redis �����������������֮һ�������Դ洢�������͵����ݣ������ı������֡����������ݵȡ�
- Redis �е� �ַ�����String�� �Ƕ����ư�ȫ�ģ����԰����κ���ʽ�����ݣ����磺�ı��������������������������ݵȣ����ַ����� Redis ����õ���������֮һ��֧�ֺܶ�����������á���ȡ���޸ġ�ɾ���ȡ�
#### 1.3 �б�
##### 1.3.1 ziplist
- Ziplist ��ҪĿ����Ϊ�˽�ʡ�ڴ棬�������ڴ洢����С����ʱ������һ��ѹ�����ݽṹ��ͨ�����յķ�ʽ�洢���ݣ�������ڴ�ʹ��Ч�ʡ�
- Ziplist ��һ�����յġ���˳�����е��ֽ����飬���д洢�Ŷ��СԪ�ء�Ziplist ������ �����ڴ�� �ķ�ʽ�������Ԫ�ش洢��ͬһ�ڴ����򣬴Ӷ��������ڴ����Ƭ�����������Ŀ�����Ż��洢�ͷ���Ч�ʣ������ǵ�������С�ұȽϾ���ʱ��Ziplist ���������ݽṹ�����ϣ�����ӽ�ʡ�ڴ档
- Ziplist ��һ��ѹ�����ݽṹ������һ���̶���ͷ����header���Ͷ����Ŀ��entry����ɡ�ÿ����Ŀ���Դ洢һ��ֵ����ÿ����Ŀ�Ĵ�С�ǿɱ�ġ�Ziplist ͨ�����·�ʽ�Ż��洢��
  ```
  1. ��ͷ�����ݴ洢��Ziplist û��ÿ��Ԫ��һ�������ָ���ͷ�������ǽ�Ԫ��ѹ���洢Ϊһ�����յ��ֽ�����
  2. �ֶ�ѹ����Ziplist �е������ǰ�˳��洢�ģ���������ѹ���ͼ����ڴ濪����
  3. СԪ���Ż���Ziplist �ر��ʺϴ洢С�ġ��򵥵�Ԫ�ء����ڴ�Ԫ�أ����糤�ַ��������������Ziplist �����ã���Ϊ����û��Ϊ��Ԫ���ṩ�㹻������ԡ�
  ```
- һ�� Ziplist �������¼������֣�
```
  1. ͷ����Header�������ڱ�ʶ Ziplist �ĳ��ȡ���С��Ŀ�ĳ��ȵ���Ϣ��
  2. ��Ŀ��Entry�����洢���������Ԫ�ء�ÿ����Ŀ��һ���ֽڴ���ɣ�������Ԫ�صĳ�����Ϣ��ֵ��
  3. ������ǣ�Ziplist ������ı�ǽ�������ʾ���ݽṹ�Ľ�����
```
##### 1.3.1 quicklist
- �ṹ��
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
- Quicklist �� Redis �� LIST ���������У�ʹ�õ�һ�ִ洢��ʽ���������� Redis 3.2 �汾֮����ȡ����ԭ�ȵ�˫������� ziplist �Ļ��ʵ�֡�Quicklist ��Ҫͨ�������ڴ����Ĳ���߲���Ч�ʣ�ʹ�ö��б��������͵Ĳ���������롢ɾ���ȣ����Ӹ�Ч��
- ���Quicklist ���ɶ�� ziplist��ѹ���б��� ˫������ڵ㣨ͨ���� quicklistNode����ɵĸ������ݽṹ��ÿ�� quicklistNode �洢һ�� ziplist����ÿ�� ziplist �ڲ��洢�б�Ԫ�ء���һ���б�Ĵ�С����ʱ��Quicklist �ᶯ̬�������Ĵ洢��ʽ��ȷ�����ڴ�������ϴﵽƽ�⡣
- ����ԭ��Quicklist �� Redis �б��Ԫ�ػ���Ϊ�����С��ѹ���б�ziplist������Щѹ���б���֯��˫������Ľڵ��С�
#### 1.4 ɢ��
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

// key value �Ĺ�ϣ��Ͱ
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
- Redis�Ĺ�ϣ�����ݽṹ����SipHash���㷨�����㷨�ܹ���Ч��ֹHash����ײ�����������в������ܡ�
- ���ݣ�����redis�ǵ��̵߳ģ����������ht[0]������������Ǩ�Ƶ�ht[1]����ô�ᵼ���̳߳�������������redis�����ݻ�����ÿ�β�������ʱ������ִ��һ�����ݵ���������
- ����һ��Ὣsize����Ϊ2��n���ݣ������ڼ�������ʱ�����Բ�ȡλ���㣬���Ч�ʡ�

#### 1.5 ����
- Redis�ļ�������set��zset��ɡ�
##### 1.5.1 ���򼯺� set
- ���һ�������ж���������������˷��ڴ棬Ϊ��Redis�������intset�����������⡣
- �ڴ��Ż���intset �ڴ洢����������ʱ����ͨ�� Set ���ͣ����ڹ�ϣ�����ӽ�ʡ�ڴ档
- ���޵��������ͣ�intset �����ڴ洢�������͵����ݣ���ֻ������ Set ���͡�
- ��̬������intset ����ݼ����������Ĵ�С��̬������洢��ʽ���Դﵽ���ŵ��ڴ������ʡ�
- �洢��ʽ��intset �Ĵ洢��ʽ�Ǹ��ݼ����������Ĵ�С��ѡ����ʵ��ڴ��ʽ��������˵��intset ��������±�׼����ʹ���������ݸ�ʽ��
```
    1. ��������е��������� fit�����䣩Ϊ 8-bit �������� -128 �� 127 ֮�䣩�����ʹ�� 8 λ�Ĵ洢��ʽ��
    2. ��������е����������� fit Ϊ 16-bit �������� -32768 �� 32767 ֮�䣩�����ʹ�� 16 λ�洢��ʽ��
    3. ��������е��������� fit Ϊ 32-bit �������� -2147483648 �� 2147483647 ֮�䣩����ʹ�� 32 λ�洢��ʽ��
```
- ��������:
```cpp
// Ϊ�˷�ֹ�洢���ֶ��˷ѿռ䣬intsetʹ�ò�ͬ�ı������洢����
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
##### 1.5.2 ���򼯺� zset
- skiplist���������ɶ��������ɵģ�ÿһ�㶼����һ��������ٵĽڵ㣬��ÿ���ڵ����ͬʱ�����ڶ������ϡ���ײ������������е�Ԫ�أ��ϸ߲�ε��������������Ԫ�ء�ÿһ��Ľڵ�ͨ�� ��Ծָ�� ���ӣ��Ӷ�ʵ����Ծ���ҵ�Ч����
- �ŵ㣺Skiplist ���ŵ�
```
    1. ʵ�ּ򵥣����ƽ������skiplist ��ʵ�ָ�Ϊ�򵥣�����Ҫ���ӵ�ƽ�����ת������
    2. �����ԣ�ͨ�����ʾ�����Σ�������ĳЩ��������µ�����ƿ����
```
- ���ݽṹ�������£�
```cpp
/* ZSETs use a specialized version of Skiplists */
/* �����������֮�ŵ��ڴ����----���� */

// ����ڵ�
typedef struct zskiplistNode {
    sds ele; // Ԫ��
    double score; // �������
    struct zskiplistNode *backward; // ָ��ǰһ���ڵ��ָ��
    struct zskiplistLevel {
        struct zskiplistNode *forward; // ָ����һ���ڵ��ָ��
        unsigned long span; // ��¼�ӵ�ǰ�ڵ㵽��һ���ڵ�Ŀ�ȣ����統ǰ�ڵ��ǵ�2���������һ���ڵ��ǵ�4��������Ϊ2
    } level[]; // �ṹ�����飬ÿ��һ��Ԫ�أ���¼��ǰ�ڵ�ָ����һ�ڵ��ָ��Ϳ��
} zskiplistNode;
// ����
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length; // �ڵ㳤��
    int level; // ������
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
