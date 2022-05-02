# nginx源码分析——字符串数据结构

nginx官方设计并实现了字符串，其源文件在`src\core\ngx_string.h`中，用结构体`ngx_str_t`表示

```c
typedef struct {
    size_t      len;  //字符串的有效长度
    u_char     *data; //字符串的内容，指向字符串的起始位置
} ngx_str_t;
```

为了与C语言中字符串兼容，其设计了如下宏定义：

```c
//通过一个以‘0’结尾的普通字符串str构造一个nginx的字符串。
//鉴于api中采用sizeof操作符计算字符串长度，因此该api的参数必须是一个常量字符串。
#define ngx_string(str)     { sizeof(str) - 1, (u_char *) str }

//声明变量时，初始化字符串为空字符串，符串的长度为0，data为NULL。
#define ngx_null_string     { 0, NULL }

//设置字符串str为text，text必须为常量字符串。
#define ngx_str_set(str, text)                                               \
    (str)->len = sizeof(text) - 1; (str)->data = (u_char *) text

//设置字符串str为空串，长度为0，data为NULL。
#define ngx_str_null(str)   (str)->len = 0; (str)->data = NULL
```

包括如下对nginx字符串操作的函数

```c
//将src的前n个字符转换成小写存放在dst字符串当中
void ngx_strlow(u_char *dst, u_char *src, size_t n);
```



## Nginx基础架构

| 模块名称           | 作用                                     |
| ------------------ | ---------------------------------------- |
| ngx_module_t       | 设计模块的初始化、退出以及对配置项的处理 |
| ngx_conf_module    | 配置模块                                 |
| ngx_core_module    | 核心模块                                 |
| ngx_errlog_module  | 核心模块                                 |
| ngx_events_module  | 核心模块                                 |
| ngx_openssl_module | 核心模块                                 |
| ngx_http_module    | 核心模块                                 |
| ngx_mail_module    | 核心模块                                 |
| ngx_core_t         | 核心结构体                               |



## 事件模块

如何收集、管理、分发事件。

|        模块名称        |                             作用                             |
| :--------------------: | :----------------------------------------------------------: |
|   ngx_events_module    |                          解析配置项                          |
| ngx_event_core_module  |            决定使用哪种驱动机制，以及如何管理事件            |
|   ngx_event_module_t   | 通用接口，允许每个事件模块建立自己的配置项结构体，用于存储感兴趣的配置项在nginx.conf中对应的参数 |
|      ngx_event_t       |                           每个事件                           |
| ngx_handle_read_event  |                  将读事件添加到事件驱动模块                  |
| ngx_handle_write_event |                  将写事件添加到事件驱动模块                  |

### Nginx事件数据结构

#### ngx_event_t

