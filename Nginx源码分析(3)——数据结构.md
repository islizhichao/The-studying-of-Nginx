# Nginx源码分析(3)——数据结构

## 双向链表

ngx_queue_t双向链表是Nginx提供的轻量级链表容器，与大学数据结构学过的链表不同，它其实可以算做一种**结构数据**，其与Linux系统中设计的链表思想类似，提供统一的链表接口，用户只需要将自己的数据放入链表节点即可，而实际操作中只操作其接口；其次，双向链表不管理节点中元素的内存分配，ngx_queue_t只把这些已经分配好内存的元素用双向链表连接起来。ngx_queue_t为我们提供了很多方法，接下来让我们一探究竟。

### 双向链表的数据结构

ngx_queue_t容器的实现使用了一个数据结构ngx_queue_t，如下所示：

```c
typedef struct ngx_queue_s  ngx_queue_t;

//参考：
//http://blog.csdn.net/livelylittlefish/article/details/6607324
struct ngx_queue_s {
    ngx_queue_t  *prev;   //前一个
    ngx_queue_t  *next;   //下一个
};
```

### 双向链表的使用方法

| 函数                    | 功能                   |
| ----------------------- | :--------------------- |
| ngx_queue_init()        | 初始化队列             |
| ngx_queue_empty()       | 判断队列是否为空       |
| ngx_queue_insert_head() | 在头节点之后插入新节点 |
| ngx_queue_insert_tail() | 在尾节点之后插入新节点 |
| ngx_queue_head()        | 头节点                 |
| ngx_queue_last()        | 尾节点                 |
| ngx_queue_sentinel()    | 头部标记节点           |
| ngx_queue_remove()      | 移除节点x              |
| ngx_queue_split()       | 分隔链表               |
| ngx_queue_add()         | 链接队列               |
| ngx_queue_middle()      | 队列的中间节点         |
| ngx_queue_sort()        | 队列排序               |

其中能够使ngx_queue_t统一使用的函数为：

| 函数             | 功能               |
| ---------------- | ------------------ |
| ngx_queue_next() | 下一个节点         |
| ngx_queue_prev() | 前一个结点         |
| ngx_queue_data() | 获取队列中节点数据 |

其中较为重要的就是`ngx_queue_data(q,type,link)`

```c
//获取队列中节点数据， q是队列中的节点，type队列类型，link是队列类型中ngx_queue_t的元素名
#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))
```



### 双向链表的具体示例

```c++
#include "ngx_queue.h"
#include <cstdio>
#include <cstddef>

typedef struct {
    u_char* str;
    ngx_queue_t qEle;
    int num;
}TestNode;

ngx_int_t compTestNode(const ngx_queue_t* a, const ngx_queue_t* b)
{
    TestNode* aNode = ngx_queue_data(a,TestNode,qEle);
    TestNode* bNode = ngx_queue_data(b,TestNode,qEle);
    return aNode->num > bNode->num;
}

int main()
{
    ngx_queue_t queueContainer;
    ngx_queue_init(&queueContainer);
    TestNode node[5];
    for(int i = 0; i < 5; ++i)
    {
        node[i].num = i;
    }

    ngx_queue_insert_tail(&queueContainer,&node[0].qEle);
    ngx_queue_insert_head(&queueContainer,&node[1].qEle);
    ngx_queue_insert_tail(&queueContainer,&node[2].qEle);
    ngx_queue_insert_after(&queueContainer,&node[3].qEle);
    ngx_queue_insert_tail(&queueContainer,&node[4].qEle);

    ngx_queue_t* q;
    for(q = ngx_queue_head(&queueContainer); q != ngx_queue_sentinel(&queueContainer); q = ngx_queue_next(q))
    {
        TestNode* eleNode = ngx_queue_data(q,TestNode,qEle);
        printf("%d\n",eleNode->num);
    }
    
    ngx_queue_sort(&queueContainer,compTestNode);

    for(q = ngx_queue_head(&queueContainer); q != ngx_queue_sentinel(&queueContainer); q = ngx_queue_next(q))
    {
        TestNode* eleNode = ngx_queue_data(q,TestNode,qEle);
        printf("%d\n",eleNode->num);
    }
    return 0;
}
```

