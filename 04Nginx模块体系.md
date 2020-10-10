# Nginx模块体系

## 模块体系

### 结构定义

在Nginx 1.9.2版本中，`ngx_module_t`定义在<core/ngx_conf_file.h>中

```c

/*
在执行configure命令时仅使用-add-module参数添加了第三方HTTP过滤模块。这里没有把默认未编译进Nginx的官方HTTP过滤模块考虑进去。这样，在
configure执行完毕后，Nginx各HTTP过滤模块的执行顺序就确定了。默认HTTP过滤模块间的顺序必须如图6-1所示，因为它们是“写死”在
auto/modules(auto/sources)脚本中的。读者可以通过阅读这个modules脚本的源代码了解Nginx是如何根据各官方过滤模块功能的不同来决定它们的顺序的

默认即编译进Nginx的HTTP过滤模块
┏━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃默认即编译进Nginx的HTTP过滤模块     ┃    功能                                                          ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP头部做处理。在返回200成功时，根据请求中If-              ┃
┃                                    ┃Modified-Since或者If-Unmodified-Since头部取得浏览器缓存文件的时   ┃
┃ngx_http_not_modified filter module ┃                                                                  ┃
┃                                    ┃间，再分析返回用户文件的最后修改时间，以此决定是否直接发送304     ┃
┃                                    ┃ Not Modified响应给用户                                           ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  处理请求中的Range信息，根据Range中的要求返回文件的一部分给      ┃
┃ngx_http_range_body_filter_module   ┃                                                                  ┃
┃                                    ┃用户                                                              ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP包体做处理。将用户发送的ngx_chain_t结构的HTTP包         ┃
┃                                    ┃体复制到新的ngx_chain_t结构中（都是各种指针的复制，不包括实际     ┃
┃ngx_http_copy_filter_module         ┃                                                                  ┃
┃                                    ┃HTTP响应内容），后续的HTTP过滤模块处埋的ngx_chain_t类型的成       ┃
┃                                    ┃员都是ngx_http_copy_filter_module模块处理后的变量                 ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP头部做处理。允许通过修改nginx.conf配置文件，在返回      ┃
┃ngx_http_headers filter module      ┃                                                                  ┃
┃                                    ┃给用户的响应中添加任意的HTTP头部                                  ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP头部做处理。这就是执行configure命令时提到的http_        ┃
┃ngx_http_userid filter module       ┃                                                                  ┃
┃                                    ┃userid module模块，它基于cookie提供了简单的认证管理功能           ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  可以将文本类型返回给用户的响应包，按照nginx．conf中的配置重新   ┃
┃ngx_http_charset filter module      ┃                                                                  ┃
┃                                    ┃进行编码，再返回给用户                                            ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  支持SSI（Server Side Include，服务器端嵌入）功能，将文件内容包  ┃
┃ngx_http_ssi_filter module          ┃                                                                  ┃
┃                                    ┃含到网页中并返回给用户                                            ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP包体做处理。                             它仅应用于     ┃
┃ngx_http_postpone_filter module     ┃subrequest产生的子请求。它使得多个子请求同时向客户端发送响应时    ┃
┃                                    ┃能够有序，所谓的“有序”是揩按照构造子请求的顺序发送响应          ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  对特定的HTTP响应包体（如网页或者文本文件）进行gzip压缩，再      ┃
┃ngx_http_gzip_filter_module         ┃                                                                  ┃
┃                                    ┃把压缩后的内容返回给用户                                          ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃ngx_http_range_header_filter module ┃  支持range协议                                                   ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃ngx_http_chunked filter module      ┃  支持chunk编码                                                   ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                                    ┃  仅对HTTP头部做处理。该过滤模块将会把r->headers out结构体        ┃
┃                                    ┃中的成员序列化为返回给用户的HTTP响应字符流，包括响应行(如         ┃
┃ngx_http_header filter module       ┃                                                                  ┃
┃                                    ┃HTTP/I.1 200 0K)和响应头部，并通过调用ngx_http_write filter       ┃
┃                                    ┃ module过滤模块中的过滤方法直接将HTTP包头发送到客户端             ┃
┣━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃ngx_http_write filter module        ┃  仅对HTTP包体做处理。该模块负责向客户端发送HTTP响应              ┃
┗━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
第三方过滤模块为何要在ngx_http_headers_filter_module模块之后、ngx_http_userid_filter_module横块之前
*/

/*
配置模块与核心模块都是与Nginx框架密切相关的，是其他模块的基础。而事件模块则是HTTP模块和mail模块的基础，原因参见8.2.2节。
HTTP模块和mail模块的“地位”相似，它们都更关注于应用层面。在事件模块中，ngx_event_core module事件模块是其他所有事件模块
的基础；在HTTP模块中，ngx_http_core module模块是其他所有HTTP模块的基础；在mail模块中，ngx_mail_core module模块是其他所
有mail模块的基础。
*/
struct ngx_module_s {//相关空间初始化，赋值等可以参考ngx_http_block
    /*
    对于一类模块（由下面的type成员决定类别）而言，ctx_index表示当前模块在这类模块中的序号。这个成员常常是由管理这类模块的一个
    Nginx核心模块设置的，对于所有的HTTP模块而言，ctx_index是由核心模块ngx_http_module设置的。ctx_index非常重要，Nginx的模块化
    设计非常依赖于各个模块的顺序，它们既用于表达优先级，也用于表明每个模块的位置，借以帮助Nginx框架快速获得某个模块的数据（）
    */
    //ctx index表明了模块在相同类型模块中的顺序
    ngx_uint_t            ctx_index; //初始化赋值见ngx_http_block, 这个值是按照在http_modules中的位置顺序来排序的，见ngx_http_block

   /*
    index表示当前模块在ngx_modules数组中的序号。注意，ctx_index表示的是当前模块在一类模块中的序号，而index表示当前模块在所有模块中的序号，
    它同样关键。Nginx启动时会根据ngx_modules数组设置各模块的index值。例如：
    ngx_max_module = 0;
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = ngx_max_module++;
    }
    */
    ngx_uint_t            index; //模块在所有模块中的序号，是第几个模块

    //spare系列的保留变量，暂未使用
    ngx_uint_t            spare0;
    ngx_uint_t            spare1;
    ngx_uint_t            spare2;
    ngx_uint_t            spare3;

    
    //模块的版本，便于将来的扩展。目前只有一种，默认为1
    ngx_uint_t            version;

    
    /*
    ctx用于指向一类模块的上下文结构体，为什么需要ctx呢？因为前面说过，Nginx模块有许多种类，不同类模块之间的功能差别很大。例如，
    事件类型的模块主要处理I/O事件相关的功能，HTTP类型的模块主要处理HTTP应用层的功能。这样，每个模块都有了自己的特性，而ctx将会
    指向特定类型模块的公共接口。例如，在HTTP模块中，ctx需要指向ngx_http_module_t结构体,可以参考例如ngx_http_core_module, 
    event模块中，指向ngx_event_module_t
    */
    void                 *ctx; //HTTP框架初始化时完成的
    ngx_command_t        *commands; //commands将处理nginx.conf中的配置项

    
    /*
    
    结构体中的type字段决定了该模块的模块类型：
    
    core module对应的值为NGX_CORE_MODULE
    
    http module对应的值为NGX_HTTP_MODULE
    
    mail module对应的值为NGX_MAIL_MODULE
    
    event module对应的值为NGX_EVENT_MODULE
    
    每个大模块中都有一些具体功能实现的子模块，如ngx_lua模块就是http module中的子模块。
    
    type表示该模块的类型，它与ctx指针是紧密相关的。在官方Nginx中，它的取值范围是以下5种：NGX_HTTP_MODULE、NGX_CORE_MODULE、
    NGX_CONF_MODULE、NGX_EVENT_MODULE、NGX_MAIL_MODULE。这5种模块间的关系参考图8-2。实际上，还可以自定义新的模块类型
    */
    ngx_uint_t            type;


    /*
    在Nginx的启动、停止过程中，以下7个函数指针表示有7个执行点会分别调用这7种方法（  ）。对于任一个方法而言，
    如果不需要Nginx在某个时刻执行它，那么简单地把它设为NULL空指针即可
    */

    /*
    对于下列回调方法：init_module、init_process、exit_process、exit_master，调用它们的是Nginx的框架代码。换句话说，这4个回调方法
    与HTTP框架无关，即使nginx.conf中没有配置http {...}这种开启HTTP功能的配置项，这些回调方法仍然会被调用。因此，通常开发HTTP模块
    时都把它们设为NULL空指针。这样，当Nginx不作为Web服务器使用时，不会执行HTTP模块的任何代码。
     */
    

    /*虽然从字面上理解应当在master进程启动时回调init_master，但到目前为止，框架代码从来不会调用它，因此，可将init_master设为NULL */
    ngx_int_t           (*init_master)(ngx_log_t *log); //实际上没用
    /*init_module回调方法在初始化所有模块时被调用。在master/worker模式下，这个阶段将在启动worker子进程前完成*/
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle); //ngx_init_cycle中调用，在解析玩所有的nginx.conf配置后才会调用模块的ngx_conf_parse
    /* init_process回调方法在正常服务前被调用。在master/worker模式下，多个worker子进程已经产生，在每个worker进程
    的初始化过程会调用所有模块的init_process函数*/
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle); //ngx_worker_process_init或者ngx_single_process_cycle中调用
    
    /* 由于Nginx暂不支持多线程模式，所以init_thread在框架代码中没有被调用过，设为NULL*/
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle); //实际上没用
    
    // 同上，exit_thread也不支持，设为NULL
    void                (*exit_thread)(ngx_cycle_t *cycle);//实际上没用
    
    /* exit_process回调方法在服务停止前调用。在master/worker模式下，worker进程会在退出前调用它，见ngx_worker_process_exit*/
    void                (*exit_process)(ngx_cycle_t *cycle); //ngx_single_process_cycle 或者 ngx_worker_process_exit中调用
    // exit_master回调方法将在master进程退出前被调用
    void                (*exit_master)(ngx_cycle_t *cycle); //ngx_master_process_exit中调用

    
    /*以下8个spare_hook变量也是保留字段，目前没有使用，但可用Nginx提供的NGX_MODULE_V1_PADDING宏来填充。看一下该宏的定义：
    #define NGX_MODULE_V1_PADDING  0, 0, 0, 0, 0, 0, 0, 0*/
    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};

```