```c++
struct ngx_event_s {
    // 事件相关的对象。通常data都是指向ngx_connection_t连接对象。开启文件异步I/O时，它可能会指向ngx_event_aio_t结构体
    void            *data;

    /*
    标志位，为1时表示事件是可写的。通常情况下，它表示对应的TCP连接目前状态是可写的，也就是连接处于可以发送网络包的状态。
    */
    unsigned         write:1;

    // 标志位，为1时表示为此事件可以建立新的连接。通常情况下，在ngx_cycle_t中的listening动态数组中，每一个监听对象ngx_listening_t
    // 对应的读事件中的accept标志位才会是1
    unsigned         accept:1;

    /* used to detect the stale events in kqueue, rtsig, and epoll */
    /*
    这个标志位用于区分当前事件是否过期，它仅仅是给事件驱动模块使用的，而事件消费模块可不用关心。
    为什么需要这个标志位呢？当开始处理一批事件时，处理前面的事件可能会关闭一些连接，而这些连接有可能影响这批事件中还未处理到的后面的事件。
    这时，可通过instance标志位来避免处理后面的已经过期的事件。
    */
    unsigned         instance:1;

    /*
     * the event was passed or would be passed to a kernel;
     * in aio mode - operation was posted.
     */
     /*
    标志位，为1表示当前事件是活跃的，为0表示事件是不活跃的。
    这个状态对应着事件驱动模块处理方式的不同。例如，在添加事件，删除事件和处理事件时，active标志位的不同都会对应着不同的处理方式。
    在使用事件时，一般不会直接改变active标志位。
     */
    unsigned         active:1;

    /*
    标志位，为1表示禁用事件，仅在kqueue或者rtsig事件驱动模块中有效，而对于epoll事件驱动模块则没有意义。
    */
    unsigned         disabled:1;

    /* the ready event; in aio mode 0 means that no operation can be posted */
    // 标志位，为1表示当前事件已经准备就绪，也就是说，允许这个事件的消费模块处理这个事件。在HTTP框架中，经常会检查事件的ready标志位，
    // 以确定是否可以接收请求或者发送相应
    unsigned         ready:1;

    // 该标志位仅对kqueue,eventport等模块有意义，而对于linux上的epoll事件驱动模块则是无意义的。
    unsigned         oneshot:1;

    /* aio operation is complete */
    // 该标志位用于异步AIO事件的处理
    unsigned         complete:1;

    // 标志位，为1时表示当前处理的字符流已经结束
    unsigned         eof:1;
    // 标志位，为1表示事件在处理过程中出现错误
    unsigned         error:1;

    // 标志位，为1表示这个事件已经超时，用以提示事件的消费模块做超时处理，它与timer_set都用了定时器
    unsigned         timedout:1;
    // 标志位，为1表示这个事件存在于定时器中
    unsigned         timer_set:1;

    // 标志位，delayed为1表示需要延迟处理这个事件，它仅用于限速功能
    unsigned         delayed:1;

    // 标志位目前没有使用
    unsigned         read_discarded:1;

    // 目前没有使用
    unsigned         unexpected_eof:1;

    // 标志位，为1表示延迟建立TCP连接，也就是说，经过TCP三次握手后并不建立连接，而是要等到真正受到数据包后才会建立TCP连接
    unsigned         deferred_accept:1;

    /* the pending eof reported by kqueue or in aio chain operation */
    // 标志位，为1表示等待字符流结束，它只与kqueue和aio事件驱动机制有关
    unsigned         pending_eof:1;

#if !(NGX_THREADS)
    // 标志位，如果为1，表示在处理post事件时，当前事件已经准备就绪
    unsigned         posted_ready:1;
#endif

#if (NGX_WIN32)
    /* setsockopt(SO_UPDATE_ACCEPT_CONTEXT) was successful */
    unsigned         accept_context_updated:1;
#endif

#if (NGX_HAVE_KQUEUE)
    unsigned         kq_vnode:1;

    /* the pending errno reported by kqueue */
    int              kq_errno;
#endif

    /*
     * kqueue only:
     *   accept:     number of sockets that wait to be accepted
     *   read:       bytes to read when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *   write:      available space in buffer when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *
     * iocp: TODO
     *
     * otherwise:
     *   accept:     1 if accept many, 0 otherwise
     */

#if (NGX_HAVE_KQUEUE) || (NGX_HAVE_IOCP)
    int              available;
#else
    // 标志位，在epoll事件驱动机制下表示一次尽可能多建立TCP连接，它与mulit_accept配置项对应
    unsigned         available:1;
#endif

    // 这个事件发生时的处理方法，每个事件消费模块都会重新实现它
    ngx_event_handler_pt  handler;


#if (NGX_HAVE_AIO)

#if (NGX_HAVE_IOCP)
    // Windows系统下的一种事件驱动模型
    ngx_event_ovlp_t ovlp;
#else
    // Linux aio机制中定义的结构体
    struct aiocb     aiocb;
#endif

#endif

    // 由于epoll 事件驱动方式不使用index，所以这里不再说明
    ngx_uint_t       index;

    // 可用于记录error_log日志的ngx_log_t对象
    ngx_log_t       *log;

    // 定时器节点，用于定时器红黑树中
    ngx_rbtree_node_t   timer;

    // 标志位，为1时表示当前事件已经关闭，epoll模块没有使用它
    unsigned         closed:1;

    /* to test on worker exit */
    // 无实际意义
    unsigned         channel:1;
    // 无实际意义
    unsigned         resolver:1;

#if (NGX_THREADS)

    unsigned         locked:1;

    unsigned         posted_ready:1;
    unsigned         posted_timedout:1;
    unsigned         posted_eof:1;

#if (NGX_HAVE_KQUEUE)
    /* the pending errno reported by kqueue */
    int              posted_errno;
#endif

#if (NGX_HAVE_KQUEUE) || (NGX_HAVE_IOCP)
    int              posted_available;
#else
    unsigned         posted_available:1;
#endif

    ngx_atomic_t    *lock;
    ngx_atomic_t    *own_lock;

#endif

    /* the links of the posted queue */
    /*
    post事件将会构成一个队列，再统一处理，这个队列以next和prev作为链表指针，以此构成一个简易的双向链表，
    其中next指向后一个事件的地址，prev指向前一个事件的地址。
    */
    ngx_event_t     *next;
    ngx_event_t    **prev;


#if 0

    /* the threads support */

    /*
     * the event thread context, we store it here
     * if $(CC) does not understand __thread declaration
     * and pthread_getspecific() is too costly
     */

    void            *thr_ctx;

#if (NGX_EVENT_T_PADDING)

    /* event should not cross cache line in SMP */

    uint32_t         padding[NGX_EVENT_T_PADDING];
#endif
#endif
};
```