对照示例代码，我们逐行分析。

```c
//创建一个双向链表容器并初始化
ngx_queue_t queueContainer;
ngx_queue_init(&queueContainer);
```

![](images\Snipaste_2021-04-12_21-19-25.png)

```c
/* x插入h节点前面 */
#define ngx_queue_insert_tail(h, x)                                           \
    (x)->prev = (h)->prev;                                                    \
    (x)->prev->next = x;                                                      \
    (x)->next = h;                                                            \
    (h)->prev = x

ngx_queue_insert_tail(&queueContainer,&node[0].qEle);
```

![](images\Snipaste_2021-04-12_21-25-26.png)

![](images\Snipaste_2021-04-12_21-32-46.png)





```c
#define ngx_queue_insert_head(h, x)                                           \
    (x)->next = (h)->next;                                                    \
    (x)->next->prev = x;                                                      \
    (x)->prev = h;                                                            \
    (h)->next = x
ngx_queue_insert_head(&queueContainer,&node[1].qEle);
```

![](images\Snipaste_2021-04-12_21-37-59.png)

```c
#define ngx_queue_insert_after   ngx_queue_insert_head
ngx_queue_insert_after(&queueContainer,&node[3].qEle);
```



```c
/* the stable insertion sort 
ngx_queue_sort(queue, cmd)使用插入排序算法对queue队列进行排序，完成后在next方向上为升序，prev方向为降序。 
*/
//对locationqueue进行排序，即ngx_queue_sort，排序的规则可以读一下ngx_http_cmp_locations函数。
//该函数功能就是按照cmp比较，然后对珍格格queue进行从新排序，ngx_queue使用例子可以参考http://blog.chinaunix.net/uid-26284395-id-3134435.html
//可以看到，这里采用的是插入排序算法，时间复杂度为O(n)，整个代码非常简洁。
void
ngx_queue_sort(ngx_queue_t *queue,
    ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *))
{
    ngx_queue_t  *q, *prev, *next;

    q = ngx_queue_head(queue);

    if (q == ngx_queue_last(queue)) {
        return;
    }

    for (q = ngx_queue_next(q); q != ngx_queue_sentinel(queue); q = next) {

        prev = ngx_queue_prev(q);
        next = ngx_queue_next(q);

        ngx_queue_remove(q); //这三行是依次序从前到后找出队列中需要排序的元素

        do {
            if (cmp(prev, q) <= 0) {//从大到下排序
                break;
            }

            prev = ngx_queue_prev(prev);

        } while (prev != ngx_queue_sentinel(queue)); //查找这个元素需要插入到前面依据拍好序的队列的那个地方

        ngx_queue_insert_after(prev, q);
    }
}
```



## 动态数组

C++语言中STL模板库中的vector就像`ngx_array_t`一样是一个动态数组，当我们申请一个动态数组的时候，它允许我们可不确定元素个数，并在分配过程中自动扩容。

### 动态数组的数据结构

```c++
// 动态数组
struct ngx_array_s {
    // elts指向数组的首地址
    void        *elts; 
    // nelts是数组中已经使用的元素个数
    ngx_uint_t   nelts; 
    // 每个数组元素占用的内存大小
    size_t       size;  
    // 当前数组中能够容纳元素个数的总大小
    ngx_uint_t   nalloc; 
    // 内存池对象
    ngx_pool_t  *pool;  
};
```

<img src="images\Snipaste_2021-04-07_15-10-53.png" style="zoom:67%;" />



### 动态数组的使用方法

