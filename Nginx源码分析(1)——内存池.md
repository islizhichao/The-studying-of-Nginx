# Nginx源码分析(1)——内存池

Nginx能够具有如此高的性能，除了与自身模块架构设计有关，还与其专门设计的数据结构（链表、动态数组、字符串等）密不可分，那么为了阅读Nginx的源码，探索Nginx实现的秘密，首先来探索Nginx万物开始的地方——内存池。

内存池常常是大型软件中常见的一种使用技术，在C++ STL中的内存分配器、tcmalloc库中都可以看到它的身影。

其主要原理为：一次性向系统申请大块内存，内部进行分割使用，最后一次性释放，归还系统。

主要作用为：

- 减少系统调用次数：程序如果频繁调用内存分配函数，每次都需要陷入内核，这些都是较耗时的操作
- 减少内存碎片与泄露：统一申请、统一归还，可以减少内存泄漏的情况发生；其次可以申请后产生的内存碎片带来的利用率不足问题

所以，让我们带着这些问题，看看Nginx内存池是如何设计并解决这些问题的。



## 内存池结构定义

### 内存池头部结构

Nginx中的内存池头部结构是`ngx_pool_t`，结构体定义如下

```c++
typedef ngx_pool_s ngx_pool_t;
//该结构维护整个内存池的头部信息
//Nginx使用内存池时总是只申请,不释放,使用完毕后直接destroy整个内存池
struct ngx_pool_s {
    ngx_pool_data_t       d;       //数据块
    size_t                max;     //数据块大小，即小块内存的最大值
    ngx_pool_t           *current; //保存当前内存值
    ngx_chain_t          *chain;   //可以挂一个chain结构
    ngx_pool_large_t     *large;   //分配大块内存用，即超过max的内存请求
    ngx_pool_cleanup_t   *cleanup; //挂载一些内存池释放的时候，同时释放的资源
    ngx_log_t            *log;	   //日志信息，（暂且不讨论）
};
```

![](images\Snipaste_2021-04-06_10-39-04.png)

ngx_pool_t代表了Nginx中的内存池概念，Nginx会为每个TCP/HTTP请求、动态数组等分配一个独立的内存池，在整个过程中，统一管理内存的分配、释放，尽量避免使用malloc或new来动态分配内存，从而提高性能。



### 内存池数据结构

```c++
//该结构用来维护内存池的数据块，供用户分配之用
typedef struct {
    u_char               *last;  //当前内存分配结束位置，即下一段可分配内存的起始位置
    u_char               *end;   //内存池结束位置
    ngx_pool_t           *next;  //链接到下一个内存池
    ngx_uint_t            failed;//统计该内存池不能满足分配请求的次数
} ngx_pool_data_t;
```

![](images\Snipaste_2021-04-06_10-54-50.png)

 

### 内存池大数据结构

```c++
typedef struct ngx_pool_large_s  ngx_pool_large_t;
//大内存结构
struct ngx_pool_large_s {
    ngx_pool_large_t     *next; //下一个大块内存
    void                 *alloc;//nginx分配的大块内存空间
};
```

![](images\Snipaste_2021-04-06_10-59-48.png)

### 内存池中的清理数据结构

```c++
typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;	//回调函数指针
    void                 *data;		//要清理的数据块。回调时，把这个数据传入回调函数
    ngx_pool_cleanup_t   *next;		//指向下一个回调函数结构体
};
```

### 内存池总体数据结构

![](images\Snipaste_2021-04-06_11-21-52.png)



## 内存池基本操作



```c++
//使用malloc分配内存空间
void *ngx_alloc(size_t size, ngx_log_t *log);

//使用malloc分配内存空间，并且将空间内容初始化为0
void *ngx_calloc(size_t size, ngx_log_t *log);

//通过ngx_create_pool可以创建一个内存池
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);

//销毁内存池
void ngx_destroy_pool(ngx_pool_t *pool);

//重置内存池
void ngx_reset_pool(ngx_pool_t *pool);

void *ngx_palloc(ngx_pool_t *pool, size_t size);    //通过ngx_palloc可以从内存池中分配指定大小的内存. palloc取得的内存是对齐的
void *ngx_pnalloc(ngx_pool_t *pool, size_t size);   //pnalloc取得的内存是不对齐的
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);   //pcalloc直接调用palloc分配好内存，然后进行一次0初始化操作
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment); //在分配size大小的内存，并按照alignment对齐，然后挂到large字段下
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);		//内存清除


ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
void ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd);
void ngx_pool_cleanup_file(void *data);
void ngx_pool_delete_file(void *data);
```

接下来对相关函数进行介绍，主要对Unix下的相关函数：

### ngx_alloc

该函数只是对malloc函数进行简单的封装

```c++
void *
ngx_alloc(size_t size, ngx_log_t *log)
{
    void  *p;

    p = malloc(size);
    if (p == NULL) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "malloc(%uz) failed", size);
    }

    ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, log, 0, "malloc: %p:%uz", p, size);

    return p;
}
```