### **模块的函数指针表**

ngx_module_t里的成员type相当于类型的标记（RTTI），而ctx则是一个函数指针表，类似于C++里的虚函数表，不同的模块有不同的定义，实现了模块的“子类化”，这是因为C语言没有C++的继承机制。

ctx类型是void*，意味着它的内容是不确定的，必须结合type才能确定ctx的具体含义。

**core模块**

core模块的ctx是：

```c
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);		//创建配置结构体
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);	//初始化配置结构体
} ngx_core_module_t;
```

**http模块**

http模块的ctx是ngx_http_module_t，

```c
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```



### 模块的类图

如果从C++得视角来观察Nginx的模块架构，那么ngx_module_t就是一个抽象基类，ngx_core_module_t、ngx_http_module_t以ctx的方式继承了ngx_module_t，最后在各个具体模块实现了类的函数指针，是它们的实现类。

![Nginx模块的类图](.\images\Nginx模块的类图.jpg)

### 模块的组织方式

**ngx_modules.c**

```c
//定义在objs/ngx_modules.c，由configure脚本动态生产
ngx_module_t *ngx_modules[] = {   //模块指针数组                                
    &ngx_core_module,             //core模块                     
    &ngx_errlog_module,           //errlog模块                        
    &ngx_conf_module,             //conf模块                      
    &ngx_regex_module,            //regex模块                       
    &ngx_events_module,           //event模块                       
    &ngx_event_core_module,       //event核心模块                       
    &ngx_epoll_module,            //epoll模块                   
    &ngx_http_module,             //http模块
    &ngx_http_core_module, 		  //http核心模块
    ...
    NULL
};
```

**ngx_cycle_t**

是整个Nginx架构中的核心数据结构，是Nginx运行整个生命周期都要使用到的数据，其中有一个数组成员modules，保存了运行时的所有模块。

```c
//core/ngx_cycle.h
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;	//运行时模块指针数组，所有的模块都在这里
    ngx_uint_t                modules_n;//模块数组中可用的序号
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;
    time_t                    connections_reuse_time;

    ngx_array_t               listening;
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;

    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```

**模块的索引序号**

ngx_module_t里的index成员标记了模块在modules数组的索引位置，相当于一级索引；而ctx_index是模块在本类型的序号，相当于二级索引。



**模块的初始化**

**index的初始化**