|                            方法名                            |                           参数含义                           | 执行意义                                                     |
| :----------------------------------------------------------: | :----------------------------------------------------------: | ------------------------------------------------------------ |
|  ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size)  | p是内存池，n是初始分配元素的最大个数，size是每个元素所占用的内存大小 | 创建一个动态数组，并预分配n个大小为size的内存空间            |
| ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size) | a是一个动态数组结构体的指针，p是内存池，n是初始分配元素的最大个数，size是每个元素所占用的内存大小 | 初始化一个已经存在的动态数组，并预分配n个大小为size的内存空间 |
|              ngx_array_destroy(ngx_array_t *a)               |                 a是一个动态数组结构体的指针                  | 销毁已经分配的数组元素空间和ngx_array_t动态数组对象          |
|                ngx_array_push(ngx_array_t *a)                |                 a是一个动态数组结构体的指针                  | 向当前动态数组添加一个元素，并返回这个新添加元素的地址       |
|        ngx_array_push_n(ngx_array_t *a, ngx_uint_t n)        |        a是一个动态数组结构体的指针，n是添加元素的个数        | 向当前动态数组添加n个元素，并返回这批元素的第一个元素地址    |

### 动态数组的具体示例

```c
#include "ngx_array.h"

typedef struct {
    u_char* str;
    int num;
}TestNode;

int main()
{
    ngx_pool_t* g_pool = ngx_create_pool(4096);
    ngx_array_t* dynamicArray = ngx_array_create(g_pool,1,sizeof(TestNode));
    TestNode* a = (TestNode*)ngx_array_push(dynamicArray);
    a->num = 1;
    a = (TestNode*)ngx_array_push(dynamicArray);
    a->num = 2;

    TestNode* b = (TestNode*)ngx_array_push_n(dynamicArray, 3);
    b->num = 3;
    (b + 1)->num = 4;
    (b + 2)->num = 5;

    TestNode* nodeArray = (TestNode*)dynamicArray->elts;
    ngx_uint_t arraySeq = 0;
    for(; arraySeq < dynamicArray->nelts; arraySeq++)
    {
        a = nodeArray + arraySeq;
        printf("%d\n",a->num);
    }
    ngx_array_destroy(dynamicArray);
    return 0;
}
```

动态数组扩容情况分为以下两种情况：

- 如果当前内存池中剩余的空间大于或等于本次需要新增的空间，那么扩容将只扩充新增的空间。对于ngx_array_push方法来说，就是扩充一个元素，对于ngx_array_push_n来说，就是扩充n个元素
- 如果当前内存池中剩余的空间小于本次需要新增的空间，那么对ngx_array_push方法来说，会将原来动态数组的容量扩充一倍；而对于ngx_array_push_n来说，情况要复杂一些，如果参数n小于原先动态数组的容量，将扩容一倍；如果参数n大于原先动态数组的容量，则会重现分配2*n的空间，将原先数组拷贝过来。

## 单向链表



## 红黑树



## 基数树

## 散列表

散列表也称为哈希表，是典型的以空间换事件的数据结构体，在合理的条件下，对hash表中元素的检索、插入速度可以达到O(1)时间复杂度，所以这种高效的方式非常适合频繁读取、插入、删除元素。但与常见的哈希表不同，Ngixn中哈希表有如下特点：

- 支持通配符。允许以*作为通配符，包括前置通配符，如\*.test.com，或者后置通配符，如www.test.\*。
- 与常见解决哈希表冲突的方式不同，Nginx中哈希表使用开放寻址法来解决碰撞问题（我猜这样是为了节约空间）。

### 散列表的数据结构

散列表中的元素，使用`ngx_hash_elt_t`结构体来存储，