### ngx_calloc

使用malloc分配内存空间，并且将空间内容初始化为0

```c++
/* 该函数从系统内存分配内存，而不是内存池，另外该函数还会对内存进行初始化，这就是两个不同点*/
void *
ngx_calloc(size_t size, ngx_log_t *log)
{
    void  *p;

    p = ngx_alloc(size, log);

    if (p) {
        ngx_memzero(p, size);
    }

    return p;
}
```

### ngx_create_pool

```c++
//通过ngx_create_pool可以创建一个内存池
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);  // 分配内存函数，uinx,windows分开走
    if (p == NULL) {
        return NULL;
    }

    p->d.last = (u_char *) p + sizeof(ngx_pool_t); //初始指向 ngx_pool_t 结构体后面
    p->d.end = (u_char *) p + size; //整个结构的结尾后面
    p->d.next = NULL;
    p->d.failed = 0;

	//sizeof(ngx_pool_t)用来存储自身
    size = size - sizeof(ngx_pool_t);
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL; //最大不超过 NGX_MAX_ALLOC_FROM_POOL,也就是getpagesize()-1 大小

    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```

其中`ngx_memalign`封装了`posix_memalign`函数。

```c++
#if (NGX_HAVE_POSIX_MEMALIGN)

void *
ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
{
    void  *p;
    int    err;

    err = posix_memalign(&p, alignment, size);

    if (err) {
        ngx_log_error(NGX_LOG_EMERG, log, err,
                      "posix_memalign(%uz, %uz) failed", alignment, size);
        p = NULL;
    }

    ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "posix_memalign: %p:%uz @%uz", p, size, alignment);

    return p;
}

#elif (NGX_HAVE_MEMALIGN)

void *
ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
{
    void  *p;

    p = memalign(alignment, size);
    if (p == NULL) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "memalign(%uz, %uz) failed", alignment, size);
    }

    ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "memalign: %p:%uz @%uz", p, size, alignment);

    return p;
}

#endif
```

在大多数情况下，编译器和C库透明地帮你处理对齐问题。**POSIX** 标明了通过malloc( ), calloc( ), 和 realloc( ) 返回的地址对于任何的C类型来说都是对齐的。在Linux中，这些函数返回的地址在32位系统是以8字节为边界对齐，在64位系统是以16字节为边界对齐的。有时候，对于更大的边界，例如页面，程序员需要动态的对齐。虽然动机是多种多样的，但最常见的是直接块I/O的缓存的对齐或者其它的软件对硬件的交 互，因此，**POSIX** 1003.1d提供一个叫做**posix**_**memalign**( )的函数：

```c++
/* one or the other -- either suffices */
\#define _XOPEN_SOURCE 600
\#define _GNU_SOURCE
\#include <stdlib.h>
int **posix**_**memalign** (void **memptr,
          size_t alignment,
          size_t size);

\* See http://perens.com/FreeSoftware/ElectricFence/ and http://valgrind.org, respectively.
```

调用**posix**_**memalign**( )成功时会返回size字节的动态内存，并且这块内存的地址是alignment的倍数。参数alignment必须是2的幂，还是void指针的大小的倍数。返回的内存块的地址放在了memptr里面，函数返回值是0.



### ngx_pnalloc

```c++
/* 原理和ngx_alloc一样，只是没有考虑数据对齐*/
void *
ngx_pnalloc(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    ngx_pool_t  *p;

    if (size <= pool->max) {

        p = pool->current;

        do {
            m = p->d.last;

            if ((size_t) (p->d.end - m) >= size) {
                p->d.last = m + size;

                return m;
            }

            p = p->d.next;

        } while (p);

        return ngx_palloc_block(pool, size);
    }

    return ngx_palloc_large(pool, size);
}
```



### ngx_palloc_block

```c++
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new, *current;

    psize = (size_t) (pool->d.end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;

    current = pool->current;

    for (p = current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            current = p->d.next;
        }
    }

    p->d.next = new;

    pool->current = current ? current : new;

    return m;
}
```

### ngx_palloc_large