#### ngx_events_module

```c
static ngx_command_t  ngx_events_commands[] = {

    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,
      0,
      0,
      NULL },

      ngx_null_command
};


static ngx_core_module_t  ngx_events_module_ctx = {
    ngx_string("events"),
    NULL,
    NULL
};


ngx_module_t  ngx_events_module = {
    NGX_MODULE_V1,
    &ngx_events_module_ctx,                /* module context */
    ngx_events_commands,                   /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```



#### ngx_event_core_module

ngx_event_core_module模块是一个事件类型的模块。

```c++
static ngx_str_t  event_core_name = ngx_string("event_core");


static ngx_command_t  ngx_event_core_commands[] = {

    { ngx_string("worker_connections"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_connections,
      0,
      0,
      NULL },

    { ngx_string("connections"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_connections,
      0,
      0,
      NULL },

    { ngx_string("use"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_use,
      0,
      0,
      NULL },

    { ngx_string("multi_accept"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, multi_accept),
      NULL },

    { ngx_string("accept_mutex"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex),
      NULL },

    { ngx_string("accept_mutex_delay"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex_delay),
      NULL },

    { ngx_string("debug_connection"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_debug_connection,
      0,  
      0,
      NULL },

      ngx_null_command
};


ngx_event_module_t  ngx_event_core_module_ctx = {
    &event_core_name,
    ngx_event_create_conf,                 /* create configuration */
    ngx_event_init_conf,                   /* init configuration */

    { NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL }
};


ngx_module_t  ngx_event_core_module = {
    NGX_MODULE_V1,
    &ngx_event_core_module_ctx,            /* module context */
    ngx_event_core_commands,               /* module directives */
    NGX_EVENT_MODULE,                      /* module type */
    NULL,                                  /* init master */
    ngx_event_module_init,                 /* init module */
    ngx_event_process_init,                /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

#### ngx_event_module_t

每一个事件模块都必须实现`ngx_event_module_t`接口

```c++
typedef struct {
    // 事件模块的名称
    ngx_str_t              *name;

    // 在解析配置项前，这个回调方法用于创建存储配置项参数的结构体
    void                 *(*create_conf)(ngx_cycle_t *cycle);
    // 在解析配置项完成后，init_conf方法会被调用，用于综合处理当前事件模块感兴趣的全部配置项。
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);

    // 对于事件驱动机制，每个事件模块需要实现的10个抽象方法
    ngx_event_actions_t     actions;
} ngx_event_module_t;
```

#### ngx_event_conf_t

```c++
typedef struct {
    // 连接池的大小
    ngx_uint_t    connections;
    // 选用的事件模块在所有事件模块中的序号
    ngx_uint_t    use;

    // 标志位，如果为1，则表示在接收到一个新连接事件时，一次性建立尽可能多的连接
    ngx_flag_t    multi_accept;
    //标识位，为1表示启用负载均衡锁
    ngx_flag_t    accept_mutex;

    /*
    负载均衡锁会使有些worker进程在拿不到锁时延迟建立新连接，accept_mutex_delay就是这段延迟时间的长度
    */
    ngx_msec_t    accept_mutex_delay;

    // 所选用事件模块的名字，它与use成员是匹配的
    u_char       *name;

#if (NGX_DEBUG)
    /*
    在 --with-debug 编译模式下，可以仅针对某些客户端建立的连接输出调试级别的日志，而debug_connection数组用于保存这些客户端的地址信息
    */
    ngx_array_t   debug_connection;
#endif
} ngx_event_conf_t;
```

#### ngx_event_actions_t

```c++
typedef struct {
    /*
    添加事件方法，它将负责把1个感兴趣的事件添加到操作系统提供的事件驱动机制（如epoll，kqueue等）中，
    这样，在事件发生之后，将可以在调用下面的process_envets时获取这个事件。
    */
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    /*
    删除事件方法，它将一个已经存在于事件驱动机制中的事件一出，这样以后即使这个事件发生，调用process_events方法时也无法再获取这个事件
    */
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    /*
    启用一个事件，目前事件框架不会调用这个方法，大部分事件驱动模块对于该方法的实现都是与上面的add方法完全一致的
    */
    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    /*
    禁用一个事件，目前事件框架不会调用这个方法，大部分事件驱动模块对于该方法的实现都是与上面的del方法一致
    */
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    /*
    向事件驱动机制中添加一个新的连接，这意味着连接上的读写事件都添加到事件驱动机制中了
    */
    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    // 从事件驱动机制中一出一个连续的读写事件
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    // 仅在多线程环境下会被调用，目前，nginx在产品环境下还不会以多线程方式运行。
    ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
    // 在正常的工作循环中，将通过调用process_events方法来处理事件。
    // 这个方法仅在ngx_process_events_and_timers方法中调用，它是处理，分发事件的核心
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                   ngx_uint_t flags);

    // 初始化事件驱动模块的方法
    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    // 退出事件驱动模块前调用的方法。
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```



### Nginx连接的数据结构

#### ngx_connection_s

```c++
struct ngx_connection_s {
    /*
    连接未使用时，data成员用于充当连接池中空闲连接链表中的next指针。当连接被使用时，data的意义由使用它的nginx模块而定，
    如在HTTP框架中，data指向ngx_http_request_t请求
    */
    void               *data;