```c
/*
ngx_hash_elt_t中uchar name[1]的设计，如果在name很短的情况下，name和 ushort 的字节对齐可能只用占到一个字节，这样就比放一
个uchar* 的指针少占用一个字节，可以看到ngx是真的在内存上考虑，节省每一分内存来提高并发。
*/
//hash表中元素ngx_hash_elt_t 预添加哈希散列元素结构 ngx_hash_key_t
typedef struct {//实际一个ngx_hash_elt_t暂用的空间为NGX_HASH_ELT_SIZE，比sizeof(ngx_hash_elt_t)大，因为需要存储具体的字符串name
    //如果value=NULL,表示是该具体hash桶bucket[i]中的最后一个ngx_hash_elt_t成员
    void             *value;//value，即某个key对应的值，即<key,value>中的value   
    u_short           len;//name长度  key的长度    
    //每次移动test[]的时候，都是移动NGX_HASH_ELT_SIZE(&names[n])，里面有给name预留name字符串长度空间，见ngx_hash_init
    u_char            name[1]; //元素关键字的首地址

//ngx_hash_elt_t是ngx_hash_t->buckets[i]桶中的具体成员
} ngx_hash_elt_t; //hash元素结构   //ngx_hash_init中names数组存入hash桶前，其结构是ngx_hash_key_t形式，在往hash桶里面存数据的时候，会把ngx_hash_key_t里面的成员拷贝到ngx_hash_elt_t中相应成员 
```



完全匹配散列表的数据结构，使用`ngx_hash_t`结构体表示

```c
//在创建hash桶的时候赋值，见ngx_hash_init
typedef struct { //hash桶遍历可以参考ngx_hash_find  
    ngx_hash_elt_t  **buckets; //hash桶(有size个桶)    指向各个桶的头部指针，也就是bucket[]数组，bucket[I]又指向每个桶中的第一个ngx_hash_elt_t成员，见ngx_hash_init
    ngx_uint_t        size;//hash桶个数，注意是桶的个数，不是每个桶中的成员个数，见ngx_hash_init
} ngx_hash_t;
```

ngx_hash_t基本散列表的结构示意图如下所示：

![](images\Snipaste_2021-04-14_14-40-12.png)

Nginx中散列表提供了`ngx_hash_find`方法用于查询元素

```c
void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len);
//其中hash为散列表结构体的指针
//key为根据计算散列方法得到的散列关键字
//name和len则表示关键字的地址和长度
//结果为返回散列表中关键字与name、len完全相同的槽中；如果没有就返回NULL
```

此外Nginx还提供的两种基本的散列方法

```c
#define ngx_hash(key, c)   ((ngx_uint_t) key * 31 + c)
//使用BKDR算法将任意长度的字符串映射为整型
ngx_uint_t ngx_hash_key(u_char *data, size_t len);
//将字符串全小写后，再使用BKDR算法映射为整型
ngx_uint_t ngx_hash_key_lc(u_char *data, size_t len);
```

使用函数指针`ngx_hash_key_pt`指向散列方法：

```c
typedef ngx_uint_t (*ngx_hash_key_pt) (u_char *data, size_t len);
```







为了支持通配符，Nginx还设计了一种支持通配符的散列表`ngx_hash_combined_t`，可以支持简单的前置通配符或后置通配符。

所谓支持通配符的散列表，就是把预散列元素的关键字中的通配符去除后，作为关键字加入散列表中（==目前仅支持前置或后置通配符==）比如：对于关键之"www.test.*"，直接简历一个专用的后置通配符散列表，存储元素的关键字为"www.test"，这样当检索www.test.cn是否匹配www.test.\*时，可以使用Nginx提供的ngx_hash_find_wc_tail检索，把查询的www.test.cn转化为www.test字符串再开始查询。

同理对于关键字"*.test.com"这样的情况，也建立了一个专用的前置通配符散列表，存储元素的关键字为com.test.。这样，当我们检索smtp.test.com是否匹配时，使用Nginx提供的ngx_hash_find_wc_head检索，ngx_hash_find_wc_head方法会把要查询的smtp.test.com转化为com.test.字符串再开始查询