```c++
//控制大块内存的申请
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;

    p = ngx_alloc(size, pool->log); // 在大数据块链表中申请调用了ngx_alloc
    if (p == NULL) {
        return NULL;
    }

    n = 0;

    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    large = ngx_palloc(pool, sizeof(ngx_pool_large_t));
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

### ngx_palloc

```c++
//ngx_palloc相对ngx_pnalloc，其会将申请的内存大小向上扩增到NGX_ALIGNMENT的倍数，以方便内存对齐，减少内存访问次数。
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    ngx_pool_t  *p;

    if (size <= pool->max) {

        p = pool->current;

        do {
            m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT); // 对齐内存指针，加快存取速度

            if ((size_t) (p->d.end - m) >= size) {
                p->d.last = m + size;

                return m;
            }

            p = p->d.next;

        } while (p);

        return ngx_palloc_block(pool, size);
    }
	//size过大，需要在大数据块内存链表上申请内存空间
    return ngx_palloc_large(pool, size);
}
```

### ngx_pcalloc

```c++
//ngx_pcalloc其只是ngx_palloc的一个封装，将申请到的内存全部初始化为0。
void *
ngx_pcalloc(ngx_pool_t *pool, size_t size)
{
    void *p;

    p = ngx_palloc(pool, size);
    if (p) {
        ngx_memzero(p, size);  //初始化
    }

    return p;
}
```



### ngx_pmemalign

```c++
void *
ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment)
{
    void              *p;
    ngx_pool_large_t  *large;

    p = ngx_memalign(alignment, size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    large = ngx_palloc(pool, sizeof(ngx_pool_large_t));
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

### ngx_pfree

```c++
#define ngx_free          free
//控制大块内存的释放。注意，这个函数只会释放大内存，不会释放其对应的头部结构，遗留下来的头部结构会做下一次申请大内存之用
ngx_int_t
ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p", l->alloc);
            ngx_free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
}
```



### ngx_reset_pool

```c++
#define ngx_free          free
//重置，将内存池恢复到初始分配的状态
void
ngx_reset_pool(ngx_pool_t *pool)
{
    ngx_pool_t        *p;
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc); //释放大块内存
        }
    }

    pool->large = NULL;

    for (p = pool; p; p = p->d.next) {
        p->d.last = (u_char *) p + sizeof(ngx_pool_t); //小块内存结尾指针指向刚分配的位置，其中的数据将被覆盖
    }
}
```



### ngx_destroy_pool

```c++
//销毁内存池
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;
	// 遍历节点上的各个节点
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);  //释放节点占用的内存
        }
    }
	//对大块数据内存的清理
    for (l = pool->large; l; l = l->next) {

        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);

        if (l->alloc) {
            ngx_free(l->alloc); // 直接ngx_free，宏，本质是free
        }
    }

#if (NGX_DEBUG)

    /*
     * we could allocate the pool->log from this pool
     * so we cannot use this log while free()ing the pool
     */

    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: %p, unused: %uz", p, p->d.end - p->d.last);

        if (n == NULL) {
            break;
        }
    }

#endif

    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);

        if (n == NULL) {
            break;
        }
    }
}
```

### ngx_pool_clean_add

```c++
// 用于向内存回收链表中添加节点数据
ngx_pool_cleanup_t *
ngx_pool_cleanup_add(ngx_pool_t *p, size_t size)
{
    ngx_pool_cleanup_t  *c;

    c = ngx_palloc(p, sizeof(ngx_pool_cleanup_t));
    if (c == NULL) {
        return NULL;
    }

    if (size) {
        c->data = ngx_palloc(p, size); // 申请存放目标数据的空间
        if (c->data == NULL) {
            return NULL;
        }

    } else {
        c->data = NULL;
    }

    c->handler = NULL;
    c->next = p->cleanup;

    p->cleanup = c;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, p->log, 0, "add cleanup: %p", c);

    return c;
}
```



### ngx_pool_run_cleanup_file

```c++
void
ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd)
{
    ngx_pool_cleanup_t       *c;
    ngx_pool_cleanup_file_t  *cf;

    for (c = p->cleanup; c; c = c->next) {
        if (c->handler == ngx_pool_cleanup_file) {

            cf = c->data;

            if (cf->fd == fd) {
                c->handler(cf);
                c->handler = NULL;
                return;
            }
        }
    }
}
```



### ngx_pool_cleanup_file

```c++
//关闭文件句柄
void
ngx_pool_cleanup_file(void *data)
{
    ngx_pool_cleanup_file_t  *c = data;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, c->log, 0, "file cleanup: fd:%d",
                   c->fd);

    if (ngx_close_file(c->fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", c->name);
    }
}
```



### ngx_pool_delete_file

```c++
void
ngx_pool_delete_file(void *data)
{
    ngx_pool_cleanup_file_t  *c = data;

    ngx_err_t  err;

    ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, c->log, 0, "file cleanup: fd:%d %s",
                   c->fd, c->name);

    if (ngx_delete_file(c->name) == NGX_FILE_ERROR) {
        err = ngx_errno;

        if (err != NGX_ENOENT) {
            ngx_log_error(NGX_LOG_CRIT, c->log, err,
                          ngx_delete_file_n " \"%s\" failed", c->name);
        }
    }

    if (ngx_close_file(c->fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      ngx_close_file_n " \"%s\" faild", c->name);
    }
}
```

### ngx_pool_cleanup_file_t

```c++
/*ngx_pool_cleanup_t中的*data成员通常指向ngx_pool_cleanup_file_t结构体*/
typedef struct {
    ngx_fd_t              fd;	//文件句柄
    u_char               *name;	//文件名称
    ngx_log_t            *log;	//日志对象
} ngx_pool_cleanup_file_t;
```

