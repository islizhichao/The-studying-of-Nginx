# Nginx高级数据结构

## 动态数组

Nginx借鉴了C++标准容器std::vector，以C语言实现了“泛型的”，可以在运行时随意变更大小的动态数组ngx_array_t，而且由于使用ngx_pool_t分配内存保证没有内存泄漏。

```c
typedef struct { //可以通过ngx_array_create函数创建空间，并初始化各个成员
    void        *elts; //可以是ngx_keyval_t  ngx_str_t  ngx_bufs_t ngx_hash_key_t等
    ngx_uint_t   nelts; //已经使用了多少个
    size_t       size; //每个elts的空间大小，
    ngx_uint_t   nalloc; //最多有多少个elts元素
    ngx_pool_t  *pool; //赋值见ngx_init_cycle，为cycle的时候分配的pool空间
} ngx_array_t;
```

## 单向链表

Nginx的单向链表ngx_list_t设计融合了一些ngx_array_t的特点，在一个节点里存储多个元素。

**结构定义**

```c
typedef struct ngx_list_part_s  ngx_list_part_t; //ngx_list_part_t只描述链表的一个元素

//链表图形化见<<输入理解nginx>> 3.2.3　ngx_list_t数据结构
//内存分配参考ngx_list_push
struct ngx_list_part_s { //ngx_list_part_t只描述链表的一个元素   数据部分总的空间大小为size * nalloc字节
    void             *elts; //指向数组的起始地址。
    ngx_uint_t        nelts; //数组当前已使用了多少容量  表示数组中已经使用了多少个元素。当然，nelts必须小于ngx_list_t 结构体中的nalloc。
    ngx_list_part_t  *next; //下一个链表元素ngx_list_part_t的地址。
};
```

ngx_list_t结构定义了链表，实际上是头节点+元信息。

```c
/* ngx_list_t和ngx_queue_t的却别在于:ngx_list_t需要负责容器内成员节点内存分配，而ngx_queue_t不需要 */
//用法和数组遍历方法可以参考//ngx_http_request_s->headers_in.headers，例如可以参考函数ngx_http_fastcgi_create_request
typedef struct { //ngx_list_t描述整个链表
    ngx_list_part_t  *last; //指向链表的最后一个数组元素。
    ngx_list_part_t   part; //链表的首个数组元素。 part可能指向多个数组，通过part->next来指向当前数组所在的下一个数组的头部
    /*
    链表中的每个ngx_list_part_t元素都是一个数组。因为数组存储的是某种类型的数据结构，且ngx_list_t 是非常灵活的数据结构，所以它不会限制存储
    什么样的数据，只是通过size限制每一个数组元素的占用的空间大小，也就是用户要存储的一个数据所占用的字节数必须小于或等于size。
    */ //size就是数组元素中的每个子元素的大小最大这么大
    size_t            size;  //创建list的时候在ngx_list_create->ngx_list_init中需要制定n和size大小
    ngx_uint_t        nalloc; //链表的数组元素一旦分配后是不可更改的。nalloc表示每个ngx_list_part_t数组的容量，即最多可存储多少个数据。
    ngx_pool_t       *pool; //链表中管理内存分配的内存池对象。用户要存放的数据占用的内存都是由pool分配的，下文中会详细介绍。
} ngx_list_t;
```

## 双向队列

ngx_queue_t是侵入式容器，必须把ngx_queue_t作为元素的一个成员，才可以放入队列。

**结构定义**

```c
typedef struct ngx_queue_s  ngx_queue_t;
/* ngx_list_t和ngx_queue_t的却别在于:ngx_list_t需要负责容器内成员节点内存分配，而ngx_queue_t不需要 */
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

ngx_queue_t的使用方式需要结构体添加它作为成员，例如：

```c
struct Xinfo
{
	int x = 0;
    ngx_queue_t queue;
};
```

我们称这种方式为侵入式，即侵入结构体中。

而Nginx使用了一种巧妙的方法，可以从作为数据成员的ngx_queue_t结构访问到完整的数据节点：

```c
#define ngx_queue_data(q,type,link) (type*)((u_char*)q - offset(type,link)) 
```

其中`q`、`type`、`link`分别为ngx_queue_t指针、节点类型和ngx_queue_t成员名，利用C标准宏offsetof计算成员在结构体里的偏移量，然后再倒着减去偏移量，最后得到真是的地址。

但是这种获取数据的方法只能在C语言中使用，因为C结构的内存分布式平坦的，在C++里要小心使用。



## 红黑树

红黑树是一种自平衡二叉树，不仅可以快速查找，而且插入和删除的效率都很高，常用于构造关联数组（例如C++标准库中的set和map）。

```c
typedef struct ngx_rbtree_node_s  ngx_rbtree_node_t;