```c
/*
nginx为了处理带有通配符的域名的匹配问题，实现了ngx_hash_wildcard_t这样的hash表。他可以支持两种类型的带有通配符的域名。一种是通配符在前的，
例如：“*.abc.com”，也可以省略掉星号，直接写成”.abc.com”。这样的key，可以匹配www.abc.com，qqq.www.abc.com之类的。另外一种是通配符在末
尾的，例如：“mail.xxx.*”，请特别注意通配符在末尾的不像位于开始的通配符可以被省略掉。这样的通配符，可以匹配mail.xxx.com、mail.xxx.com.cn、
mail.xxx.net之类的域名。

有一点必须说明，就是一个ngx_hash_wildcard_t类型的hash表只能包含通配符在前的key或者是通配符在后的key。不能同时包含两种类型的通配符
的key。ngx_hash_wildcard_t类型变量的构建是通过函数ngx_hash_wildcard_init完成的，而查询是通过函数ngx_hash_find_wc_head或者
ngx_hash_find_wc_tail来做的。ngx_hash_find_wc_head是查询包含通配符在前的key的hash表的，而ngx_hash_find_wc_tail是查询包含通配符在后的key的hash表的。
*/
typedef struct {
    ngx_hash_t        hash;
    void             *value;
} ngx_hash_wildcard_t;
```

Nginx对于server_name主机名通配符的支持规则：

- 首先选择精准匹配的server_name，比如www.test.com匹配上www.test.com
- 其次选择通配符在前面的server_name，比如www.test.com匹配*.test.com
- 最后选择通配符在后面的server_name，比如www.test.com匹配www.test.*

按此规则，Ngixn实现的`ngx_hash_combined_t`通配符散列表数据结构：

```c
/*
Nginx对于server- name主机名通配符的支持规则。
    首先，选择所有字符串完全匹配的server name，如www.testweb.com。
    其次，选择通配符在前面的server name，如*.testweb.com。
    再次，选择通配符在后面的server name，如www.testweb.*。

ngx_hash_combined_t是由3个哈希表组成，一个普通hash表hash，一个包含前向通配符的hash表wc_head和一个包含后向通配符的hash表 wc_tail。
*/ //ngx_http_virtual_names_t中包含该结构
typedef struct { //这里面的hash信息是ngx_http_server_names中存储到hash表中的server_name及其所在server{}上下文ctx,server_name为key，上下文ctx为value
    //该hash中存放的是ngx_hash_keys_arrays_t->keys[]数组中的成员
    ngx_hash_t            hash; //普通hash，完全匹配
    //该wc_head hash中存储的是ngx_hash_keys_arrays_t->dns_wc_head数组中的成员
    ngx_hash_wildcard_t  *wc_head; //前置通配符hash
    //该wc_head hash中存储的是ngx_hash_keys_arrays_t->dns_wc_tail数组中的成员
    ngx_hash_wildcard_t  *wc_tail; //后置通配符hash
} ngx_hash_combined_t;
```

使用ngx_hash_find_combined方法查询通配符散列表元素

```c
void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key,
    u_char *name, size_t len);
```



### 散列表的使用方法

初始化散列表方法：

```c
//hinit时散列表初始化结构体的指针
//names是数组的首地址，存储着预添加到散列表中的元素
//nelts是names数组的元素数目
ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
    ngx_uint_t nelts);
ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
    ngx_uint_t nelts);
```