    // 连接对应的读事件
    ngx_event_t        *read;
    // 连接对应的写事件
    ngx_event_t        *write;

    // 套接字句柄
    ngx_socket_t        fd;

    // 直接接受网络字符流的方法
    ngx_recv_pt         recv;
    // 直接发送网络字符流的方法
    ngx_send_pt         send;
    // 以ngx_chain_t链表为参数来接收网络字符流的方法
    ngx_recv_chain_pt   recv_chain;
    // 以ngx_chain_t链表为参数来发送网络字符流的方法
    ngx_send_chain_pt   send_chain;

    // 这个连接对应的ngx_listening_t监听对象，此连接由listening 监听端口的事件建立
    ngx_listening_t    *listening;

    // 这个连接上已经发送出去的字节数
    off_t               sent;

    // 可以记录日志的ngx_log_t对象
    ngx_log_t          *log;

    /*
    内存池，一般在accept一个新连接时，会创建一个内存池，而在这个连接结束时会销毁内存池。
    */
    ngx_pool_t         *pool;

    //连接客户端的socketaddr结构体
    struct sockaddr    *sockaddr;
    // socketaddr结构体的长度
    socklen_t           socklen;
    // 连接客户端字符串形式的IP地址
    ngx_str_t           addr_text;

#if (NGX_SSL)
    ngx_ssl_connection_t  *ssl;
#endif

    // 本机的监听端口对应的socketaddr结构体，也就是listening监听对象中的sockaddr成员
    struct sockaddr    *local_sockaddr;

    /* 
    用于接收、缓存客户端发来的字符流，每个事件消费模块可自由决定从连接池中分配多大的空间给 buffer这个接收缓存字段。
    例如，在HTTP模块中，它的大小决定于client_header_buffer_size配置项
    */ 
    ngx_buf_t          *buffer;

    /*
    该字段用于将当前连接以双向链表元素的形式添加到ngx_cycle_t核心结构体的reusable_connections_queue双向链表中，表示可重用的连接
    */
    ngx_queue_t         queue;

    /*
    连接使用次数。ngx_connection_t结构体每次建立一条来自客户端的连接，或者用于主动向后端服务器发起连接时 （ngx_peer_connection_t也使用它）
    number都会加1
    */
    ngx_atomic_uint_t   number;

    // 处理的请求次数
    ngx_uint_t          requests;

    /*
    缓存中的业务类型。任何事件消费模块都可以自定义需要的标志位。
    这个buffered字段有8位，最多可以同时表示8个不同的业务。第三方模块在自定义buffered标志位时注意不要与可能使用的模块定义的标志位冲突。
    */
    unsigned            buffered:8;

    /*
    本连接记录日志时的级别，它占用了3位，取值范围是0～7，但实际上目前只定义了5个值，由ngx_connection_log_error_e枚举表示，如下：
    typedef enum{
        NGX_ERROR_ALERT = 0,
        NGX_ERROR_ERR,
        NGX_ERROR_INFO,
        NGX_ERROR_IGNORE_ECONNRESET,
        NGX_ERROR_IGNORE_EINVAL,
    } ngx_connection_log_error_e;
    */
    unsigned            log_error:3;     /* ngx_connection_log_error_e */

