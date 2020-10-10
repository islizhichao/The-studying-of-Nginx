## 一、内存池

内存池是Nginx中较为 重要的一个知识点，首先其一次性向系统申请大块内存，内部进行切割分配使用，最后再一次性归还系统，这种方式减少了系统调用的次数，而且很好地避免内存碎片和内存泄漏。

### 1.1 结构定义

```c++
typedef struct ngx_pool_s        ngx_pool_t;
struct ngx_pool_s {
    ngx_pool_data_t       d;//节点数据    // 包含 pool 的数据区指针的结构体 pool->d.last ~ pool->d.end 中的内存区便是可用数据区。
    size_t                max;//当前内存节点可以申请的最大内存空间 // 一次最多从pool中开辟的最大空间
    //每次从pool中分配内存的时候都是从curren开始遍历pool节点获取内存的
    ngx_pool_t           *current;//内存池中可以申请内存的第一个节点      pool 当前正在使用的pool的指针 current 永远指向此pool的开始地址。current的意思是当前的pool地址

/*
pool 中的 chain 指向一个 ngx_chain_t 数据，其值是由宏 ngx_free_chain 进行赋予的，指向之前用完了的，
可以释放的ngx_chain_t数据。由函数ngx_alloc_chain_link进行使用。
*/
    ngx_chain_t          *chain;// pool 当前可用的 ngx_chain_t 数据，注意：由 ngx_free_chain 赋值   ngx_alloc_chain_link
    ngx_pool_large_t     *large;//节点中大内存块指针   // pool 中指向大数据快的指针（大数据快是指 size > max 的数据块）
    ngx_pool_cleanup_t   *cleanup;// pool 中指向 ngx_pool_cleanup_t 数据块的指针 //cleanup在ngx_pool_cleanup_add赋值
    ngx_log_t            *log; // pool 中指向 ngx_log_t 的指针，用于写日志的  ngx_event_accept会赋值
};

```

### 1.2 操作函数

主要为内存的分配与释放

```c++
void* ngx_palloc(ngx_pool_t *pool, size_t size);	//使用内存对齐，速度较快，但有浪费，因为要对齐
void* ngx_pnalloc(ngx_pool_t *pool, size_t size);	//没有使用内存对齐
void* ngx_pcalloc(ngx_pool_t *pool, size_t size);	//内部调用，ngx_palloc，并且把内存卡清零，多数情况下使用

//释放
ngx_int_t ngx_free(ngx_pool_t *pool, void *p);
```

### 1.3 清理机制

```c++
typedef void (*ngx_pool_cleanup_pt)(void *data);

typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

/*
ngx_pool_cleanup_t与ngx_http_cleanup_pt是不同的，ngx_pool_cleanup_t仅在所用的内存池销毁时才会被调用来清理资源，它何时释放资源将视所使用的内存池而定，而ngx_http_cleanup_pt是在ngx_http_request_t结构体释放时被调用来释放资源的。


如果我们需要添加自己的回调函数，则需要调用ngx_pool_cleanup_add来得到一个ngx_pool_cleanup_t，然后设置handler为我们的清理函数，
并设置data为我们要清理的数据。这样在ngx_destroy_pool中会循环调用handler清理数据；

比如：我们可以将一个开打的文件描述符作为资源挂载到内存池上，同时提供一个关闭文件描述的函数注册到handler上，那么内存池在释放
的时候，就会调用我们提供的关闭文件函数来处理文件描述符资源了
*/
//内存池pool中清理数据的用的，见ngx_pool_s  ngx_destroy_pool
struct ngx_pool_cleanup_s { //这个是添加到ngx_pool_s中的cleanup上的，见ngx_pool_cleanup_add
    ngx_pool_cleanup_pt   handler;// 当前 cleanup 数据的回调函数  ngx_destroy_pool中执行    例如清理文件句柄ngx_pool_cleanup_file等
    void                 *data;// 内存的真正地址     回调时，将此数据传入回调函数；  ngx_pool_cleanup_add中开辟空间
    
    ngx_pool_cleanup_t   *next;// 指向下一块 cleanup 内存的指针
};
```

### 1.4 字符串

为了正确、高效地处理字符串，Nginx设计了ngx_str_t结构，但它并不是一个传统意义上的字符串，而是一个“内存块引用”；

```c
typedef struct {
    size_t len;
    u_char* data;
}ngx_str_t;
```