```c
typedef struct {//如何使用以及使用方法可以参考ngx_http_referer_merge_conf
    ngx_hash_t       *hash;//指向待初始化的hash结构
    ngx_hash_key_pt   key; //hash函数指针   

    /*  max_size和bucket_size的意义
    max_size表示最多分配max_size个桶，每个桶中的元素(ngx_hash_elt_t)个数 * NGX_HASH_ELT_SIZE(&names[n])不能超过bucket_size大小
    实际ngx_hash_init处理的时候并不是直接用max_size个桶，而是从x=1到max_size去试，只要ngx_hash_init参数中的names[]数组数据能全部hash
    到这x个桶中，并且满足条件:每个桶中的元素(ngx_hash_elt_t)个数 * NGX_HASH_ELT_SIZE(&names[n])不超过bucket_size大小,则说明用x
    个桶就够用了，然后直接使用x个桶存储。 见ngx_hash_init
     */
    
    ngx_uint_t        max_size;//最多需要这么多个桶，实际上桶的个数，是通过计算得到的，参考ngx_hash_init
    //分配到该具体桶中的成员个数 * NGX_HASH_ELT_SIZE(&names[n])要小于bucket_size，表示该桶中最多存储这么多个ngx_hash_elt_t成员
    //生效地方见ngx_hash_init中的:if (test[key] > (u_short) bucket_size) { }
    ngx_uint_t        bucket_size; //表示每个hash桶中(hash->buckets[i->成员[i]])对应的成员所有ngx_hash_elt_t成员暂用空间和的最大值，就是每个桶暂用的所有空间最大值，通过这个值计算需要多少个桶
    char             *name; //该hash结构的名字(仅在错误日志中使用,用于调试打印等)   
    ngx_pool_t       *pool;  //该hash结构从pool指向的内存池中分配   
    ngx_pool_t       *temp_pool; //分配临时数据空间的内存池   
} ngx_hash_init_t; //hash初始化结构  
```



```c
//hash表中元素ngx_hash_elt_t 预添加哈希散列元素结构 ngx_hash_key_t
typedef struct { //赋值参考ngx_hash_add_key
    ngx_str_t         key; //key，为nginx的字符串结构   
    ngx_uint_t        key_hash; //由该key计算出的hash值(通过hash函数如ngx_hash_key_lc())  
    void             *value;  //该key对应的值，组成一个键-值对<key,value>    在通配符hash中也可能指向下一个通配符hash,见ngx_hash_wildcard_init
} ngx_hash_key_t; //ngx_hash_init中names数组存入hash桶前，其结构是ngx_hash_key_t形式，在往hash桶里面存数据的时候，会把ngx_hash_key_t里面的成员拷贝到ngx_hash_elt_t中相应成员
```