/*
ngx_rbtree_node_t是红黑树实现中必须用到的数据结构，一般我们把它放到结构体中的
第1个成员中，这样方便把自定义的结构体强制转换成ngx_rbtree_node_t类型。例如：
typedef struct  {
    //一般都将ngx_rbtree_node_t节点结构体放在自走义数据类型的第1位，以方便类型的强制转换 
    ngx_rbtree_node_t node;
    ngx_uint_t num;
) TestRBTreeNode;
    如果这里希望容器中元素的数据类型是TestRBTreeNode，那么只需要在第1个成员中
放上ngx_rbtree_node_t类型的node即可。在调用图7-7中ngx_rbtree_t容器所提供的方法
时，需要的参数都是ngx_rbtree_node_t类型，这时将TestRBTreeNode类型的指针强制转换
成ngx_rbtree_node_t即可。
*/
struct ngx_rbtree_node_s {
    /* key成员是每个红黑树节点的关键字，它必须是整型。红黑树的排序主要依据key成员 */
    ngx_rbtree_key_t       key; //无符号整型的关键字  参考ngx_http_file_cache_exists  其实就是ngx_http_cache_t->key的前4字节
    ngx_rbtree_node_t     *left;
    ngx_rbtree_node_t     *right;
    ngx_rbtree_node_t     *parent;
    u_char                 color; //节点的颜色，0表示黑色，l表示红色
    u_char                 data;//仅1个字节的节点数据。由于表示的空间太小，所以一般很少使用
};
```

与ngx_queue_t一样，ngx_rbtree_node_t也要作为结构体的一个成员，以侵入的方式来使用，例如保证字符串的红黑树节点是

```c
typedef struct
{
    ngx_rbtree_node_t node;
    ngx_str_t str;
}ngx_str_node_t;
```

**树结构定义**

```c
typedef void (*ngx_rbtree_insert_pt) (ngx_rbtree_node_t *root,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel); //node插入root的方法  执行地方在ngx_rbtree_insert

struct ngx_rbtree_s {
    ngx_rbtree_node_t     *root;      //指向树的根节点。注意，根节点也是数据元素
    ngx_rbtree_node_t     *sentinel;  //指向NIL峭兵节点  哨兵节点是所有最下层的叶子节点都指向一个NULL空节点，图形化参考:http://blog.csdn.net/xzongyuan/article/details/22389185
    ngx_rbtree_insert_pt   insert;    //表示红黑树添加元素的函数指针，它决定在添加新节点时的行为究竟是替换还是新增
};
```



## 缓冲区

Nginx实现了ngx_buf_t和ngx_chain_t结构，专门用于描述缓冲区。

**结构定义**

```c
typedef void *            ngx_buf_tag_t;
/*
缓冲区ngx_buf_t是Nginx处理大数据的关键数据结构，它既应用于内存数据也应用于磁盘数据
*/
typedef struct ngx_buf_s  ngx_buf_t; //从内存池中分配ngx_buf_t空间，通过ngx_chain_s和pool关联，并初始化其中的各个变量初值函数为ngx_create_temp_buf
/*
ngx_buf_t是一种基本数据结构，本质上它提供的仅仅是一些指针成员和标志位。对于HTTP模块来说，需要注意HTTP框架、事件框架是如何设置
和使用pos、last等指针以及如何处理这些标志位的，上述说明只是最常见的用法。（如果我们自定义一个ngx_buf_t结构体，不应当受限于上
述用法，而应该根据业务需求自行定义。例如用一个ngx_buf_t缓冲区转发上下游TCP流时，pos会指向将要发送到下游的TCP流起
始地址，而last会指向预备接收上游TCP流的缓冲区起始地址。）
*/
/*
实际上，Nginx还封装了一个生成ngx_buf_t的简便方法，它完全等价于上面的6行语句，如下所示。
ngx_buf_t *b = ngx_create_temp_buf(r->pool, 128);

分配完内存后，可以向这段内存写入数据。当写完数据后，要让b->last指针指向数据的末尾，如果b->last与b->pos相等，那么HTTP框架是不会发送一个字节的包体的。

最后，把上面的ngx_buf_t *b用ngx_chain_t传给ngx_http_output_filter方法就可以发送HTTP响应的包体内容了。例如：
ngx_chain_t out;
out.buf = b;
out.next = NULL;
return ngx_http_output_filter(r, &out);
*/ //参考http://blog.chinaunix.net/uid-26335251-id-3483044.html    
struct ngx_buf_s { //可以参考ngx_create_temp_buf         函数空间在ngx_create_temp_buf创建，让指针指向这些空间
/*pos通常是用来告诉使用者本次应该从pos这个位置开始处理内存中的数据，这样设置是因为同一个ngx_buf_t可能被多次反复处理。
当然，pos的含义是由使用它的模块定义的*/
    //它的pos成员和last成员指向的地址之间的内存就是接收到的还未解析的字符流
    u_char          *pos; //pos指针指向从内存池里分配的内存。 pos为已扫描的内存端中，还未解析的内存的尾部，
    u_char          *last;/*last通常表示有效的内容到此为止，注意，pos与last之间的内存是希望nginx处理的内容*/