    /*
    标志位，为1表示独立的连接，如从客户端发起的连接；
    为0表示依靠其他连接的行为而建立起来的非独立连接，如使用upstream机制向后端服务器建立起来的连接
    */
    unsigned            single_connection:1;
    // 标志位，为1表示不期待字符流结束，目前无意义
    unsigned            unexpected_eof:1;
    // 标志位，为1表示连接已超时
    unsigned            timedout:1;
    // 标志位，为1表示连接处理过程中出现错误
    unsigned            error:1;
    // 标志位，为1表示连接已经销毁。这里的连接指的是TCP连接，而不是ngx_connection_t结构体。
    // 当destroy为1时，ngx_connection_t结构体仍然存在，但其对应的套接字，内存池已经不可用。
    unsigned            destroyed:1;

    // 标志位，为1表示连接处于空闲状态，如keepalive请求中两次请求之间的状态
    unsigned            idle:1;
    // 标志位，为1表示连接可重用，它与上面的queue字段是对应使用的
    unsigned            reusable:1;
    // 标志位，为1表示连接关闭
    unsigned            close:1;

    // 标志位，为1表示正在将文件中的数据发往连接的另一端
    unsigned            sendfile:1;
    /*
    标志位，如果为1， 则表示只有在连接套接字对应的发送缓冲区必须满足最低设置的大小阀值，事件驱动模型才会分发该事件。
    */ 
    unsigned            sndlowat:1;
    // 标志位，表示如何使用TCP的nodelay特性。它的取值范围是ngx_connection_tcp_nodelay_e
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    // 标志位，表示如何使用TCP的nopush特性，它的取值范围是ngx_connection_tcp_nopush_e
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */

#if (NGX_HAVE_IOCP)
    unsigned            accept_context_updated:1;
#endif

#if (NGX_HAVE_AIO_SENDFILE)
    // 标志位，为1时表示使用异步I/O的方式将磁盘上文件发送给网络连接的另一端
    unsigned            aio_sendfile:1;
    // 使用异步 I/O 方式发送的文件，busy_sendfile缓冲区保存待发送文件的信息
    ngx_buf_t          *busy_sendfile;
#endif

#if (NGX_THREADS)
    ngx_atomic_t        lock;
#endif
};
```

#### ngx_peer_connection_s

```c++
//定义了load_balance模块实现负载均衡算法的回调函数和相关字段
struct ngx_peer_connection_s {
    // 一个主动连接实际上也需要ngx_connection_t结构体中的大部分成员，并且出于重用的考虑而定义了connection成员
    ngx_connection_t                *connection;

    // 远端服务器的socket地址
    struct sockaddr                 *sockaddr;
    // sockaddr的地址长度
    socklen_t                        socklen;
    // 远端服务器的名称
    ngx_str_t                       *name;

    // 表示在连接一个远端服务器时，当前连接出现异常失败后可以重试的次数，也就是允许的最多失败次数
    ngx_uint_t                       tries;

    // 获取连接的方法，如果使用长连接构成的连接池，那么必须要实现get方法
    ngx_event_get_peer_pt            get;
    // 与get方法对应的释放连接的方法
    ngx_event_free_peer_pt           free;
    /*
    这个data指针仅用于和上面的get,free方法配合传递参数，他的具体含义与实现get方法，free方法的模块相关
    */
    void                            *data;

#if (NGX_SSL)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

#if (NGX_THREADS)
    ngx_atomic_t                    *lock;
#endif

    // 本机地址信息
    ngx_addr_t                      *local;

    // 套接字的接受缓冲区大小
    int                              rcvbuf;

    // 记录日志的ngx_log_t对象
    ngx_log_t                       *log;

    // 标志位，为1表示上面的connection连接已经缓存
    unsigned                         cached:1;

                                     /* ngx_connection_log_error_e */
    unsigned                         log_error:2;
};
```



## HTTP模块

|         模块名称         | 作用                                                         |
| :----------------------: | ------------------------------------------------------------ |
|     ngx_http_module      | 核心模块                                                     |
|   ngx_http_core_module   |                                                              |
| ngx_http_upstream_module |                                                              |
|    ngx_http_module_t     | 管理所有HTTP模块的配置项，每一个HTTP模块，都必须实现ngx_http_module_t接口 |
|                          |                                                              |
|                          |                                                              |