```c
/*
注：keys，dns_wc_head，dns_wc_tail，三个数组中存放的元素时ngx_hash_key_t类型的，而keys_hash,dns_wc_head_hash，dns_wc_tail_hash，
三个二维数组中存放的元素是ngx_str_t类型的。
ngx_hash_keys_array_init就是为上述结构分配空间。
*/ //该结构只是用来把完全匹配   前置匹配  后置匹配通过该结构体全部存储在该结构体对应的hash和数组中
typedef struct { //该结构一般用来存储域名信息
    //散列中槽总数  如果是大hash桶方式，则hsize=NGX_HASH_LARGE_HSIZE,小hash桶方式，hsize=107,见ngx_hash_keys_array_init
    ngx_uint_t        hsize; 

    ngx_pool_t       *pool;//内存池，用于分配永久性的内存
    ngx_pool_t       *temp_pool; //临时内存池，下面的临时动态数组都是由临时内存池分配

    //下面这几个实际上是hash通的各个桶的头部指针，每个hash有ha->hsize个桶头部指针，在ngx_hash_add_key的时候头部指针指向每个桶中具体的成员列表
    //下面的这些可以参考ngx_hash_add_key
     /*
    keys_hash这是个二维数组，第一个维度代表的是bucket的编号，那么keys_hash[i]中存放的是所有的key算出来的hash值对hsize取模以后的值为i的key。
    假设有3个key,分别是key1,key2和key3假设hash值算出来以后对hsize取模的值都是i，那么这三个key的值就顺序存放在keys_hash[i][0],
    keys_hash[i][1], keys_hash[i][2]。该值在调用的过程中用来保存和检测是否有冲突的key值，也就是是否有重复。
    */ //完全匹配keys数组只存放key字符串，节点类型为ngx_str_t，keys_hash hash表存放key对应的key-value中，hash表中数据节点类型为ngx_hash_key_t

   /* 
    hash桶keys_hash  dns_wc_head_hash   dns_wc_tail_hash头部指针信息初始化在ngx_hash_keys_array_init，其中的具体
    桶keys_hash[i] dns_wc_head_hash[i]  dns_wc_tail_hash[i]中的数据类型为ngx_str_t，每个桶的数据成员默认4个，见ngx_hash_add_key，
    桶中存储的数据信息就是ngx_hash_add_key参数中key参数字符串
    
    数组keys[] dns_wc_head[] dns_wc_tail[]中的数据类型为ngx_hash_key_t，见ngx_hash_keys_array_init，
    ngx_hash_key_t中的key和value分别存储ngx_hash_add_key中的key参数和value参数
    */

    /*

    赋值见ngx_hash_add_key
    
    原始key                  存放到hash桶(keys_hash或dns_wc_head_hash                 存放到数组中(keys或dns_wc_head或
                                    或dns_wc_tail_hash)                                     dns_wc_tail)
                                    
 www.example.com                 www.example.com(存入keys_hash)                        www.example.com (存入keys数组成员ngx_hash_key_t对应的key中)
  .example.com             example.com(存到keys_hash，同时存入dns_wc_tail_hash)        com.example  (存入dns_wc_head数组成员ngx_hash_key_t对应的key中)
 www.example.*                     www.example. (存入dns_wc_tail_hash)                 www.example  (存入dns_wc_tail数组成员ngx_hash_key_t对应的key中)
 *.example.com                     example.com  (存入dns_wc_head_hash)                 com.example. (存入dns_wc_head数组成员ngx_hash_key_t对应的key中)

    //这里面的ngx_hash_keys_arrays_t桶keys_hash  dns_wc_head_hash  dns_wc_tail_hash，对应的具体桶中的空间是数组，数组大小是提前进行数组初始化的时候设置好的
    ngx_hash_t->buckets[]中的具体桶中的成员是根据实际成员个数创建的空间
    */
    ngx_array_t       keys;//数组成员ngx_hash_key_t
    ngx_array_t      *keys_hash;//keys_hash[i]对应hash桶头部指针，，具体桶中成员类型ngx_str_t

    ngx_array_t       dns_wc_head; //数组成员ngx_hash_key_t
    ngx_array_t      *dns_wc_head_hash;//dns_wc_head_hash[i]对应hash桶头部指针，具体桶中成员类型ngx_str_t

    ngx_array_t       dns_wc_tail; //数组成员ngx_hash_key_t
    ngx_array_t      *dns_wc_tail_hash; //dns_wc_tail_hash[i]对应hash桶头部指针，具体桶中成员类型ngx_str_t
} ngx_hash_keys_arrays_t; //ngx_http_referer_conf_t中的keys成员
```

### 散列表的具体示例