    /* 处理文件时，file_pos与file_last的含义与处理内存时的pos与last相同，file_pos表示将要处理的文件位置，file_last表示截止的文件位置 */
    off_t            file_pos;  //可以结合ngx_output_chain_copy_buf阅读更好理解    输出数据的打印可以在ngx_http_write_filter中查看调试信息
    //也就是实际存到临时文件中的字节数,见ngx_event_pipe_write_chain_to_temp_file
    off_t            file_last; //写入文件内容的最尾处的长度赋值见ngx_http_read_client_request_body     输出数据的打印可以在ngx_http_write_filter中查看调试信息

    //如果ngx_buf_t缓冲区用于内存，那么start指向这段内存的起始地址
    u_char          *start;         /* start of buffer */ //创建空间见ngx_http_upstream_process_header
    u_char          *end;           /* end of buffer */ //与start成员对应，指向缓冲区内存的末尾
    //ngx_http_request_body_length_filter中赋值为ngx_http_read_client_request_body
    //实际上是一个void*类型的指针，使用者可以关联任意的对象上去，只要对使用者有意义。
    ngx_buf_tag_t    tag;/*表示当前缓冲区的类型，例如由哪个模块使用就指向这个模块ngx_module_t变量的地址*/
    ngx_file_t      *file;//引用的文件  用于存储接收到所有包体后，把包体内容写入到file文件中，赋值见ngx_http_read_client_request_body
    
 /*当前缓冲区的影子缓冲区，该成员很少用到，仅仅在使用缓冲区转发上游服务器的响应时才使用了shadow成员，这是因为Nginx太节
 约内存了，分配一块内存并使用ngx_buf_t表示接收到的上游服务器响应后，在向下游客户端转发时可能会把这块内存存储到文件中，也
 可能直接向下游发送，此时Nginx绝不会重新复制一份内存用于新的目的，而是再次建立一个ngx_buf_t结构体指向原内存，这样多个
 ngx_buf_t结构体指向了同一块内存，它们之间的关系就通过shadow成员来引用。这种设计过于复杂，通常不建议使用

 当这个buf完整copy了另外一个buf的所有字段的时候，那么这两个buf指向的实际上是同一块内存，或者是同一个文件的同一部分，此
 时这两个buf的shadow字段都是指向对方的。那么对于这样的两个buf，在释放的时候，就需要使用者特别小心，具体是由哪里释放，要
 提前考虑好，如果造成资源的多次释放，可能会造成程序崩溃！
 */
    ngx_buf_t       *shadow; //参考ngx_http_fastcgi_input_filter


    /* the buf's content could be changed */
    //为1时表示该buf所包含的内容是在一个用户创建的内存块中，并且可以被在filter处理的过程中进行变更，而不会造成问题
    unsigned         temporary:1; //临时内存标志位，为1时表示数据在内存中且这段内存可以修改

    /*
     * the buf's content is in a memory cache or in a read only memory
     * and must not be changed
     */ //为1时表示该buf所包含的内容是在内存中，但是这些内容确不能被进行处理的filter进行变更。
    unsigned         memory:1;//标志位，为1时表示数据在内存中且这段内存不可以被修改

    /* the buf's content is mmap()ed and must not be changed */
    //为1时表示该buf所包含的内容是在内存中, 是通过mmap使用内存映射从文件中映射到内存中的，这些内容确不能被进行处理的filter进行变更。
    unsigned         mmap:1;//标志位，为1时表示这段内存是用mmap系统调用映射过来的，不可以被修改

    //可以回收的。也就是这个buf是可以被释放的。这个字段通常是配合shadow字段一起使用的，对于使用ngx_create_temp_buf 函数创建的buf，
    //并且是另外一个buf的shadow，那么可以使用这个字段来标示这个buf是可以被释放的。
    //置1了表示该buf需要马上发送出去，参考ngx_http_write_filter -> if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
    unsigned         recycled:1; //标志位，为1时表示可回收利用，当该buf被新的buf指针指向的时候，就置1，见ngx_http_upstream_send_response