```c
#include  "ngx_hash.h"

typedef struct 
{
    ngx_str_t servername;
    ngx_int_t seq;
}TestWildcardHashNode;

int main()
{
    ngx_hash_init_t hash;
    ngx_hash_keys_arrays_t ha;
    ngx_hash_combined_t combinedHash;
    ngx_memzero(&ha,sizeof(ngx_hash_keys_arrays_t));
    ngx_pool_t* pool = ngx_create_pool(1024);
    ha.temp_pool = ngx_create_pool(16384);
    if(ha.temp_pool == NULL)
        return NGX_ERROR;
    ha.pool = pool;
    if(ngx_hash_keys_array_init(&ha,NGX_HASH_LARGE) != NGX_OK)
        return NGX_ERROR;

    TestWildcardHashNode testHashNode[3];
    testHashNode[0].servername.len = ngx_strlen("*.test.com");
    testHashNode[0].servername.data = ngx_palloc(pool,ngx_strlen("*.test.com"));
    ngx_memcpy(testHashNode[0].servername.data,"*.test.com",ngx_strlen("*.test.com"));

    testHashNode[1].servername.len = ngx_strlen("www.test.*");
    testHashNode[1].servername.data = ngx_palloc(pool,ngx_strlen("www.test.*"));
    ngx_memcpy(testHashNode[1].servername.data,"www.test.*",ngx_strlen("www.test.*"));

    testHashNode[2].servername.len = ngx_strlen("www.test.com");
    testHashNode[2].servername.data = ngx_palloc(pool,ngx_strlen("www.test.com"));
    ngx_memcpy(testHashNode[2].servername.data,"www.test.com",ngx_strlen("www.test.com"));


    for(int i = 0; i < 3; ++i)
    {
        testHashNode[i].seq = i;
        ngx_hash_add_key(&ha,&testHashNode[i].servername,&testHashNode[i],NGX_HASH_WILDCARD_KEY);
    }
    ngx_cacheline_size = 32; //不加入这个会出现段错误
    hash.key = ngx_hash_key_lc;
    hash.max_size = 100;
    hash.bucket_size = 48;
    hash.name = "test_server_name_hash";
    hash.pool = pool;

    if(ha.keys.nelts)
    {
        hash.hash = &combinedHash.hash;
        hash.temp_pool = NULL;
        if(ngx_hash_init(&hash,ha.keys.elts,ha.keys.nelts) != NGX_OK)
            return NGX_ERROR;

    }

    if(ha.dns_wc_head.nelts)
    {
        hash.hash = NULL;
        hash.temp_pool = ha.temp_pool;
        if(ngx_hash_wildcard_init(&hash,ha.dns_wc_head.elts,ha.dns_wc_head.nelts) != NGX_OK)
            return NGX_ERROR;
        combinedHash.wc_head =(ngx_hash_wildcard_t*) hash.hash;
    }

    if(ha.dns_wc_tail.nelts)
    {
        hash.hash = NULL;
        hash.temp_pool = ha.temp_pool;
        if(ngx_hash_wildcard_init(&hash,ha.dns_wc_tail.elts,ha.dns_wc_tail.nelts) != NGX_OK)
            return NGX_ERROR;
        combinedHash.wc_tail = (ngx_hash_wildcard_t *)hash.hash;
    }
    ngx_destroy_pool(ha.temp_pool);
    ngx_str_t findServer;
    findServer.len = ngx_strlen("www.test.org");
    findServer.data = ngx_pcalloc(pool,ngx_strlen("www.test.org"));
    ngx_memcpy(findServer.data,"www.test.org",ngx_strlen("www.test.org"));
    TestWildcardHashNode* findHashNode = ngx_hash_find_combined(&combinedHash,ngx_hash_key_lc( findServer.data,findServer.len),findServer.data,findServer.len);
    printf("%s\n",findHashNode->servername.data);
    
    findServer.len = ngx_strlen("www.test.com");
    findServer.data = ngx_pcalloc(pool,ngx_strlen("www.test.com"));
    ngx_memcpy(findServer.data,"www.test.com",ngx_strlen("www.test.com"));
    findHashNode = ngx_hash_find_combined(&combinedHash,ngx_hash_key_lc( findServer.data,findServer.len),findServer.data,findServer.len);
    printf("%s\n",findHashNode->servername.data);
    findServer.len = ngx_strlen("smtp.test.com");
    findServer.data = ngx_pcalloc(pool,ngx_strlen("smtp.test.com"));
    ngx_memcpy(findServer.data,"smtp.test.com",ngx_strlen("smtp.test.com"));
    findHashNode = ngx_hash_find_combined(&combinedHash,ngx_hash_key_lc( findServer.data,findServer.len),findServer.data,findServer.len);
    printf("%s\n",findHashNode->servername.data);
    

    return 0;
}
```

```c
//初始化ngx_hash_keys_arrays_t结构体，在ha加入成员之前必须先调用这个方法
ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);

//向ha中添加1个元素
ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key,
    void *value, ngx_uint_t flags);
```