    /*
    ngx_buf_t有一个标志位in_file，将in_file置为1就表示这次ngx_buf_t缓冲区发送的是文件而不是内存。
调用ngx_http_output_filter后，若Nginx检测到in_file为1，将会从ngx_buf_t缓冲区中的file成员处获取实际的文件。file的类型是ngx_file_t
    */ //为1时表示该buf所包含的内容是在文件中。   输出数据的打印可以在ngx_http_write_filter中查看调试信息
    unsigned         in_file:1;//标志位，为1时表示这段缓冲区处理的是文件而不是内存，说明包体全部存入文件中，需要配置"client_body_in_file_only" on | clean 

    //遇到有flush字段被设置为1的的buf的chain，则该chain的数据即便不是最后结束的数据（last_buf被设置，标志所有要输出的内容都完了），
    //也会进行输出，不会受postpone_output配置的限制，但是会受到发送速率等其他条件的限制。
    ////置1了表示该buf需要马上发送出去，参考ngx_http_write_filter -> if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
    unsigned         flush:1;//标志位，为1时表示需要执行flush操作  标示需要立即发送缓冲的所有数据；
    /*标志位，对于操作这块缓冲区时是否使用同步方式，需谨慎考虑，这可能会阻塞Nginx进程，Nginx中所有操作几乎都是异步的，这是
    它支持高并发的关键。有些框架代码在sync为1时可能会有阻塞的方式进行I/O操作，它的意义视使用它的Nginx模块而定*/
    unsigned         sync:1;
    /*标志位，表示是否是最后一块缓冲区，因为ngx_buf_t可以由ngx_chain_t链表串联起来，因此，当last_buf为1时，表示当前是最后一块待处理的缓冲区*/
    /*
    如果接受包体接收完成，则存储最后一个包体内容的buf的last_buf置1，见ngx_http_request_body_length_filter  
    如果发送包体的时候只有头部，这里会置1，见ngx_http_header_filter
    如果各个模块在发送包体内容的时候，如果送入ngx_http_write_filter函数的in参数chain表中的某个buf为该包体的最后一段内容，则该buf中的last_buf会置1
     */ // ngx_http_send_special中会置1  见ngx_http_write_filter -> if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
    //置1了表示该buf需要马上发送出去，参考ngx_http_write_filter -> if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
    unsigned         last_buf:1;  //数据被以多个chain传递给了过滤器，此字段为1表明这是最后一个chain。
    //在当前的chain里面，此buf是最后一个。特别要注意的是last_in_chain的buf不一定是last_buf，但是last_buf的buf一定是last_in_chain的。这是因为数据会被以多个chain传递给某个filter模块。
    unsigned         last_in_chain:1;//标志位，表示是否是ngx_chain_t中的最后一块缓冲区

    //在创建一个buf的shadow的时候，通常将新创建的一个buf的last_shadow置为1。  
    //参考ngx_http_fastcgi_input_filter
    //当数据被发送出去后，该标志位1的ngx_buf_t最终会添加到free_raw_bufs中
    //该值一般都为1，
    unsigned         last_shadow:1; /*标志位，表示是否是最后一个影子缓冲区，与shadow域配合使用。通常不建议使用它*/
    unsigned         temp_file:1;//标志位，表示当前缓冲区是否属于临时文件

    /* STUB */ int   num;  //是为读取后端服务器包体分配的第几个buf ，见ngx_event_pipe_read_upstream  表示属于链表chain中的第几个buf
};
```

## 数据块链

在处理TCP/HTTP请求时会经常创建多个缓冲区来存放数据，Nginx把缓冲区块简单地组织成一个单向链表，使用ngx_chain_t结构描述。

**结构定义**

```c
buf指向当前的ngx_buf_t缓冲区，next则用来指向下一个ngx_chain_t。如果这是最后一个ngx_chain_t，则需要把next置为NULL。

在向用户发送HTTP 包体时，就要传入ngx_chain_t链表对象，注意，如果是最后一个ngx_chain_t，那么必须将next置为NULL，
否则永远不会发送成功，而且这个请求将一直不会结束（Nginx框架的要求）。
*/
struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

```c
ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool);

/*
pool 中的 chain 指向一个 ngx_chain_t 数据，其值是由宏 ngx_free_chain 进行赋予的，指向之前用完了的，
可以释放的ngx_chain_t数据。由函数ngx_alloc_chain_link进行使用。
*/

#define ngx_free_chain(pool, cl)                                             \
    cl->next = pool->chain;                                                  \
    pool->chain = cl
```

由于ngx_chain_t在Nginx里应用的很频繁，所以Nginx对此进行了优化，在内存池里保存了一个空闲ngx_chain_t链表，分配时从这个链表摘取，释放时再挂上去。

