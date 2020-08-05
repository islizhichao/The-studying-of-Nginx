## 3.4 HTTP模块的数据结构

定义HTTP模块方式

`ngx_module_t ngx_http_mytest_modules`

```c
//  src / core / ngx_conf_file.c
typedef struct ngx_module_s ngx_module_t;
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

其中最重要的是设置ctx和commands这两个成员，对于Http类型的模块来说。`ngx_module_t`中的`ctx`指针必须指向`ngx_http_module_t`接口。

```c
typedef struct {
　　ngx_int_t (*preconfiguration)(ngx_conf_t *cf);                   //解析配置文件前调用
　　ngx_int_t (*postconfiguration)(ngx_conf_t *cf);                  //完成配置文件解析后调用
　　void *(*create_main_conf)(ngx_conf_t *cf);                       //当需要创建数据结构用户存储main级别的全局配置项时候调用
　　char *(*init_main_conf)(ngx_conf_t *cf, void *conf);             //初始化main级别配置项
　　void *(*create_srv_conf)(ngx_conf_t *cf);                        //当需要创建数据结构用户存储srv级别的全局配置项时候调用
　　char *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf); //合并server级别的配置项
　　void *(*create_loc_conf)(ngx_conf_t *cf);                        //当需要创建数据结构用户存储loc级别的全局配置项时候调用
　　char *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf); //合并location级别的配置项
} ngx_http_module_t;
```

`commands`数组用于定义模块的配置文件参数，每一个数组元素都是`ngx_command_t`类型，数组结尾用`ngx_null_command`表示。

```c
typedef struct ngx_command_s     ngx_command_t;
*/ //每个module都有自己的command，见ngx_modules中对应模块的command。 每个进程中都有一个唯一的ngx_cycle_t核心结构体，它有一个成员conf_ctx维护着所有模块的配置结构体
struct ngx_command_s { //所有配置的最初源头在ngx_init_cycle
    ngx_str_t             name;//配置项名称，如"gzip"
    /*配置项类型，type将指定配置项可以出现的位置。例如，出现在server{}或location{}中，以及它可以携带的参数个数*/
    /*
    type决定这个配置项可以在哪些块（如http、server、location、if、upstream块等）
中出现，以及可以携带的参数类型和个数等。
注意，type可以同时取多个值，各值之间用|符号连接，例如，type可以取
值为NGX_TTP_MAIN_CONF | NGX_HTTP_SRV_CONFI | NGX_HTTP_LOC_CONF | NGX_CONF_TAKE。 
    */
    ngx_uint_t            type; //取值可能为NGX_HTTP_LOC_CONF | NGX_CONF_TAKE2等

    //出现了name中指定的配置项后，将会调用set方法处理配置项的参数
    //cf里面存储的是从配置文件里面解析出的内容，conf是最终用来存储解析内容的内存空间，cmd为存到空间的那个地方(使用偏移量来衡量)
    //在ngx_conf_parse解析完参数后，在ngx_conf_handler中执行
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf); //参考上面的图形化信息
    ngx_uint_t            conf;//crate分配内存的时候的偏移量 NGX_HTTP_LOC_CONF_OFFSET NGX_HTTP_SRV_CONF_OFFSET
    /*通常用于使用预设的解析方法解析配置项，这是配置模块的一个优秀设计。它需要与conf配合使用*/
    ngx_uint_t            offset;

    //如果使用Nginx预设的配置项解析方法，就需要根据这些预设方法来决定post的使用方式。表4-4说明了post相对于14个预设方法的用途。
    /*
    参考14个回调方法后面的 
    if (cmd->post) {
        post = cmd->post;
        return post->post_handler(cf, post, fp);
    }
    */
    void                 *post; 
};

//ngx_null_command只是一个空的ngx_command_t，表示模块的命令数组解析完毕，如下所示：
#define ngx_null_command  { ngx_null_string, 0, NULL, 0, 0, NULL }
```





## 3.5 定义自己的Http模块





## 3.6 处理用户请求

```c
/*
对于每个ngx_http_request_t请求来说，只能访问一个上游服务器，但对于一个客户端请求来说，可以派生出许多子请求，任何一个子请求都
可以访问一个上游服务器，这些子请求的结果组合起来就可以使来自客户端的请求处理复杂的业务。
*/
//ngx_http_parse_request_line解析请求行， ngx_http_process_request_headers解析头部行(请求头部)
//请求的所有信息都可以都可以在ngx_http_request_s结构获取到
struct ngx_http_request_s { //当接收到客户端请求数据后，调用ngx_http_create_request中创建并赋值
    uint32_t                          signature;         /* "HTTP" */ 

    /*
        在接收到客户端数据后，会创建一个ngx_http_request_s，其connection成员指向对应的accept成功后获取到的连接信息
    ngx_connection_t，见ngx_http_create_request
        这个请求对应的客户端连接  如果该r是子请求，其connection成员指向顶层root父请求的ngx_connection_t，因为它们都是对应的同一个客户端连接,见ngx_http_subrequest
    */
    ngx_connection_t                 *connection; 

    /*
    ctx与ngx_http_conf_ctxt结构的3个数组成员非常相似，它们都
    表示指向void指针的数组。HTTP框架就是在ctx数组中保存所有HTTP模块上下文结构体的指针的,所有模块的请求上下文空间在
    ngx_http_create_request中创建。获取和设置分别在ngx_http_get_module_ctx和ngx_http_set_ctx，为每个请求创建ngx_http_request_s的时候
    都会为该请求的ctx[]为所有的模块创建一个指针，也就是每个模块在ngx_http_request_s中有一个ctx
    */ //在应答完后，在ngx_http_filter_finalize_request会把ctx指向的空间全部清0  参考4.5节 
    void                            **ctx; //指向存放所有HTTP模块的上下文结构体的指针数组,实际上发送给客户端的应答完成后，会把ctx全部置0
    /*
    当客户端建立连接后，并发送请求数据过来后，在ngx_http_create_request中从ngx_http_connection_t->conf_ctx获取这三个值，也就是根据客户端连接
    本端所处IP:port所对应的默认server{}块上下文，如果是以下情况:ip:port相同，单在不同的server{}块中，那么有可能客户端请求过来的时候携带的host
    头部项的server_name不在默认的server{}中，而在另外的server{}中，所以需要通过ngx_http_set_virtual_server重新获取server{}和location{}上下文配置
    例如:
        server {  #1
            listen 1.1.1.1:80;
            server_name aaa
        }

        server {   #2
            listen 1.1.1.1:80;
            server_name bbb
        }
        这个配置在ngx_http_init_connection中把ngx_http_connection_t->conf_ctx指向ngx_http_addr_conf_s->default_server,也就是指向#1,然后
        ngx_http_create_request中把main_conf srv_conf  loc_conf 指向#1,
        但如果请求行的头部的host:bbb，那么需要重新获取对应的server{} #2,见ngx_http_set_virtual_server
*/ //ngx_http_create_request和ngx_http_set_virtual_server 已经rewrite过程中(例如ngx_http_core_find_location 
//ngx_http_core_post_rewrite_phase ngx_http_internal_redirect ngx_http_internal_redirect 子请求ngx_http_subrequest)都可能对他们赋值
    void                            **main_conf; //指向请求对应的存放main级别配置结构体的指针数组
    void                            **srv_conf; //指向请求对应的存放srv级别配置结构体的指针数组   赋值见ngx_http_set_virtual_server
    void                            **loc_conf; //指向请求对应的存放loc级别配置结构体的指针数组   赋值见ngx_http_set_virtual_server

    /*
     在接收完HTTP头部，第一次在业务上处理HTTP请求时，HTTP框架提供的处理方法是ngx_http_process_request。但如果该方法无法一次处
     理完该请求的全部业务，在归还控制权到epoll事件模块后，该请求再次被回调时，将通过ngx_http_request_handler方法来处理，而这个
     方法中对于可读事件的处理就是调用read_event_handler处理请求。也就是说，HTTP模块希望在底层处理请求的读事件时，重新实现read_event_handler方法

     //在读取客户端来的包体时，赋值为ngx_http_read_client_request_body_handler
     丢弃客户端的包体时，赋值为ngx_http_discarded_request_body_handler
     */ //注意ngx_http_upstream_t和ngx_http_request_t都有该成员 分别在ngx_http_request_handler和ngx_http_upstream_handler中执行
    ngx_http_event_handler_pt         read_event_handler;  

    /* 与read_event_handler回调方法类似，如果ngx_http_request_handler方法判断当前事件是可写事件，则调用write_event_handler处理请求 */
    /*请求行和请求头部解析完成后，会在ngx_http_handler中赋值为ngx_http_core_run_phases   子请求的的handler为ngx_http_handler
       当发送响应的时候，如果一次没有发送完，则设在为ngx_http_writer
     */ //注意ngx_http_upstream_t和ngx_http_request_t都有该成员 分别在ngx_http_request_handler和ngx_http_upstream_handler中执行
     //如果采用buffer方式缓存后端包体，则在发送包体给客户端浏览器的时候，会把客户端连接的write_e_hand置为ngx_http_upstream_process_downstream
     //在触发epoll_in的同时也会触发epoll_out，从而会执行该函数
    ngx_http_event_handler_pt         write_event_handler;//父请求重新激活后的回调方法

#if (NGX_HTTP_CACHE)
//通过ngx_http_upstream_cache_get获取
    ngx_http_cache_t                 *cache;//在客户端请求过来后，在ngx_http_upstream_cache->ngx_http_file_cache_new中赋值r->caceh = ngx_http_cache_t
#endif

    /* 
    如果没有使用upstream机制，那么ngx_http_request_t中的upstream成员是NULL空指针,在ngx_http_upstream_create中创建空间
    */
    ngx_http_upstream_t              *upstream; //upstream机制用到的结构体
    ngx_array_t                      *upstream_states; //创建空间和赋值见ngx_http_upstream_init_request
                                         /* of ngx_http_upstream_state_t */

    /*
    表示这个请求的内存池，在ngx_http_free_request方法中销毁。它与ngx_connection-t中的内存池意义不同，当请求释放时，TCP连接可能并
    没有关闭，这时请求的内存池会销毁，但ngx_connection_t的内存池并不会销毁
     */
    ngx_pool_t                       *pool;
    //其中，header_in指向Nginx收到的未经解析的HTTP头部，这里暂不关注它（header_in就是接收HTTP头部的缓冲区）。 header_in存放请求行，headers_in存放头部行
    //请求行和请求头部内容都在该buffer中
    ngx_buf_t                        *header_in;//用于接收HTTP请求内容的缓冲区，主要用于接收HTTP头部，该指针指向ngx_connection_t->buffer

    //类型的headers_in则存储已经解析过的HTTP头部。
    /*常用的HTTP头部信息可以通过r->headers_in获取，不常用的HTTP头部则需要遍历r->headers_in.headers来遍历获取*/
/*
 ngx_http_process_request_headers方法在接收、解析完HTTP请求的头部后，会把解析完的每一个HTTP头部加入到headers_in的headers链表中，同时会构造headers_in中的其他成员
 */ //参考ngx_http_headers_in，通过该数组中的回调hander来存储解析到的请求行name:value中的value到headers_in的响应成员中，见ngx_http_process_request_headers
    //注意:在需要把客户端请求头发送到后端的话，在请求头后面可能添加有HTTP_相关变量，例如fastcgi，见ngx_http_fastcgi_create_request
    ngx_http_headers_in_t             headers_in; //http头部行解析后的内容都由该成员存储  header_in存放请求行，headers_in存放头部行
    //只要指定headers_out中的成员，就可以在调用ngx_http_send_header时正确地把HTTP头部发出
    //HTTP模块会把想要发送的HTTP响应信息放到headers_out中，期望HTTP框架将headers_out中的成员序列化为HTTP响应包发送给用户
    ngx_http_headers_out_t            headers_out; 
    //如果是upstream赋值的来源是后端服务器会有的头部行中拷贝，参考ngx_http_upstream_headers_in中的copy_handler

/*
接收完请求的包体后，可以在r->request_body->temp_file->file中获取临时文件（假定将r->request_body_in_file_only标志位设为1，那就一定可以
在这个变量获取到包体。）。file是一个ngx_file_t类型。这里，我们可以从
r->request_body->temp_file->file.name中获取Nginx接收到的请求包体所在文件的名称（包括路径）。
*/ //在ngx_http_read_client_request_body中分配存储空间 读取的客户端包体存储在r->request_body->bufs链表和临时文件r->request_body->temp_file中 ngx_http_read_client_request_body
//读取客户包体即使是存入临时文件中，当所有包体读取完毕后(见ngx_http_do_read_client_request_body)，还是会让r->request_body->bufs指向文件中的相关偏移内存地址
//向上游发送包体u->request_bufs(ngx_http_fastcgi_create_request),接收客户端的包体在r->request_body
    ngx_http_request_body_t          *request_body; //接收HTTP请求中包体的数据结构，为NULL表示还没有分配空间
    //min(lingering_time,lingering_timeout)这段时间内可以继续读取数据，如果客户端有发送数据过来，见ngx_http_set_lingering_close
    time_t                            lingering_time; //延迟关闭连接的时间
    //ngx_http_request_t结构体中有两个成员表示这个请求的开始处理时间：start sec成员和start msec成员
    /*
     当前请求初始化时的时间。start sec是格林威治时间1970年1月1日凌晨0点0分0秒到当前时间的秒数。如果这个请求是子请求，则该时间
     是子请求的生成时间；如果这个请求是用户发来的请求，则是在建立起TCP连接后，第一次接收到可读事件时的时间
     */
    time_t                            start_sec;
    ngx_msec_t                        start_msec;//与start_sec配合使用，表示相对于start_set秒的毫秒偏移量



//以下9个成员都是ngx_http_proces s_request_line方法在接收、解析HTTP请求行时解析出的信息
/*
注意　Nginx中对内存的控制相当严格，为了避免不必要的内存开销，许多需要用到的成员都不是重新分配内存后存储的，而是直接指向用户请求中的相应地址。
例如，method_name.data、request_start这两个指针实际指向的都是同一个地址。而且，因为它们是简单的内存指针，不是指向字符串的指针，所以，在大部分情况下，都不能将这些u_char*指针当做字符串使用。
*/ //NGX_HTTP_GET | NGX_HTTP_HEAD等,为NGX_HTTP_HEAD表示只需要发送HTTP头部字段
    /* HTTP2的method赋值见ngx_http_v2_parse_method */
    ngx_uint_t                        method; //对应客户端请求中请求行的请求方法GET、POS等，取值见NGX_HTTP_GET,也可以用下面的method_name进行字符串比较
/*
http_protocol指向用户请求中HTTP的起始地址。
http_version是Nginx解析过的协议版本，它的取值范围如下：
#define NGX_HTTP_VERSION_9                 9
#define NGX_HTTP_VERSION_10                1000
#define NGX_HTTP_VERSION_11                1001
建议使用http_version分析HTTP的协议版本。
最后，使用request_start和request_end可以获取原始的用户请求行。
*/
    ngx_uint_t                        http_version;//http_version是Nginx解析过的协议版本，它的取值范围如下：
    /* 如果是HTTP2，则赋值见ngx_http_v2_construct_request_line */
    ngx_str_t                         request_line; //请求行内容  


/*
2016/01/07 12:38:01[      ngx_http_process_request_line,  1002]  [debug] 20090#20090: *14 http request line: "GET /download/nginx-1.9.2.rar?st=xhWL03HbtjrojpEAfiD6Mw&e=1452139931 HTTP/1.1"
2016/01/07 12:38:01[       ngx_http_process_request_uri,  1223]  [debug] 20090#20090: *14 http uri: "/download/nginx-1.9.2.rar"
2016/01/07 12:38:01[       ngx_http_process_request_uri,  1226]  [debug] 20090#20090: *14 http args: "st=xhWL03HbtjrojpEAfiD6Mw&e=1452139931"
2016/01/07 12:38:01[       ngx_http_process_request_uri,  1229]  [debug] 20090#20090: *14 http exten: "rar"
*/

    
//ngx_str_t类型的uri成员指向用户请求中的URI。同理，u_char*类型的uri_start和uri_end也与request_start、method_end的用法相似，唯一不
//同的是，method_end指向方法名的最后一个字符，而uri_end指向URI结束后的下一个地址，也就是最后一个字符的下一个字符地址（HTTP框架的行为），
//这是大部分u_char*类型指针对“xxx_start”和“xxx_end”变量的用法。
    //http://10.135.10.167/mytest中的/mytest  http://10.135.10.167/mytest?abc?ttt中的/mytest  
    //同时"GET /mytest?abc?ttt HTTP/1.1"中的mytest和uri中的一样    
    ngx_str_t                         uri; 
    //arg指向用户请求中的URL参数。  http://10.135.10.167/mytest?abc?ttt中的abc?ttt   
    //同时"GET /mytest?abc?ttt HTTP/1.1"中的mytest?abc?ttt和uri中的一样    

 /*把请求中GET /download/nginx-1.9.2.rar?st=xhWL03HbtjrojpEAfiD6Mw&e=1452139931 HTTP/1.1的st和e形成变量$arg_st #arg_e，value分别
为xhWL03HbtjrojpEAfiD6Mw 1452139931即$arg_st=xhWL03HbtjrojpEAfiD6Mw，#arg_e=1452139931，见ngx_http_arg */
    ngx_str_t                         args;
    /*
    ngx_str_t类型的extern成员指向用户请求的文件扩展名。例如，在访问“GET /a.txt HTTP/1.1”时，extern的值是{len = 3, data = "txt"}，
    而在访问“GET /a HTTP/1.1”时，extern的值为空，也就是{len = 0, data = 0x0}。
    uri_ext指针指向的地址与extern.data相同。
    */
    ngx_str_t                         exten; //http://10.135.10.167/mytest/ac.txt中的txt
/*
url参数中出现+、空格、=、%、&、#等字符的解决办法 
url出现了有+，空格，/，?，%，#，&，=等特殊符号的时候，可能在服务器端无法获得正确的参数值，如何是好？
解决办法
将这些字符转化成服务器可以识别的字符，对应关系如下：
URL字符转义

用其它字符替代吧，或用全角的。

+    URL 中+号表示空格                      %2B   
空格 URL中的空格可以用+号或者编码           %20 
/   分隔目录和子目录                        %2F     
?    分隔实际的URL和参数                    %3F     
%    指定特殊字符                           %25     
#    表示书签                               %23     
&    URL 中指定的参数间的分隔符             %26     
=    URL 中指定参数的值                     %3D
*/
//unparsed_uri表示没有进行URL解码的原始请求。例如，当uri为“/a b”时，unparsed_uri是“/a%20b”（空格字符做完编码后是%20）。
    ngx_str_t                         unparsed_uri;//参考:为什么要对URI进行编码:
    /* HTTP2的method赋值见ngx_http_v2_parse_method，在组新的HTTP2头部行后，赋值见ngx_http_v2_construct_request_line */ 
    ngx_str_t                         method_name;//见method   GET  POST等
    ngx_str_t                         http_protocol;//GET /sample.jsp HTTP/1.1  中的HTTP/1.1


/* 当ngx_http_header_filter方法无法一次性发送HTTP头部时，将会有以下两个现象同时发生:请求的out成员中将会保存剩余的响应头部,见ngx_http_header_filter */    
/* 表示需要发送给客户端的HTTP响应。out中保存着由headers_out中序列化后的表示HTTP头部的TCP流。在调用ngx_http_output_filter方法后，
out中还会保存待发送的HTTP包体，它是实现异步发送HTTP响应的关键 */
    ngx_chain_t                      *out;//ngx_http_write_filter把in中的数据拼接到out后面，然后调用writev发送，没有发送完
    
    /* 当前请求既可能是用户发来的请求，也可能是派生出的子请求，而main则标识一系列相关的派生子请
    求的原始请求，我们一般可通过main和当前请求的地址是否相等来判断当前请求是否为用户发来的原始请求 */

    //main成员始终指向一系列有亲缘关系的请求中的唯一的那个原始请求,初始赋值见ngx_http_create_request
    //客户端的建立连接的时候r->main =r(ngx_http_create_request),如果是创建子请求，sr->main = r->main(ngx_http_subrequest)子请求->main=最上层的r
    /* 主请求保存在main字段中，这里其实就是最上层跟请求，例如当前是四层子请求，则main始终指向第一层父请求，
        而不是第三次父请求，parent指向第三层父请求 */  
    ngx_http_request_t               *main; //赋值见ngx_http_subrequest
    ngx_http_request_t               *parent;//当前请求的父请求。注意，父请求未必是原始请求 赋值见ngx_http_subrequest

    //ngx_http_subrequest中赋值，表示对应的子请求r，该结构可以表示子请求信息
    //postponed删除在ngx_http_finalize_request     
    //当客户端请求需要通过多个subrequest访问后端的时候，就需要对这多个后端的应答进行合适的顺序整理才能发往客户端
    ngx_http_postponed_request_t     *postponed; //与subrequest子请求相关的功能  postponed中数据依次发送参考ngx_http_postpone_filter方法
    ngx_http_post_subrequest_t       *post_subrequest;/* 保存回调handler及数据，在子请求执行完，将会调用 */  
    
/* 所有的子请求都是通过posted_requests这个单链表来链接起来的，执行post子请求时调用的
ngx_http_run_posted_requests方法就是通过遍历该单链表来执行子请求的 */ 
//ngx_http_post_request中创建ngx_http_posted_request_t空间  
//ngx_http_post_request将该子请求挂载在主请求的posted_requests链表队尾，在ngx_http_run_posted_requests中执行
    ngx_http_posted_request_t        *posted_requests; //通过posted_requests就把各个子请求以单向链表的数据结构形式组织起来

/*
全局的ngx_http_phase_engine_t结构体中定义了一个ngx_http_phase_handler_t回调方法组成的数组，而phase_handler成员则与该数组配合使用，
表示请求下次应当执行以phase_handler作为序号指定的数组中的回调方法。HTTP框架正是以这种方式把各个HTTP摸块集成起来处理请求的
*///phase_handler实际上是该阶段的处理方法函数在ngx_http_phase_engine_t->handlers数组中的位置
    ngx_int_t                         phase_handler; 
    //表示NGX HTTP CONTENT PHASE阶段提供给HTTP模块处理请求的一种方式，content handler指向HTTP模块实现的请求处理方法,在ngx_http_core_content_phase中执行
    //ngx_http_proxy_handler  ngx_http_redis2_handler  ngx_http_fastcgi_handler等
    ngx_http_handler_pt               content_handler; ////在ngx_http_update_location_config中赋值给r->content_handler = clcf->handler;
/*
    在NGX_HTTP_ACCESS_PHASE阶段需要判断请求是否具有访问权限时，通过access_code来传递HTTP模块的handler回调方法的返回值，如果access_code为0，
则表示请求具备访问权限，反之则说明请求不具备访问权限

    NGXHTTPPREACCESSPHASE、NGX_HTTP_ACCESS_PHASE、NGX HTTPPOST_ACCESS_PHASE，很好理解，做访问权限检查的前期、中期、后期工作，
其中后期工作是固定的，判断前面访问权限检查的结果（状态码存故在字段r->access_code内），如果当前请求没有访问权限，那么直接返回状
态403错误，所以这个阶段也无法去挂载额外的回调函数。
*/
    ngx_uint_t                        access_code; //赋值见ngx_http_core_access_phase
    /*
    ngx_http_core_main_conf_t->variables数组成员的结构式ngx_http_variable_s， ngx_http_request_s->variables数组成员结构是ngx_variable_value_t,
    这两个结构的关系很密切，一个所谓变量，一个所谓变量值

    r->variables这个变量和cmcf->variables是一一对应的，形成var_ name与var_value对，所以两个数组里的同一个下标位置元素刚好就是
相互对应的变量名和变量值，而我们在使用某个变量时总会先通过函数ngx_http_get_variable_index获得它在变量名数组里的index下标，也就是变
量名里的index字段值，然后利用这个index下标进而去变量值数组里取对应的值
    */ //分配的节点数见ngx_http_create_request，和ngx_http_core_main_conf_t->variables一一对应
    //变量ngx_http_script_var_code_t->index表示Nginx变量$file在ngx_http_core_main_conf_t->variables数组内的下标，对应每个请求的变量值存储空间就为r->variables[code->index],参考ngx_http_script_set_var_code
    ngx_http_variable_value_t        *variables; //注意和ngx_http_core_main_conf_t->variables的区别

#if (NGX_PCRE)
    /*  
     例如正则表达式语句re.name= ^(/download/.*)/media/(.*)/tt/(.*)$，  s=/download/aa/media/bdb/tt/ad,则他们会匹配，同时匹配的
     变量数有3个，则返回值为3+1=4,如果不匹配则返回-1

     这里*2是因为获取前面例子中的3个变量对应的值需要成对使用r->captures，参考ngx_http_script_copy_capture_code等
     */
    ngx_uint_t                        ncaptures; //赋值见ngx_http_regex_exec   //最大的$n*2
    int                              *captures; //每个不同的正则解析之后的结果，存放在这里。$1,$2等
    u_char                           *captures_data; //进行正则表达式匹配的原字符串，例如http://10.135.2.1/download/aaa/media/bbb.com中的/download/aaa/media/bbb.com
#endif

/* limit_rate成员表示发送响应的最大速率，当它大于0时，表示需要限速。limit rate表示每秒可以发送的字节数，超过这个数字就需要限速；
然而，限速这个动作必须是在发送了limit_rate_after字节的响应后才能生效（对于小响应包的优化设计） */
//实际最后通过ngx_writev_chain发送数据的时候，还会限制一次
    size_t                            limit_rate; //限速的相关计算方法参考ngx_http_write_filter
    size_t                            limit_rate_after;

    /* used to learn the Apache compatible response length without a header */
    size_t                            header_size; //所有头部行内容之和，可以参考ngx_http_header_filter

    off_t                             request_length; //HTTP请求的全部长度，包括HTTP包体

    ngx_uint_t                        err_status; //错误码，取值为NGX_HTTP_BAD_REQUEST等

    //当连接建立成功后，当收到客户端的第一个请求的时候会通过ngx_http_wait_request_handler->ngx_http_create_request创建ngx_http_request_t
    //同时把r->http_connection指向accept客户端连接成功时候创建的ngx_http_connection_t，这里面有存储server{}上下文ctx和server_name等信息
    //该ngx_http_request_t会一直有效，除非关闭连接。因此该函数只会调用一次，也就是第一个客户端请求报文过来的时候创建，一直持续到连接关闭
    //该结构存储了服务器端接收客户端连接时，服务器端所在的server{]上下文ctx  server_name等配置信息
    ngx_http_connection_t            *http_connection; //存储ngx_connection_t->data指向的ngx_http_connection_t，见ngx_http_create_request

#if (NGX_HTTP_SPDY)
    ngx_http_spdy_stream_t           *spdy_stream;
#endif

    #if (NGX_HTTP_V2)
        /* 赋值见ngx_http_v2_create_stream */
        ngx_http_v2_stream_t             *stream;
    #endif


    ngx_http_log_handler_pt           log_handler;
    //在这个请求中如果打开了某些资源，并需要在请求结束时释放，那么都需要在把定义的释放资源方法添加到cleanup成员中
    /* 
    如果没有需要清理的资源，则cleanup为空指针，否则HTTP模块可以向cleanup中以单链表的形式无限制地添加ngx_http_cleanup_t结构体，
    用以在请求结束时释放资源 */
    ngx_http_cleanup_t               *cleanup;
    //默认值r->subrequests = NGX_HTTP_MAX_SUBREQUESTS + 1;见ngx_http_create_request
    unsigned                          subrequests:8; //该r最多还可以处理多少个子请求  

/*
在阅读HTTP反向代理模块(ngx_http_proxy_module)源代码时，会发现它并没有调用r->main->count++，其中proxy模块是这样启动upstream机制的：
ngx_http_read_client_request_body(r，ngx_http_upstream_init);，这表示读取完用户请求的HTTP包体后才会调用ngx_http_upstream_init方法
启动upstream机制。由于ngx_http_read_client_request_body的第一行有效语句是r->maln->count++，所以HTTP反向代理模块不能
再次在其代码中执行r->main->count++。

这个过程看起来似乎让人困惑。为什么有时需要把引用计数加1，有时却不需要呢？因为ngx_http_read- client_request_body读取请求包体是
一个异步操作（需要epoll多次调度方能完成的可称其为异步操作），ngx_http_upstream_init方法启用upstream机制也是一个异步操作，因此，
从理论上来说，每执行一次异步操作应该把引用计数加1，而异步操作结束时应该调用ngx_http_finalize_request方法把引用计数减1。另外，
ngx_http_read_client_request_body方法内是加过引用计数的，而ngx_http_upstream_init方法内却没有加过引用计数（或许Nginx将来会修改
这个问题）。在HTTP反向代理模块中，它的ngx_http_proxy_handler方法中用“ngx_http_read- client_request_body(r，ngx_http_upstream_init);”
语句同时启动了两个异步操作，注意，这行语句中只加了一次引用计数。执行这行语句的ngx_http_proxy_handler方法返回时只调用
ngx_http_finalize_request方法一次，这是正确的。对于mytest模块也一样，务必要保证对引用计数的增加和减少是配对进行的。
*/
/*
表示当前请求的引用次数。例如，在使用subrequest功能时，依附在这个请求上的子请求数目会返回到count上，每增加一个子请求，count数就要加1。
其中任何一个子请求派生出新的子请求时，对应的原始请求（main指针指向的请求）的count值都要加1。又如，当我们接收HTTP包体时，由于这也是
一个异步调用，所以count上也需要加1，这样在结束请求时，就不会在count引用计数未清零时销毁请求
*/
    unsigned                          count:8; //应用计数   ngx_http_close_request中-1
    /* 如果AIO上下文中还在处理这个请求，blocked必然是大于0的，这时ngx_http_close_request方法不能结束请求 
        ngx_http_copy_aio_handler会自增，当内核把数据发送出去后会在ngx_http_copy_aio_event_handler自剪
     */
    unsigned                          blocked:8; //阻塞标志位，目前仅由aio使用  为0，表示没有HTTP模块还需要处理请求
    //ngx_http_copy_aio_handler handler ngx_http_copy_aio_event_handler执行后，会置回到0   
    //ngx_http_copy_thread_handler ngx_http_copy_thread_event_handler置0
    //ngx_http_cache_thread_handler置1， ngx_http_cache_thread_event_handler置0
    //ngx_http_file_cache_aio_read中置1，
    unsigned                          aio:1;  //标志位，为1时表示当前请求正在使用异步文件IO

    unsigned                          http_state:4; //赋值见ngx_http_state_e中的成员

    /* URI with "/." and on Win32 with "//" */
    unsigned                          complex_uri:1;

    /* URI with "%" */
    unsigned                          quoted_uri:1;

    /* URI with "+" */
    unsigned                          plus_in_uri:1;

    /* URI with " " */
    unsigned                          space_in_uri:1; //uri中是否带有空格
    //头部帧内容部分header合法性检查，见ngx_http_v2_validate_header
    unsigned                          invalid_header:1; //头部行解析不正确，见ngx_http_parse_header_line

    unsigned                          add_uri_to_alias:1;
    unsigned                          valid_location:1; //ngx_http_handler中置1
    //如果有rewrite 内部重定向 uri带有args等会直接置0，否则如果uri中有空格会置1
    unsigned                          valid_unparsed_uri:1;//r->valid_unparsed_uri = r->space_in_uri ? 0 : 1;

    /*
    将uri_changed设置为0后，也就标志说URL没有变化，那么，在ngx_http_core_post_rewrite_phase中就不会执行里面的if语句，也就不会
    再次走到find config的过程了，而是继续处理后面的。不然正常情况，rewrite成功后是会重新来一次的，相当于一个全新的请求。
     */ // 例如rewrite   ^.*$ www.galaxywind.com last;就会多次执行rewrite       ngx_http_script_regex_start_code中置1
    unsigned                          uri_changed:1; //标志位，为1时表示URL发生过rewrite重写  只要不是rewrite xxx bbb sss;aaa不是break结束都会置1
    //表示使用rewrite重写URL的次数。因为目前最多可以更改10次，所以uri_changes初始化为11，而每重写URL -次就把uri_changes减1，
    //一旦uri_changes等于0，则向用户返回失败
    unsigned                          uri_changes:4; //NGX_HTTP_MAX_URI_CHANGES + 1;

    unsigned                          request_body_in_single_buf:1;//client_body_in_single_buffer on | off;设置
    //置1包体需要存入临时文件中  如果request_body_no_buffering为1表示不用缓存包体，那么request_body_in_file_only也为0，因为不用缓存包体，那么就不用写到临时文件中
    /*注意:如果每次开辟的client_body_buffer_size空间都存储满了还没有读取到完整的包体，则还是会把之前读满了的buf中的内容拷贝到临时文件，参考
        ngx_http_do_read_client_request_body -> ngx_http_request_body_filter和ngx_http_read_client_request_body -> ngx_http_request_body_filter
     */
    unsigned                          request_body_in_file_only:1; //"client_body_in_file_only on |clean"设置 和request_body_no_buffering是互斥的
    unsigned                          request_body_in_persistent_file:1; //"client_body_in_file_only on"设置
    unsigned                          request_body_in_clean_file:1;//"client_body_in_file_only clean"设置
    unsigned                          request_body_file_group_access:1; //是否有组权限，如果有一般为0600
    unsigned                          request_body_file_log_level:3;
    //默认是为0的表示需要缓存客户端包体,决定是否需要转发客户端包体到后端，如果request_body_no_buffering为1表示不用缓存包体，那么request_body_in_file_only也为0，因为不用缓存包体，那么就不用写到临时文件中
    unsigned                          request_body_no_buffering:1; //是否缓存HTTP包体，如果不缓存包体，和request_body_in_file_only是互斥的，见ngx_http_read_client_request_body

    /*
        upstream有3种处理上游响应包体的方式，但HTTP模块如何告诉upstream使用哪一种方式处理上游的响应包体呢？
    当请求的ngx_http_request_t结构体中subrequest_in_memory标志位为1时，将采用第1种方式，即upstream不转发响应包体
    到下游，由HTTP模块实现的input_filter方法处理包体；当subrequest_in_memory为0时，upstream会转发响应包体。当ngx_http_upstream_conf_t
    配置结构体中的buffering标志位为1时，将开启更多的内存和磁盘文件用于缓存上游的响应包体，这意味上游网速更快；当buffering
    为0时，将使用固定大小的缓冲区（就是上面介绍的buffer缓冲区）来转发响应包体。
    */
    unsigned                          subrequest_in_memory:1; //ngx_http_subrequest中赋值 NGX_HTTP_SUBREQUEST_IN_MEMORY
    unsigned                          waited:1; //ngx_http_subrequest中赋值 NGX_HTTP_SUBREQUEST_WAITED

#if (NGX_HTTP_CACHE)
    unsigned                          cached:1;//如果客户端请求过来有读到缓存文件，则置1，见ngx_http_file_cache_read  ngx_http_upstream_cache_send
#endif

#if (NGX_HTTP_GZIP)
    unsigned                          gzip_tested:1;
    unsigned                          gzip_ok:1;
    unsigned                          gzip_vary:1;
#endif

    unsigned                          proxy:1;
    unsigned                          bypass_cache:1;
    unsigned                          no_cache:1;

    /*
     * instead of using the request context data in
     * ngx_http_limit_conn_module and ngx_http_limit_req_module
     * we use the single bits in the request structure
     */
    unsigned                          limit_conn_set:1;
    unsigned                          limit_req_set:1;

#if 0
    unsigned                          cacheable:1;
#endif

    unsigned                          pipeline:1;
    //如果后端发送过来的头部行中不带有Content-length:xxx 这种情况1.1版本HTTP直接设置chunked为1， 见ngx_http_chunked_header_filter
    //如果后端带有Transfer-Encoding: chunked会置1
    unsigned                          chunked:1; //chunk编码方式组包实际组包过程参考ngx_http_chunked_body_filter
    //当下游的r->method == NGX_HTTP_HEAD请求方法只请求头部行，则会在ngx_http_header_filter中置1
    //HTTP2头部帧发送在ngx_http_v2_header_filter中置1
    unsigned                          header_only:1; //表示是否只有行、头部，没有包体  ngx_http_header_filter中置1
    //在1.0以上版本默认是长连接，1.0以上版本默认置1，如果在请求头里面没有设置连接方式，见ngx_http_handler
    //标志位，为1时表示当前请求是keepalive请求  1长连接   0短连接  长连接时间通过请求头部的Keep-Alive:设置，参考ngx_http_headers_in_t
    unsigned                          keepalive:1;  //赋值见ngx_http_handler
//延迟关闭标志位，为1时表示需要延迟关闭。例如，在接收完HTTP头部时如果发现包体存在，该标志位会设为1，而放弃接收包体时则会设为o
    unsigned                          lingering_close:1; 
    //如果discard_body为1，则证明曾经执行过丢弃包体的方法，现在包体正在被丢弃中，见ngx_http_read_client_request_body
    unsigned                          discard_body:1;//标志住，为1时表示正在丢弃HTTP请求中的包体
    unsigned                          reading_body:1; //标记包体还没有读完，需要继续读取包体，见ngx_http_read_client_request_body

    /* 在这一步骤中，把phase_handler序号设为server_rewrite_index，这意味着无论之前执行到哪一个阶段，马上都要重新从NGX_HTTP_SERVER_REWRITE_PHASE
阶段开始再次执行，这是Nginx的请求可以反复rewrite重定向的基础。见ngx_http_handler */ 
//ngx_http_internal_redirect置1    创建子请求的时候，子请求也要置1，见ngx_http_subrequest，所有子请求需要做重定向
//内部重定向是从NGX_HTTP_SERVER_REWRITE_PHASE处继续执行(ngx_http_internal_redirect)，而重新rewrite是从NGX_HTTP_FIND_CONFIG_PHASE处执行(ngx_http_core_post_rewrite_phase)
    unsigned                          internal:1;//t标志位，为1时表示请求的当前状态是在做内部跳转， 
    unsigned                          error_page:1; //默认0，在ngx_http_special_response_handler中可能置1
    unsigned                          filter_finalize:1;
    unsigned                          post_action:1;//ngx_http_post_action中置1 默认为0，除非post_action XXX配置
    unsigned                          request_complete:1;
    unsigned                          request_output:1;//表示有数据需要往客户端发送，ngx_http_copy_filter中置1
    //为I时表示发送给客户端的HTTP响应头部已经发送。在调用ngx_http_send_header方法后，若已经成功地启动响应头部发送流程，
    //该标志位就会置为1，用来防止反复地发送头部
    unsigned                          header_sent:1;
    unsigned                          expect_tested:1;
    unsigned                          root_tested:1;
    unsigned                          done:1;
    unsigned                          logged:1;
    /* ngx_http_copy_filter中赋值 */
    unsigned                          buffered:4;//表示缓冲中是否有待发送内容的标志位，参考ngx_http_copy_filter

    unsigned                          main_filter_need_in_memory:1;
    unsigned                          filter_need_in_memory:1;
    unsigned                          filter_need_temporary:1;
    unsigned                          allow_ranges:1;  //支持断点续传 参考3.8.3节
    unsigned                          single_range:1;
    //
    unsigned                          disable_not_modified:1; //r->disable_not_modified = !u->cacheable;因此默认为0

#if (NGX_STAT_STUB)
    unsigned                          stat_reading:1;
    unsigned                          stat_writing:1;
#endif

    /* used to parse HTTP headers */ //状态机解析HTTP时使用state来表示当前的解析状态
    ngx_uint_t                        state; //解析状态，见ngx_http_parse_header_line
    //header_hash为Accept-Language:zh-cn中Accept-Language所有字符串做hash运算的结果
    ngx_uint_t                        header_hash; //头部行中一行所有内容计算ngx_hash的结构，参考ngx_http_parse_header_line
    //lowcase_index为Accept-Language:zh-cn中Accept-Language字符数，也就是15个字节
    ngx_uint_t                        lowcase_index; // 参考ngx_http_parse_header_line
    //存储Accept-Language:zh-cn中的Accept-Language字符串到lowcase_header。如果是AAA_BBB:CCC,则该数组存储的是_BBB
    u_char                            lowcase_header[NGX_HTTP_LC_HEADER_LEN]; //http头部内容，不包括应答行或者请求行，参考ngx_http_parse_header_line

/*
例如:Accept:image/gif.image/jpeg,** 
Accept对应于key，header_name_start header_name_end分别指向这个Accept字符串的头和尾
image/gif.image/jpeg,** 为value部分，header_start header_end分别对应value的头和尾，可以参考mytest_upstream_process_header
*/
    //header_name_start指向Accept-Language:zh-cn中的A处
    u_char                           *header_name_start; //解析到的一行http头部行中的一行的name开始处 //赋值见ngx_http_parse_header_line
    //header_name_start指向Accept-Language:zh-cn中的:处
    u_char                           *header_name_end; //解析到的一行http头部行中的一行的name的尾部 //赋值见ngx_http_parse_header_line
    u_char                           *header_start;//header_start指向Accept-Language:zh-cn中的z字符处
    u_char                           *header_end;//header_end指向Accept-Language:zh-cn中的末尾换行处

    /*
     * a memory that can be reused after parsing a request line
     * via ngx_http_ephemeral_t
     */

//ngx_str_t类型的uri成员指向用户请求中的URI。同理，u_char*类型的uri_start和uri_end也与request_start、request_end的用法相似，唯一不
//同的是，method_end指向方法名的最后一个字符，而uri_end指向URI结束后的下一个地址，也就是最后一个字符的下一个字符地址（HTTP框架的行为），
//这是大部分u_char*类型指针对“xxx_start”和“xxx_end”变量的用法。
    u_char                           *uri_start;//HTTP2的赋值见ngx_http_v2_parse_path
    u_char                           *uri_end;//HTTP2的赋值见ngx_http_v2_parse_path

/*
ngx_str_t类型的extern成员指向用户请求的文件扩展名。例如，在访问“GET /a.txt HTTP/1.1”时，extern的值是{len = 3, data = "txt"}，
而在访问“GET /a HTTP/1.1”时，extern的值为空，也就是{len = 0, data = 0x0}。
uri_ext指针指向的地址与extern.data相同。
*/ //GET /sample.jsp HTTP/1.1 后面的文件如果有.字符，则指向该.后面的jsp字符串，表示文件扩展名
    u_char                           *uri_ext;
    //"GET /aaaaaaaa?bbbb.txt HTTP/1.1"中的bbb.txt字符串头位置处
    u_char                           *args_start;//args_start指向URL参数的起始地址，配合uri_end使用也可以获得URL参数。

    /* 通过request_start和request_end可以获得用户完整的请求行 */
    u_char                           *request_start; //请求行开始处
    u_char                           *request_end;  //请求行结尾处
    u_char                           *method_end;  //GET  POST字符串结尾处

    //HTTP2的赋值见ngx_http_v2_parse_scheme
    u_char                           *schema_start;
    u_char                           *schema_end;
    u_char                           *host_start;
    u_char                           *host_end;
    u_char                           *port_start;
    u_char                           *port_end;

    // HTTP/1.1前面的1代表major，后面的1代表minor
    unsigned                          http_minor:16;
    unsigned                          http_major:16;
};

```

### 3.6.3 获取HTTP头部

```c
struct ngx_http_request_s {
    ...
    ngx_buf_t *header_in;
    ngx_http_headers_in_t headers_in;
    ...
};
```

```c
src/http/ngx_http_request.h
/*
常用的HTTP头部信息可以通过r->headers_in获取，不常用的HTTP头部则需要遍历r->headers_in.headers来遍历获取
*/

//类型的headers_in则存储已经解析过的HTTP头部。下面介绍ngx_http_headers_in_t结构体中的成员。 ngx_http_request_s请求内容中包含该结构
//参考ngx_http_headers_in，通过该数组中的回调hander来存储解析到的请求行name:value中的value到headers_in的响应成员中，见ngx_http_process_request_headers
typedef struct { 
    /*
    获取HTTP头部时，直接使用r->headers_in的相应成员就可以了。这里举例说明一下如何通过遍历headers链表获取非RFC2616标准的HTTP头部，
    可以先回顾一下ngx_list_t链表和ngx_table_elt_t结构体的用法。headers是一个ngx_list_t链表，它存储着解析过的所有HTTP头部，链表中
    的元素都是ngx_table_elt_t类型。下面尝试在一个用户请求中找到“Rpc-Description”头部，首先判断其值是否为“uploadFile”，再决定
    后续的服务器行为，代码如下。
    ngx_list_part_t *part = &r->headers_in.headers.part;
    ngx_table_elt_t *header = part->elts;
    
    //开始遍历链表
    for (i = 0; ; i++) {
     //判断是否到达链表中当前数组的结尾处
     if (i >= part->nelts) {
      //是否还有下一个链表数组元素
      if (part->next == NULL) {
       break;
      }
    
       part设置为next来访问下一个链表数组；header也指向下一个链表数组的首地址；i设置为0时，表示从头开始遍历新的链表数组
      part = part->next;
      header = part->elts;
      i = 0;
     }
    
     //hash为0时表示不是合法的头部
     if (header[i].hash == 0) {
      continue;
     }
    
     判断当前的头部是否是“Rpc-Description”。如果想要忽略大小写，则应该先用header[i].lowcase_key代替header[i].key.data，然后比较字符串
     if (0 == ngx_strncasecmp(header[i].key.data,
       (u_char*) "Rpc-Description",
       header[i].key.len))
     {
      //判断这个HTTP头部的值是否是“uploadFile”
      if (0 == ngx_strncmp(header[i].value.data,
        "uploadFile",
        header[i].value.len))
      {
       //找到了正确的头部，继续向下执行
      }
     }
    }
    
    对于常见的HTTP头部，直接获取r->headers_in中已经由HTTP框架解析过的成员即可，而对于不常见的HTTP头部，需要遍历r->headers_in.headers链表才能获得。
*/
    /*常用的HTTP头部信息可以通过r->headers_in获取，不常用的HTTP头部则需要遍历r->headers_in.headers来遍历获取*/
    /*所有解析过的HTTP头部都在headers链表中，可以使用遍历链表的方法来获取所有的HTTP头部。注意，这里headers链表的
    每一个元素都是ngx_table_elt_t成员*/ //从ngx_http_headers_in获取变量后存储到该链表中，链表中的成员就是下面的各个ngx_table_elt_t成员
    //HTTP2的相关头部赋值见ngx_http_v2_state_process_header
    ngx_list_t                        headers; //在ngx_http_process_request_line初始化list空间  ngx_http_process_request_headers中存储解析到的请求行value和key

    /*以下每个ngx_table_elt_t成员都是RFC1616规范中定义的HTTP头部， 它们实际都指向headers链表中的相应成员。注意，
    当它们为NULL空指针时，表示没有解析到相应的HTTP头部*/ //server和host指向内容一样，都是头部中携带的host头部
    ngx_table_elt_t                  *host; //http1.0以上必须带上host头部行，见ngx_http_process_request_header
    ngx_table_elt_t                  *connection;

/*
If-Modified-Since:从字面上看, 就是说: 如果从某个时间点算起, 如果文件被修改了. 
    1.如果真的被修改: 那么就开始传输, 服务器返回:200 OK  
    2.如果没有被修改: 那么就无需传输, 服务器返回: 403 Not Modified.
用途:客户端尝试下载最新版本的文件. 比如网页刷新, 加载大图的时候。很明显: 如果从图片下载以后都没有再被修改, 当然就没必要重新下载了!

If-Unmodified-Since: 从字面上看, 意思是: 如果从某个时间点算起, 文件没有被修改.....
    1. 如果没有被修改: 则开始`继续'传送文件: 服务器返回: 200 OK
    2. 如果文件被修改: 则不传输, 服务器返回: 412 Precondition failed (预处理错误)
用途:断点续传(一般会指定Range参数). 要想断点续传, 那么文件就一定不能被修改, 否则就不是同一个文件了

总之一句话: 一个是修改了才下载, 一个是没修改才下载.
*/
    ngx_table_elt_t                  *if_modified_since;
    ngx_table_elt_t                  *if_unmodified_since;

/*
ETags和If-None-Match是一种常用的判断资源是否改变的方法。类似于Last-Modified和HTTP-If-Modified-Since。但是有所不同的是Last-Modified和HTTP-If-Modified-Since只判断资源的最后修改时间，而ETags和If-None-Match可以是资源任何的任何属性。
ETags和If-None-Match的工作原理是在HTTPResponse中添加ETags信息。当客户端再次请求该资源时，将在HTTPRequest中加入If-None-Match信息（ETags的值）。如果服务器验证资源的ETags没有改变（该资源没有改变），将返回一个304状态；否则，服务器将返回200状态，并返回该资源和新的ETags。
*/
    ngx_table_elt_t                  *if_match; //和etag配对
    ngx_table_elt_t                  *if_none_match; //和etag配对
    ngx_table_elt_t                  *user_agent;
    ngx_table_elt_t                  *referer;
    ngx_table_elt_t                  *content_length;
    ngx_table_elt_t                  *content_type;

    ngx_table_elt_t                  *range;
    ngx_table_elt_t                  *if_range;

    ngx_table_elt_t                  *transfer_encoding; //Transfer-Encoding
    ngx_table_elt_t                  *expect;
    //Upgrade 允许服务器指定一种新的协议或者新的协议版本，与响应编码101（切换协议）配合使用。例如：Upgrade: HTTP/2.0 
    ngx_table_elt_t                  *upgrade;

#if (NGX_HTTP_GZIP)
    ngx_table_elt_t                  *accept_encoding;
    ngx_table_elt_t                  *via;
#endif

    ngx_table_elt_t                  *authorization;
    //只有在connection_type == NGX_HTTP_CONNECTION_KEEP_ALIVE的情况下才生效    Connection=keep-alive时才有效
    ngx_table_elt_t                  *keep_alive; //赋值ngx_http_process_header_line

#if (NGX_HTTP_X_FORWARDED_FOR)
    ngx_array_t                       x_forwarded_for;
#endif

#if (NGX_HTTP_REALIP)
    ngx_table_elt_t                  *x_real_ip;
#endif

#if (NGX_HTTP_HEADERS)
    ngx_table_elt_t                  *accept;
    ngx_table_elt_t                  *accept_language;
#endif

#if (NGX_HTTP_DAV)
    ngx_table_elt_t                  *depth;
    ngx_table_elt_t                  *destination;
    ngx_table_elt_t                  *overwrite;
    ngx_table_elt_t                  *date;
#endif

    /* user和passwd是只有ngx_http_auth_basic_module才会用到的成员 */
    ngx_str_t                         user;
    ngx_str_t                         passwd;

/*
 Cookie的实现
Cookie是web server下发给浏览器的任意的一段文本，在后续的http 请求中，浏览器会将cookie带回给Web Server。
同时在浏览器允许脚本执行的情况下，Cookie是可以被JavaScript等脚本设置的。


a. 如何种植Cookie

http请求cookie下发流程
http方式:以访问http://www.webryan.net/index.php为例
Step1.客户端发起http请求到Server

GET /index.php HTTP/1.1
 Host: www.webryan.net
 (这里是省去了User-Agent,Accept等字段)

Step2. 服务器返回http response,其中可以包含Cookie设置

HTTP/1.1 200 OK
 Content-type: text/html
 Set-Cookie: name=value
 Set-Cookie: name2=value2; Expires=Wed, 09 Jun 2021 10:18:14 GMT
 (content of page)

Step3. 后续访问webryan.net的相关页面

GET /spec.html HTTP/1.1
 Host: www.webryan.net
 Cookie: name=value; name2=value2
 Accept: * / *
需要修改cookie的值的话，只需要Set-Cookie: name=newvalue即可，浏览器会用新的值将旧的替换掉。
*/
    /*cookies是以ngx_array_t数组存储的，本章先不介绍这个数据结构，感兴趣的话可以直接跳到7.3节了解ngx_array_t的相关用法*/
    ngx_array_t                       cookies;
    //server和host指向内容一样，都是头部中携带的host头部
    ngx_str_t                         server;//server名称   ngx_http_process_host

    /* 在丢弃包体的时候(见ngx_http_read_discarded_request_body)，headers_in成员里的content_length_n，最初它等于content-length头部，而每丢弃一部分包体，就会在content_length_n变量
    中减去相应的大小。因此，content_length_n表示还需要丢弃的包体长度，这里首先检查请求的content_length_n成员，如果它已经等于0，则表示已经接收到完整的包体 
    */
    //解析完头部行后通过ngx_http_process_request_header来开辟空间从而来存储请求体中的内容，表示请求包体的大小，如果为-1表示请求中不带包体
    off_t                             content_length_n; //根据ngx_table_elt_t *content_length计算出的HTTP包体大小  默认赋值-1
    time_t                            keep_alive_n; //头部行Keep-Alive:所带参数 Connection=keep-alive时才有效  赋值ngx_http_process_request_header

    /*HTTP连接类型，它的取值范围是0、NGX_http_CONNECTION_CLOSE或者NGX_HTTP_CONNECTION_KEEP_ALIVE*/
    unsigned                          connection_type:2; //NGX_HTTP_CONNECTION_KEEP_ALIVE等，表示长连接 短连接

/*以下7个标志位是HTTP框架根据浏览器传来的“useragent”头部，它们可用来判断浏览器的类型，值为1时表示是相应的浏览器发来的请求，值为0时则相反*/
    unsigned                          chunked:1;//Transfer-Encoding:chunked的时候置1
    unsigned                          msie:1; //IE Internet Explorer，简称IE
    unsigned                          msie6:1; //
    unsigned                          opera:1; //Opera 浏览器－免费、快速、安全。来自挪威，风行全球
    unsigned                          gecko:1; //Gecko是一套自由及 开放源代码、以C++编写网页 排版引擎，目前为Mozilla Firefox 网页浏览器以
    unsigned                          chrome:1; //Google Chrome是一款快速、简单且安全的网络浏览器
    unsigned                          safari:1; //Safari（苹果公司研发的网络浏览器）_
    unsigned                          konqueror:1;  //Konqueror v4.8.2. 当前最快速的浏览器之一
} ngx_http_headers_in_t;
```

### 3.6.4 获取HTTP包体



## 3.7 发送响应

在`ngx_http_request_t`中有一个`headers_out`成员，用来设置响应中的HTTP头部。

```c
struct ngx_http_request_s {
    ...
    ngx_http_headers_in_t headers_in;
    ngx_http_headers_out_t headers_out;
    ...
};
```

```c
如果发送的是一个不含有HTTP包体的响应，这时就可以直接结束请求了（例如，在ngx_http_mytest_handler方法中，直接在
ngx_http_send_header方法执行后将其返回值return即可）。

注意　ngx_http_send_header方法会首先调用所有的HTTP过滤模块共同处理headers_out中定义的HTTP响应头部，全部处理完
毕后才会序列化为TCP字符流发送到客户端，
*/
typedef struct { //包含在ngx_http_request_s结构headers_out中,ngx_http_send_header中把HTTP头部发出
    /*常用的HTTP头部信息可以通过r->headers_in获取，不常用的HTTP头部则需要遍历r->headers_in.headers来遍历获取*/ 
    //使用查找方法可以参考headers_in
    //如果连接了后端(例如fastcgi到PHP服务器),里面存储的是后端服务器返回的一行一行的头部行信息,赋值在ngx_http_upstream_process_headers->ngx_http_upstream_copy_header_line
    ngx_list_t                        headers;//待发送的HTTP头部链表，与headers_in中的headers成员类似  

    ngx_uint_t                        status;/*响应中的状态值，如200表示成功。NGX_HTTP_OK */ //真正发送给客户端的头部行组包生效在ngx_http_status_lines
    ngx_str_t                         status_line;//响应的状态行，如“HTTP/1.1 201 CREATED”

/*
      HTTP 头部解释
     
     1. Accept：告诉WEB服务器自己接受什么介质类型，* / * 表示任何类型，type/ * 表示该类型下的所有子类型，type/sub-type。
      
     2. Accept-Charset：   浏览器申明自己接收的字符集
        Accept-Encoding：  浏览器申明自己接收的编码方法，通常指定压缩方法，是否支持压缩，支持什么压缩方法  （gzip，deflate）
        Accept-Language：：浏览器申明自己接收的语言语言跟字符集的区别：中文是语言，中文有多种字符集，比如big5，gb2312，gbk等等。
      
     3. Accept-Ranges：WEB服务器表明自己是否接受获取其某个实体的一部分（比如文件的一部分）的请求。bytes：表示接受，none：表示不接受。
      
     4. Age：当代理服务器用自己缓存的实体去响应请求时，用该头部表明该实体从产生到现在经过多长时间了。
      
     5. Authorization：当客户端接收到来自WEB服务器的 WWW-Authenticate 响应时，该头部来回应自己的身份验证信息给WEB服务器。
      
     6. Cache-Control：请求：no-cache（不要缓存的实体，要求现在从WEB服务器去取）
                              max-age：（只接受 Age 值小于 max-age 值，并且没有过期的对象）
                              max-stale：（可以接受过去的对象，但是过期时间必须小于 
                                                 max-stale 值）
                              min-fresh：（接受其新鲜生命期大于其当前 Age 跟 min-fresh 值之和的缓存对象）
                       响应：public(可以用 Cached 内容回应任何用户)
                               private（只能用缓存内容回应先前请求该内容的那个用户）
                               no-cache（可以缓存，但是只有在跟WEB服务器验证了其有效后，才能返回给客户端）
                               max-age：（本响应包含的对象的过期时间）
                               ALL:  no-store（不允许缓存）
      
     7. Connection：请求：close（告诉WEB服务器或者代理服务器，在完成本次请求的响应
                                                       后，断开连接，不要等待本次连接的后续请求了）。
                                      keepalive（告诉WEB服务器或者代理服务器，在完成本次请求的
                                                              响应后，保持连接，等待本次连接的后续请求）。
                            响应：close（连接已经关闭）。
                                      keepalive（连接保持着，在等待本次连接的后续请求）。
        Keep-Alive：如果浏览器请求保持连接，则该头部表明希望 WEB 服务器保持
                           连接多长时间（秒）。
                           例如：Keep-Alive：300
      
     8. Content-Encoding：WEB服务器表明自己使用了什么压缩方法（gzip，deflate）压缩响应中的对象。 
                                      例如：Content-Encoding：gzip                   
        Content-Language：WEB 服务器告诉浏览器自己响应的对象的语言。
        Content-Length：    WEB 服务器告诉浏览器自己响应的对象的长度。
                                     例如：Content-Length: 26012
        Content-Range：    WEB 服务器表明该响应包含的部分对象为整个对象的哪个部分。
                                     例如：Content-Range: bytes 21010-47021/47022
        Content-Type：      WEB 服务器告诉浏览器自己响应的对象的类型。
                                     例如：Content-Type：application/xml
      
     9. ETag：就是一个对象（比如URL）的标志值，就一个对象而言，比如一个 html 文件，
                   如果被修改了，其 Etag 也会别修改， 所以，ETag 的作用跟 Last-Modified 的
                   作用差不多，主要供 WEB 服务器 判断一个对象是否改变了。
                   比如前一次请求某个 html 文件时，获得了其 ETag，当这次又请求这个文件时， 
                   浏览器就会把先前获得的 ETag 值发送给  WEB 服务器，然后 WEB 服务器
                   会把这个 ETag 跟该文件的当前 ETag 进行对比，然后就知道这个文件
                   有没有改变了。
              
     10. Expired：WEB服务器表明该实体将在什么时候过期，对于过期了的对象，只有在
                  跟WEB服务器验证了其有效性后，才能用来响应客户请求。是 HTTP/1.0 的头部。
                  例如：Expires：Sat, 23 May 2009 10:02:12 GMT
      
     11. Host：客户端指定自己想访问的WEB服务器的域名/IP 地址和端口号。
                     例如：Host：rss.sina.com.cn
      
     12. If-Match：如果对象的 ETag 没有改变，其实也就意味著对象没有改变，才执行请求的动作。
         If-None-Match：如果对象的 ETag 改变了，其实也就意味著对象也改变了，才执行请求的动作。
      
     13. If-Modified-Since：如果请求的对象在该头部指定的时间之后修改了，才执行请求
                            的动作（比如返回对象），否则返回代码304，告诉浏览器该对象没有修改。
                            例如：If-Modified-Since：Thu, 10 Apr 2008 09:14:42 GMT
         If-Unmodified-Since：如果请求的对象在该头部指定的时间之后没修改过，才执行请求的动作（比如返回对象）。
      
     14. If-Range：浏览器告诉 WEB 服务器，如果我请求的对象没有改变，就把我缺少的部分
                    给我，如果对象改变了，就把整个对象给我。 浏览器通过发送请求对象的 
                    ETag 或者 自己所知道的最后修改时间给 WEB 服务器，让其判断对象是否
                    改变了。总是跟 Range 头部一起使用。
      
     15. Last-Modified：WEB 服务器认为对象的最后修改时间，比如文件的最后修改时间，
                        动态页面的最后产生时间等等。例如：Last-Modified：Tue, 06 May 2008 02:42:43 GMT
      
     16. Location：WEB 服务器告诉浏览器，试图访问的对象已经被移到别的位置了，到该头部指定的位置去取。
                             例如：Location：http://i0.sinaimg.cn/dy/deco/2008/0528/sinahome_0803_ws_005_text_0.gif
      
     17. Pramga：主要使用 Pramga: no-cache，相当于 Cache-Control： no-cache。例如：Pragma：no-cache
      
     18. Proxy-Authenticate： 代理服务器响应浏览器，要求其提供代理身份验证信息。
           Proxy-Authorization：浏览器响应代理服务器的身份验证请求，提供自己的身份信息。
      
     19. Range：浏览器（比如 Flashget 多线程下载时）告诉 WEB 服务器自己想取对象的哪部分。
                         例如：Range: bytes=1173546-
      
     20. Referer：浏览器向 WEB 服务器表明自己是从哪个 网页/URL 获得/点击 当前请求中的网址/URL。
                        例如：Referer：http://www.sina.com/
      
     21. Server: WEB 服务器表明自己是什么软件及版本等信息。例如：Server：Apache/2.0.61 (Unix)
      
     22. User-Agent: 浏览器表明自己的身份（是哪种浏览器）。
                             例如：User-Agent：Mozilla/5.0 (Windows; U; Windows NT 5.1; zh-CN;   
                                       rv:1.8.1.14) Gecko/20080404 Firefox/2.0.0.14
      
     23. Transfer-Encoding: WEB 服务器表明自己对本响应消息体（不是消息体里面的对象）作了怎样的编码，比如是否分块（chunked）。
                            例如：Transfer-Encoding: chunked
      
     24. Vary: WEB服务器用该头部的内容告诉 Cache 服务器，在什么条件下才能用本响应
                      所返回的对象响应后续的请求。
                      假如源WEB服务器在接到第一个请求消息时，其响应消息的头部为：
                      Content-Encoding: gzip; Vary: Content-Encoding  那么 Cache 服务器会分析后续
                      请求消息的头部，检查其 Accept-Encoding，是否跟先前响应的 Vary 头部值
                      一致，即是否使用相同的内容编码方法，这样就可以防止 Cache 服务器用自己
                      Cache 里面压缩后的实体响应给不具备解压能力的浏览器。
                      例如：Vary：Accept-Encoding
      
     25. Via： 列出从客户端到 OCS 或者相反方向的响应经过了哪些代理服务器，他们用
             什么协议（和版本）发送的请求。当客户端请求到达第一个代理服务器时，该服务器会在自己发出的请求里面
             添加 Via 头部，并填上自己的相关信息，当下一个代理服务器 收到第一个代理服务器的请求时，会在自己发
             出的请求里面复制前一个代理服务器的请求的Via头部，并把自己的相关信息加到后面， 以此类推，当 OCS 
             收到最后一个代理服务器的请求时，检查 Via 头部，就知道该请求所经过的路由。
             例如：Via：1.0 236-81.D07071953.sina.com.cn:80 (squid/2.6.STABLE13)

    */
    /*以下每个ngx_table_elt_t成员都是RFC1616规范中定义的HTTP头部， 它们实际都指向headers链表中的相应成员。注意，
    当它们为NULL空指针时，表示没有解析到相应的HTTP头部*/

/*以下成员（包括ngx_table_elt_t）都是RFC1616规范中定义的HTTP头部，设置后，ngx_http_header_filter_module过滤模块可以把它们加到待发送的网络包中*/
    ngx_table_elt_t                  *server;
    ngx_table_elt_t                  *date;
    ngx_table_elt_t                  *content_length;
    ngx_table_elt_t                  *content_encoding;
    /* rewrite ^(.*)$ http://$1.mp4 break; 如果uri为http://10.135.0.1/aaa,则location中存储的是aaa.mp4 */
    //反向代理后端应答头中带有location:xxx也会进行重定向，见ngx_http_upstream_rewrite_location
    ngx_table_elt_t                  *location; //存储redirect时候，转换后的字符串值，见ngx_http_script_regex_end_code
    ngx_table_elt_t                  *refresh;
    ngx_table_elt_t                  *last_modified;
    ngx_table_elt_t                  *content_range;
    ngx_table_elt_t                  *accept_ranges;//Accept-Ranges: bytes  应该是content_length的单位
    ngx_table_elt_t                  *www_authenticate;
    //expires促发条件是expires配置，见ngx_http_set_expires ，头部行在ngx_http_set_expires进行创建空间以及头部行组装
    ngx_table_elt_t                  *expires;//expires xx配置存储函数为ngx_http_headers_expires，真正组包生效函数为ngx_http_set_expires
    /*
     ETag是一个可以与Web资源关联的记号（token）。典型的Web资源可以一个Web页，但也可能是JSON或XML文档。服务器单独负责判断记号是什么
     及其含义，并在HTTP响应头中将其传送到客户端，以下是服务器端返回的格式：ETag:"50b1c1d4f775c61:df3"客户端的查询更新格式是这样
     的：If-None-Match : W / "50b1c1d4f775c61:df3"如果ETag没改变，则返回状态304然后不返回，这也和Last-Modified一样。测试Etag主要
     在断点下载时比较有用。 "etag:XXX" ETag值的变更说明资源状态已经被修改

     
     Etag确定浏览器缓存： Etag的原理是将文件资源编号一个etag值，Response给访问者，访问者再次请求时，带着这个Etag值，与服务端所请求
     的文件的Etag对比，如果不同了就重新发送加载，如果相同，则返回304. HTTP/1.1304 Not Modified
     */ //见ngx_http_set_etag ETag: "569204ba-4e0924 //etag设置见ngx_http_set_etag 如果客户端在第一次请求文件和第二次请求文件这段时间，文件修改了，则etag就变了
    ngx_table_elt_t                  *etag;

    ngx_str_t                        *override_charset;

/*可以调用ngx_http_set_content_type(r)方法帮助我们设置Content-Type头部，这个方法会根据URI中的文件扩展名并对应着mime.type来设置Content-Type值,取值如:image/jpeg*/
    size_t                            content_type_len;
    ngx_str_t                         content_type;//见ngx_http_set_content_type
    ngx_str_t                         charset; //是从content_type中解析出来的，见ngx_http_upstream_copy_content_type
    u_char                           *content_type_lowcase;
    ngx_uint_t                        content_type_hash;
    //cache_control促发条件是expires配置或者add_head cache_control value配置，见ngx_http_set_expires ，头部行在ngx_http_set_expires进行创建空间以及头部行组装
    ngx_array_t                       cache_control;
    /*在这里指定过content_length_n后，不用再次到ngx_table_elt_t *content_length中设置响应长度*/
    off_t                             content_length_n; //这个标示应答体的长度  不包括头部行长度，只包含消息体长度
    time_t                            date_time;

    //实际上该时间是通过ngx_open_and_stat_file->stat获取的文件最近修改的时间，客户端每次请求都会从新通过stat获取，如果客户端第一次请求该文件和第二次请求该文件过程中修改了该文件，则
    //最通过stat信息获取的时间就会和之前通过last_modified_time发送给客户端的时间不一样
    time_t                            last_modified_time; //表示文件最后被修改的实际，可以参考ngx_http_static_handler
} ngx_http_headers_out_t;
```



### 3.7.3 经典的“Hello World"示例



### 3.8.2 清理文件句柄

```c
typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

/*
ngx_pool_cleanup_t与ngx_http_cleanup_pt是不同的，ngx_pool_cleanup_t仅在所用的内存池销毁时才会被调用来清理资源，它何时释放资
源将视所使用的内存池而定，而ngx_http_cleanup_pt是在ngx_http_request_t结构体释放时被调用来释放资源的。


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





```c
typedef struct {//ngx_open_cached_file中创建空间和赋值
    ngx_fd_t              fd;//文件句柄
    u_char               *name; //文件名称
    ngx_log_t            *log;//日志对象
} ngx_pool_cleanup_file_t;
```



### 3.8.3 支持多线程下载和断点续传



# 第4章 配置、error日志和请求上下文

## 4.1 http配置项的使用场景

## 4.2 怎样使用http配置

HTTP模块是怎样获得感兴趣的配置项的。

处理http配置项可以分为以下4个步骤：

- 创建数据结构用于存储配置项对应的参数。
- 设定配置项在nginx.conf中出现时的限制条件与回调方法。
- 实现第2步中的回调方法，或使用Nginx框架中设定的14个回调方法
- 合并不同级别的配置快中出现的同名配置项。

### 4.2.1 分配用于保存配置参数的数据结构

### 4.2.2 设定配置项的解析方式

下面介绍在读取HTTP配置时是如何使用ngx_command_t结构的。

```c
struct ngx_command_s { //所有配置的最初源头在ngx_init_cycle
    ngx_str_t             name;//配置项名称，如"gzip"
    /*配置项类型，type将指定配置项可以出现的位置。例如，出现在server{}或location{}中，以及它可以携带的参数个数*/
    /*
    type决定这个配置项可以在哪些块（如http、server、location、if、upstream块等）
中出现，以及可以携带的参数类型和个数等。
注意，type可以同时取多个值，各值之间用|符号连接，例如，type可以取
值为NGX_TTP_MAIN_CONF | NGX_HTTP_SRV_CONFI | NGX_HTTP_LOC_CONF | NGX_CONF_TAKE。 
    */
    ngx_uint_t            type; //取值可能为NGX_HTTP_LOC_CONF | NGX_CONF_TAKE2等

    //出现了name中指定的配置项后，将会调用set方法处理配置项的参数
    //cf里面存储的是从配置文件里面解析出的内容，conf是最终用来存储解析内容的内存空间，cmd为存到空间的那个地方(使用偏移量来衡量)
    //在ngx_conf_parse解析完参数后，在ngx_conf_handler中执行
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf); //参考上面的图形化信息
    ngx_uint_t            conf;//crate分配内存的时候的偏移量 NGX_HTTP_LOC_CONF_OFFSET NGX_HTTP_SRV_CONF_OFFSET
    /*通常用于使用预设的解析方法解析配置项，这是配置模块的一个优秀设计。它需要与conf配合使用*/
    ngx_uint_t            offset;

    //如果使用Nginx预设的配置项解析方法，就需要根据这些预设方法来决定post的使用方式。表4-4说明了post相对于14个预设方法的用途。
    /*
    参考14个回调方法后面的 
    if (cmd->post) {
        post = cmd->post;
        return post->post_handler(cf, post, fp);
    }
    */
    void                 *post; 
};
```



### 4.2.5 合并配置项

```c
typedef struct{
	...
    void *(*create_loc_conf)(ngx_conf_t *cf);
　　char *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf); //合并location级别的配置项
    ...
};
```



## 4.2 HTTP配置模型

当Nginx检测到http{}这个关键配置项时，HTTP配置模型就启动了，这时会首先建立1个`ngx_http_conf_ctx`结构

```c
typedef struct { //相关空间创建和赋值见ngx_http_block, 该结构是ngx_conf_t->ctx成员。所有的配置所处内存的源头在ngx_cycle_t->conf_ctx,见ngx_init_cycle
/* 指向一个指针数组，数组中的每个成员都是由所有HTTP模块的create_main_conf方法创建的存放全局配置项的结构体，它们存放着解析直属http{}块内的main级别的配置项参数 */
    void        **main_conf;  /* 指针数组，数组中的每个元素指向所有HTTP模块ngx_http_module_t->create_main_conf方法产生的结构体 */

/* 指向一个指针数组，数组中的每个成员都是由所有HTTP模块的create_srv_conf方法创建的与server相关的结构体，它们或存放main级别配置项，
或存放srv级别的server配置项，这与当前的ngx_http_conf_ctx_t是在解析http{}或者server{}块时创建的有关 */
    void        **srv_conf;/* 指针数组，数组中的每个元素指向所有HTTP模块ngx_http_module_t->create->srv->conf方法产生的结构体 */

/*
指向一个指针数组，数组中的每个成员都是由所有HTTP模块的create_loc_conf方法创建的与location相关的结构体，它们可能存放着main、srv、loc级
别的配置项，这与当前的ngx_http_conf_ctx_t是在解析http{}、server{}或者location{}块时创建的有关存放location{}配置项
*/
    void        **loc_conf;/* 指针数组，数组中的每介元素指向所有HTTP模块ngx_http_module_t->create->loc->conf方法产生的结构体 */
} ngx_http_conf_ctx_t; //ctx是content的简称，表示上下文  
//参考:http://tech.uc.cn/?p=300   ngx_http_conf_ctx_t变量的指针ctx存储在ngx_cycle_t的conf_ctx所指向的指针数组，以ngx_http_module的index为下标的数组元素
//http://tech.uc.cn/?p=300参数解析相关数据结构参考
```

### 4.3.1 解析HTTP配置的流程

![1596628676106](https://images.gitee.com/uploads/images/2020/0805/205826_c92905d6_6568745.png)

### 4.3.2 HTTP配置模型的内存布局



### 4.3.3 如何合并配置项

### 4.3.4 预设配置项处理方法的工作原理

## 4.4 error日志的用法

```c
#define ngx_log_error(level, log, ...)                                        \
    if ((log)->log_level >= level) ngx_log_error_core(level, log,__FUNCTION__, __LINE__, __VA_ARGS__)

#define ngx_log_debug(level, log, ...)                                        \
    if ((log)->log_level & level)                                             \
        ngx_log_error_core(NGX_LOG_DEBUG, log,__FUNCTION__, __LINE__, __VA_ARGS__)
```

1. **level参数**

```
┏━━━━━━━━┳━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃    级别名称    ┃  值  ┃    意义                                                            ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                ┃      ┃    最高级别日志，日志的内容不会再写入log参数指定的文件，而是会直接 ┃
┃NGX_LOG_STDERR  ┃    O ┃                                                                    ┃
┃                ┃      ┃将日志输出到标准错误设备，如控制台屏幕                              ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃                ┃      ┃  大于NGX—LOG ALERT级别，而小于或等于NGX LOG EMERG级别的           ┃
┃NGX_LOG:EMERG   ┃  1   ┃                                                                    ┃
┃                ┃      ┃日志都会输出到log参数指定的文件中                                   ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX LOG ALERT   ┃    2 ┃    大干NGX LOG CRIT级别                                            ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃  NGX LOG CRIT  ┃    3 ┃    大干NGX LOG ERR级别                                             ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃  NGX LOG ERR   ┃    4 ┃    大干NGX—LOG WARN级别                                           ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃  NGX LOG WARN  ┃    5 ┃    大于NGX LOG NOTICE级别                                          ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX LOG NOTICE  ┃  6   ┃  大于NGX__ LOG INFO级别                                            ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃  NGX LOG INFO  ┃    7 ┃    大于NGX—LOG DEBUG级别                                          ┃
┣━━━━━━━━╋━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX LOG DEBUG   ┃    8 ┃    调试级别，最低级别日志     




在使用ngx_log_debug宏时，level的崽义完全不同，它表达的意义不再是级别（已经
是DEBUG级别），而是日志类型，因为ngx_log_debug宏记录的日志必须是NGX-LOG—
DEBUG调试级别的，这里的level由各子模块定义。level的取值范围参见表4-7。
表4-7 ngx_log_debug日志接口level参数的取值范围
┏━━━━━━━━━━┳━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃    级别名称        ┃  值    ┃    意义                                        ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_CORE  ┃ Ox010  ┃    Nginx核心模块的调试日志                     ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_ALLOC ┃ Ox020  ┃    Nginx在分配内存时使用的调试日志             ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_MUTEX ┃ Ox040  ┃    Nginx在使用进程锁时使用的调试日志           ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_EVENT ┃ Ox080  ┃    Nginx事件模块的调试日志                     ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_HTTP  ┃ Oxl00  ┃    Nginx http模块的调试日志                    ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_MAIL  ┃ Ox200  ┃    Nginx邮件模块的调试日志                     ┃
┣━━━━━━━━━━╋━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━┫
┃NGX_LOG_DEBUG_MYSQL ┃ Ox400  ┃    表示与MySQL相关的Nginx模块所使用的调试日志  ┃
┗━━━━━━━━━━┻━━━━┻━━━━━━━━━━━━━━━━━━━━━━━━┛
    当HTTP模块调用ngx_log_debug宏记录日志时，传人的level参数是NGX_LOG_DEBUG HTTP，
这时如果1og参数不属于HTTP模块，如使周了event事件模块的log，则
不会输出任何日志。它正是ngx_log_debug拥有level参数的意义所在。
```

2. **log参数**

```c
typedef u_char *(*ngx_log_handler_pt) (ngx_log_t *log, u_char *buf, size_t len);
struct ngx_log_s {  
    //如果设置的log级别为debug，则会在ngx_log_set_levels把level设置为NGX_LOG_DEBUG_ALL
    //赋值见ngx_log_set_levels
    ngx_uint_t           log_level;//日志级别或者日志类型  默认为NGX_LOG_ERR  如果通过error_log  logs/error.log  info;则为设置的等级  比该级别下的日志可以打印
    ngx_open_file_t     *file; //日志文件

    ngx_atomic_uint_t    connection;//连接数，不为O时会输出到日志中

    time_t               disk_full_time;

    /* 记录日志时的回调方法。当handler已经实现（不为NULL），并且不是DEBUG调试级别时，才会调用handler钩子方法 */
    ngx_log_handler_pt   handler; //从连接池获取ngx_connection_t后，c->log->handler = ngx_http_log_error;

    /*
    每个模块都可以自定义data的使用方法。通常，data参数都是在实现了上面的handler回调方法后
    才使用的。例如，HTTP框架就定义了handler方法，并在data中放入了这个请求的上下文信息，这样每次输出日
    志时都会把这个请求URI输出到日志的尾部
    */
    void                *data; //指向ngx_http_log_ctx_t，见ngx_http_init_connection

    ngx_log_writer_pt    writer;
    void                *wdata;

    /*
     * we declare "action" as "char *" because the actions are usually
     * the static strings and in the "u_char *" case we have to override
     * their types all the time
     */
     
    /*
    表示当前的动作。实际上，action与data是一样的，只有在实现了handler回调方法后才会使
    用。例如，HTTP框架就在handler方法中检查action是否为NULL，如果不为NULL，就会在日志后加入“while
    ”+action，以此表示当前日志是在进行什么操作，帮助定位问题
    */
    char                *action;
    //ngx_log_insert插入，在ngx_log_error_core找到对应级别的日志配置进行输出，因为可以配置error_log不同级别的日志存储在不同的日志文件中
    ngx_log_t           *next;
};
```

3. err参数

err参数就是错误码，一般是执行系统调用失败后取得的errno参数。



### 4.5.2 如何使用HTTP上下文

### 4.5.3 HTTP框架如何维护上下文结构



# 第5章 访问第三方服务

`upstream`和`subrequest`的设计目标是完全不同的。

## 5.1 upstream的使用方式

### 5.1.1 ngx_http_upstream_t结构体

```c
/*
ngx_http_upstream_create方法创建ngx_http_upstream_t结构体，其中的成员还需要各个HTTP模块自行设置。
启动upstream机制使用ngx_http_upstream_init方法
*/ //FastCGI memcached  uwsgi  scgi proxy模块的相关配置都放在该结构中
//ngx_http_request_t->upstream 中存取  //upstream资源回收在ngx_http_upstream_finalize_request
struct ngx_http_upstream_s { //该结构中的部分成员是从upstream{}中的相关配置里面(ngx_http_upstream_conf_t)获取的
    //处理读事件的回调方法，每一个阶段都有不同的read event handler
    //注意ngx_http_upstream_t和ngx_http_request_t都有该成员 分别在ngx_http_request_handler和ngx_http_upstream_handler中执行
    //如果在读取后端包体的时候采用buffering方式，则在读取完头部行和部分包体后，会从置为ngx_http_upstream_process_upstream方式读取后端包体数据
    ////buffering方式，非子请求，后端头部信息已经读取完毕了，如果后端还有包体需要发送，则本端通过ngx_http_upstream_process_upstream该方式读取
    //非buffering方式，非子请求，后端头部信息已经读取完毕了，如果后端还有包体需要发送，则本端通过ngx_http_upstream_process_non_buffered_upstream读取
    //如果有子请求，后端头部信息已经读取完毕了，如果后端还有包体需要发送，则本端通过ngx_http_upstream_process_body_in_memory读取
    ngx_http_upstream_handler_pt     read_event_handler; //ngx_http_upstream_process_header  ngx_http_upstream_handler中执行
    //处理写事件的回调方法，每一个阶段都有不同的write event handler  
    //注意ngx_http_upstream_t和ngx_http_request_t都有该成员 分别在ngx_http_request_handler和ngx_http_upstream_handler中执行
    ngx_http_upstream_handler_pt     write_event_handler; //ngx_http_upstream_send_request_handler用户向后端发送包体时，一次发送没完完成，再次出发epoll write的时候调用

    //表示主动向上游服务器发起的连接。 连接的fd保存在peer->connection里面
    ngx_peer_connection_t            peer;//初始赋值见ngx_http_upstream_connect->ngx_event_connect_peer(&u->peer);

    /*
     当向下游客户端转发响应时（ngx_http_request_t结构体中的subrequest_in_memory标志住为0），如果打开了缓存且认为上游网速更快（conf
     配置中的buffering标志位为1），这时会使用pipe成员来转发响应。在使用这种方式转发响应时，必须由HTTP模块在使用upstream机制前构造
     pipe结构体，否则会出现严重的coredump错误
     */ //实际上buffering为1才通过pipe发送包体到客户端浏览器
    ngx_event_pipe_t                *pipe; //ngx_http_fastcgi_handler  ngx_http_proxy_handler中创建空间

    /* request_bufs决定发送什么样的请求给上游服务器，在实现create_request方法时需要设置它 
    request_bufs以链表的方式把ngx_buf_t缓冲区链接起来，它表示所有需要发送到上游服务器的请求内容。
    所以，HTTP模块实现的create_request回调方法就在于构造request_bufg链表
    */ /* 这上面的fastcgi_param参数和客户端请求头key公用一个cl，客户端包体另外占用一个或者多个cl，他们通过next连接在一起，最终前部连接到u->request_bufs
        所有需要发往后端的数据就在u->request_bufs中了，发送的时候从里面取出来即可，参考ngx_http_fastcgi_create_request*/
    /*
    ngx_http_upstream_s->request_bufs的包体来源为ngx_http_upstream_init_request里面的u->request_bufs = r->request_body->bufs;然后在
    ngx_http_fastcgi_create_request中会重新把发往后端的头部信息以及fastcgi_param信息填加到ngx_http_upstream_s->request_bufs中
    */ //向上游发送包体u->request_bufs(ngx_http_fastcgi_create_request),接收客户端的包体在r->request_body
    //发往上游服务器的请求头内容放入该buf    空间分配在ngx_http_proxy_create_request ngx_http_fastcgi_create_request
    ngx_chain_t                     *request_bufs; 

    //定义了向下游发送响应的方式
    ngx_output_chain_ctx_t           output; //输出数据的结构，里面存有要发送的数据，以及发送的output_filter指针
    ngx_chain_writer_ctx_t           writer; //参考ngx_chain_writer，里面会将输出buf一个个连接到这里。 writer赋值给了u->output.filter_ctx，见ngx_http_upstream_init_request
    //调用ngx_output_chain后，要发送的数据都会放在这里，然后发送，然后更新这个链表，指向剩下的还没有调用writev发送的。

    //upstream访问时的所有限制性参数，
    /*
    conf成员，它用于设置upstream模块处理请求时的参数，包括连接、发送、接收的超时时间等。
    事实上，HTTP反向代理模块在nginx.conf文件中提供的配置项大都是用来设置ngx_http_upstream_conf_t结构体中的成员的。
    上面列出的3个超时时间(connect_timeout  send_imeout read_timeout)是必须要设置的，因为它们默认为0，如果不设置将永远无法与上游服务器建立起TCP连接（因为connect timeout值为0）。
    */ //使用upstream机制时的各种配置  例如fastcgi赋值在ngx_http_fastcgi_handler赋值来自于ngx_http_fastcgi_loc_conf_t->upstream
    ngx_http_upstream_conf_t        *conf; 
    
#if (NGX_HTTP_CACHE) //proxy_pache_cache或者fastcgi_path_cache解析的时候赋值，见ngx_http_file_cache_set_slot
    ngx_array_t                     *caches; //u->caches = &ngx_http_proxy_main_conf_t->caches;
#endif

    /*
     HTTP模块在实现process_header方法时，如果希望upstream直接转发响应，就需要把解析出的响应头部适配为HTTP的响应头部，同时需要
     把包头中的信息设置到headers_in结构体中，这样，会把headers_in中设置的头部添加到要发送到下游客户端的响应头部headers_out中
     */
    ngx_http_upstream_headers_in_t   headers_in;  //存放从上游返回的头部信息，

    //通过resolved可以直接指定上游服务器地址．用于解析主机域名  创建和赋值见ngx_http_xxx_eval(例如ngx_http_fastcgi_eval ngx_http_proxy_eval)
    ngx_http_upstream_resolved_t    *resolved; //解析出来的fastcgi_pass   127.0.0.1:9000;后面的字符串内容，可能有变量嘛。

    ngx_buf_t                        from_client;

    /*
    buffer成员存储接收自上游服务器发来的响应内容，由于它会被复用，所以具有下列多种意义：
    a）在使用process_header方法解析上游响应的包头时，buffer中将会保存完整的响应包头：
    b）当下面的buffering成员为1，而且此时upstream是向下游转发上游的包体时，buffer没有意义；
    c）当buffering标志住为0时，buffer缓冲区会被用于反复地接收上游的包体，进而向下游转发；
    d）当upstream并不用于转发上游包体时，buffer会被用于反复接收上游的包体，HTTP模块实现的input_filter方法需要关注它

    接收上游服务器响应包头的缓冲区，在不需要把响应直接转发给客户端，或者buffering标志位为0的情况下转发包体时，接收包体的缓冲
区仍然使用buffer。注意，如果没有自定义input_filter方法处理包体，将会使用buffer存储全部的包体，这时buf fer必须足够大！它的大小
由ngx_http_upstream_conf_t结构体中的buffer_size成员决定
    */ //ngx_http_upstream_process_header中创建空间和赋值，通过该buf接受recv后端数据 //buf大小由xxx_buffer_size(fastcgi_buffer_size proxy_buffer_size memcached_buffer_size)
//读取上游返回的数据的缓冲区，也就是proxy，FCGI返回的数据。这里面有http头部，也可能有body部分。其body部分会跟event_pipe_t的preread_bufs结构对应起来。就是预读的buf，其实是i不小心读到的。
    //该buf本来是接收头部行信息的，但是也可能会把部分或者全部包体(当包体很小的时候)收到该buf中
    ngx_buf_t                        buffer; //从上游服务器接收的内容在该buffer，发往上游的请求内容在request_bufs中
    //表示来自上游服务器的响应包体的长度    proxy包体赋值在ngx_http_proxy_input_filter_init
    off_t                            length; //要发送给客户端的数据大小，还需要读取这么多进来。 

    /*
out_bufs在两种场景下有不同的意义：
①当不需要转发包体，且使用默认的input_filter方法（也就是ngx_http_upstream_non_buffered_filter方法）处理包体时，out bufs将会指向响应包体，
事实上，out bufs链表中会产生多个ngx_buf_t缓冲区，每个缓冲区都指向buffer缓存中的一部分，而这里的一部分就是每次调用recv方法接收到的一段TCP流。
②当需要转发响应包体到下游时（buffering标志位为O，即以下游网速优先），这个链表指向上一次向下游转发响应到现在这段时间内接收自上游的缓存响应
     */
    ngx_chain_t                     *out_bufs;
    /*
    当需要转发响应包体到下游时（buffering标志位为o，即以下游网速优先），它表示上一次向下游转发响应时没有发送完的内容
     */
    ngx_chain_t                     *busy_bufs;//调用了ngx_http_output_filter，并将out_bufs的链表数据移动到这里，待发送完毕后，会移动到free_bufs
    /*
    这个链表将用于回收out_bufs中已经发送给下游的ngx_buf_t结构体，这同样应用在buffering标志位为0即以下游网速优先的场景
     */
    ngx_chain_t                     *free_bufs;//空闲的缓冲区。可以分配

/*
input_filter init与input_filter回调方法
    input_filter_init与input_filter这两个方法都用于处理上游的响应包体，因为处理包体
前HTTP模块可能需要做一些初始化工作。例如，分配一些内存用于存放解析的中间状态
等，这时upstream就提供了input_filter_init方法。而input_filter方法就是实际处理包体的
方法。这两个回调方法都可以选择不予实现，这是因为当这两个方法不实现时，upstream
模块会自动设置它们为预置方法（上文讲过，由于upstream有3种处理包体的方式，所以
upstream模块准备了3对input_filter_init、input_filter方法）。因此，一旦试图重定义mput_
filter init、input_filter方法，就意味着我们对upstream模块的默认实现是不满意的，所以才
要重定义该功能。
    在多数情况下，会在以下场景决定重新实现input_filter方法。
    (1)在转发上游响应到下游的同时，需要做一些特殊处理
    例如，ngx_http_memcached_ module模块会将实际由memcached实现的上游服务器返回
的响应包体，转发到下游的HTTP客户端上。在上述过程中，该模块通过重定义了的input_
filter方法来检测memcached协议下包体的结束，而不是完全、纯粹地透传TCP流。
    (2)当无须在上、下游间转发响应时，并不想等待接收完全部的上游响应后才开始处理
请求
    在不转发响应时，通常会将响应包体存放在内存中解析，如果试图接收到完整的响应后
再来解析，由于响应可能会非常大，这会占用大量内存。而重定义了input_filter方法后，可
以每解析完一部分包体，就释放一些内存。
    重定义input_filter方法必须符合一些规则，如怎样取到刚接收到的包体以及如何稃放缓
冲区使得固定大小的内存缓冲区可以重复使用等。注意，本章的例子并不涉及input_filter方
法，读者可以在第12章中找到input_filter方法的使用方式。
*/
//处理包体前的初始化方法，其中data参数用于传递用户数据结构，它实际上就是下面的input_filter_ctx指针  
//ngx_http_XXX_input_filter_init(如ngx_http_fastcgi_input_filter_init ngx_http_proxy_input_filter_init ngx_http_proxy_input_filter_init)  
    ngx_int_t                      (*input_filter_init)(void *data); 
    
/* 处理包体的方法，其中data参数用于传递用户数据结构，它实际上就是下面的input_filter_ctx指针，而bytes表示本次接收到的包体长度。
返回NGX ERROR时表示处理包体错误，请求需要结束，否则都将继续upstream流程

用来读取后端的数据，非buffering模式。ngx_http_upstream_non_buffered_filter，ngx_http_memcached_filter等。这个函数的调用时机: 
ngx_http_upstream_process_non_buffered_upstream等调用ngx_unix_recv接收到upstream返回的数据后就调用这里进行协议转换，不过目前转换不多。
*/ //buffering后端响应包体使用ngx_event_pipe_t->input_filter  非buffering方式响应后端包体使用ngx_http_upstream_s->input_filter ,在ngx_http_upstream_send_response分叉
    ngx_int_t                      (*input_filter)(void *data, ssize_t bytes); //ngx_http_xxx_non_buffered_filter(如ngx_http_fastcgi_non_buffered_filter ngx_http_proxy_non_buffered_copy_filter)
//用于传递HTTP模块自定义的数据结构，在input_filter_init和input_filter方法被回调时会作为参数传递过去
    void                            *input_filter_ctx;//指向所属的请求等上下文

#if (NGX_HTTP_CACHE)
/*
Ngx_http_fastcgi_module.c (src\http\modules):    u->create_key = ngx_http_fastcgi_create_key;
Ngx_http_proxy_module.c (src\http\modules):    u->create_key = ngx_http_proxy_create_key;
*/ //ngx_http_upstream_cache中执行
    ngx_int_t                      (*create_key)(ngx_http_request_t *r);
#endif
    //构造发往上游服务器的请求内容
    /*
    create_request回调方法
    create_request的回调场景最简单，即它只可能被调用1次（如果不启用upstream的
失败重试机制的话）：
    1)在Nginx主循环（这里的主循环是指ngx_worker_process_cycle方法）中，会定期地调用事件模块，以检查是否有网络事件发生。
    2)事件模块在接收到HTTP请求后会调用HTIP框架来处理。假设接收、解析完HTTP头部后发现应该由mytest模块处理，这时会调用mytest模
    块的ngx_http_mytest_handler来处理。
    4)调用ngx_http_up stream_init方法启动upstream。
    5) upstream模块会去检查文件缓存，如果缓存中已经有合适的响应包，则会直接返回缓存（当然必须是在使用反向代理文件缓存的前提下）。
    为了让读者方便地理解upstream机制，本章将不再提及文件缓存。
    6)回调mytest模块已经实现的create_request回调方法。
    7) mytest模块通过设置r->upstream->request_bufs已经决定好发送什么样的请求到上游服务器。
    8) upstream模块将会检查resolved成员，如果有resolved成员的话，就根据它设置好上游服务器的地址r->upstream->peer成员。
    9)用无阻塞的TCP套接字建立连接。
    10)无论连接是否建立成功，负责建立连接的connect方法都会立刻返回。
    II) ngx_http_upstreamL init返回。
    12) mytest模块的ngx_http_mytest_handler方法返回NGX DONE。
    13)当事件模块处理完这批网络事件后，将控制权交还给Nginx主循环。
    */ //这里定义的mytest_upstream_create_request方法用于创建发送给上游服务器的HTTP请求，upstream模块将会回调它 
    //在ngx_http_upstream_init_request中执行  HTTP模块实现的执行create_request方法用于构造发往上游服务器的请求
    //ngx_http_xxx_create_request(例如ngx_http_fastcgi_create_request)
    ngx_int_t                      (*create_request)(ngx_http_request_t *r);//生成发送到上游服务器的请求缓冲（或者一条缓冲链）

/*
reinit_request可能会被多次回调。它被调用的原因只有一个，就是在第一次试图向上游服务器建立连接时，如果连接由于各种异常原因失败，
那么会根据upstream中conf参数的策略要求再次重连上游服务器，而这时就会调用reinit_request方法了。图5-4描述了典型的reinit_request调用场景。
下面简单地介绍一下图5-4中列出的步骤。
    1) Nginx主循环中会定期地调用事件模块，检查是否有网络事件发生。
    2)事件模块在确定与上游服务器的TCP连接建立成功后，会回调upstream模块的相关方法处理。
    3) upstream棋块这时会把r->upstream->request_sent标志位置为l，表示连接已经建立成功了，现在开始向上游服务器发送请求内容。
    4)发送请求到上游服务器。
    5)发送方法当然是无阻塞的（使用了无阻塞的套接字），会立刻返回。
    6) upstream模块处理第2步中的TCP连接建立成功事件。
    7)事件模块处理完本轮网络事件后，将控制权交还给Nginx主循环。
    8) Nginx主循环重复第1步，调用事件模块检查网络事件。
    9)这时，如果发现与上游服务器建立的TCP连接已经异常断开，那么事件模块会通知upstream模块处理它。
    10)在符合重试次数的前提下，upstream模块会毫不犹豫地再次用无阻塞的套接字试图建立连接。
    11)无论连接是否建立成功都立刻返回。
    12)这时检查r->upstream->request_sent标志位，会发现它已经被置为1了。
    13)如果mytest模块没有实现reinit_request方法，那么是不会调用它的。而如果reinit_request不为NULL空指针，就会回调它。
    14) mytest模块在reinit_request中处理完自己的事情。
    15)处理宪第9步中的TCP连接断开事件，将控制权交还给事件模块。
    16)事件模块处理完本轮网络事件后，交还控制权给Nginx主循环。
*/ //与上游服务器的通信失败后，如果按照重试规则还需要再次向上游服务器发起连接，则会调用reinit_request方法
    //下面的upstream回调指针是各个模块设置的，比如ngx_http_fastcgi_handler里面设置了fcgi的相关回调函数。
    //ngx_http_XXX_reinit_request(ngx_http_fastcgi_reinit_request) //在ngx_http_upstream_reinit中执行
    ngx_int_t                      (*reinit_request)(ngx_http_request_t *r);//在后端服务器被重置的情况下（在create_request被第二次调用之前）被调用

/*
收到上游服务器的响应后就会回调process_header方法。如果process_header返回NGXAGAIN，那么是在告诉upstream还没有收到完整的响应包头，
此时，对子本次upstream请求来说，再次接收到上游服务器发来的TCP流时，还会调用process_header方法处理，直到process_header函数返回
非NGXAGAIN值这一阶段才会停止

process_header回调方法process_header是用于解析上游服务器返回的基于TCP的响应头部的，因此，process_header可能会被多次调用，
它的调用次数与process_header的返回值有关。如图5-5所示，如果process_header返回NGX_AGAIN，这意味着还没有接收到完整的响应头部，
如果再次接收到上游服务器发来的TCP流，还会把它当做头部，仍然调用process_header处理。而在图5-6中，如果process_header返回NGX_OK
（或者其他非NGX_AGAIN的值），那么在这次连接的后续处理中将不会再次调用process_header。
 process header回调场景的序列图
下面简单地介绍一下图5-5中列出的步骤。
    1) Nginx主循环中会定期地调用事件模块，检查是否有网络事件发生。
    2)事件模块接收到上游服务器发来的响应时，会回调upstream模块处理。
    3) upstream模块这时可以从套接字缓冲区中读取到来自上游的TCP流。
    4)读取的响应会存放到r->upstream->buffer指向的内存中。注意：在未解析完响应头部前，若多次接收到字符流，所有接收自上游的
    响应都会完整地存放到r->upstream->buffer缓冲区中。因此，在解析上游响应包头时，如果buffer缓冲区全满却还没有解析到完整的响应
    头部（也就是说，process_header -直在返回NGX_AGAIN），那么请求就会出错。
    5)调用mytest模块实现的process_header方法。
    6) process_header方法实际上就是在解析r->upstream->buffer缓冲区，试图从中取到完整的响应头部（当然，如果上游服务器与Nginx通过HTTP通信，
    就是接收到完整的HTTP头部）。
    7)如果process_header返回NGX AGAIN，那么表示还没有解析到完整的响应头部，下次还会调用process_header处理接收到的上游响应。
    8)调用元阻塞的读取套接字接口。
    9)这时有可能返回套接字缓冲区已经为空。
    10)当第2步中的读取上游响应事件处理完毕后，控制权交还给事件模块。
    11)事件模块处理完本轮网络事件后，交还控制权给Nginx主循环。
*/ //ngx_http_upstream_process_header和ngx_http_upstream_cache_send函数中调用
/*
解析上游服务器返回响应的包头，返回NGX_AGAIN表示包头还没有接收完整，返回NGX_HTTP_UPSTREAM_INVALID_HEADER表示包头不合法，返回
NGX ERROR表示出现错误，返回NGX_OK表示解析到完整的包头
*/ //ngx_http_fastcgi_process_header  ngx_http_proxy_process_status_line->ngx_http_proxy_process_status_line(ngx_http_XXX_process_header) 
//在ngx_http_upstream_process_header中执行
    ngx_int_t                      (*process_header)(ngx_http_request_t *r); //处理上游服务器回复的第一个bit，时常是保存一个指向上游回复负载的指针
    void                           (*abort_request)(ngx_http_request_t *r);//在客户端放弃请求的时候被调用 ngx_http_XXX_abort_request
   
/*
当调用ngx_http_upstream_init启动upstream机制后，在各种原因（无论成功还是失败）导致该请求被销毁前都会调用finalize_request方
法。在finalize_request方法中可以不做任何事情，但必须实现finalize_request方法，否则Nginx会出现空指针调用的严重错误。

当请求结束时，将会回调finalize_request方法，如果我们希望此时释放资源，如打开
的句柄等，．那么可以把这样的代码添加到finalize_request方法中。本例中定义了mytest_
upstream_finalize_request方法，由于我们没有任何需要释放的资源，所以该方法没有完成任
何实际工作，只是因为upstream模块要求必须实现finalize_request回调方法
*/ //销毁upstream请求时调用  ngx_http_XXX_finalize_request  //在ngx_http_upstream_finalize_request中执行  ngx_http_fastcgi_finalize_request
    void                           (*finalize_request)(ngx_http_request_t *r,
                                         ngx_int_t rc); //请求结束时会调用 //在Nginx完成从上游服务器读入回复以后被调用
/*
在重定向URL阶段，如果实现了rewrite_redirect回调方法，那么这时会调用rewrite_redirect。
可以查看upstream模块的ngx_http_upstream_rewrite_location方法。如果upstream模块接收到完整的上游响应头部，
而且由HTTP模块的process_header回调方法将解析出的对应于Location的头部设置到了ngx_http_upstream_t中的headers in成员时，
ngx_http_upstream_process_headers方法将会最终调用rewrite_redirect方法
因此，rewrite_ redirect的使用场景比较少，它主要应用于HTTP反向代理模蛱(ngx_http_proxy_module)。 赋值为ngx_http_proxy_rewrite_redirect
*/ 
//在上游返回的响应出现Location或者Refresh头部表示重定向时，会通迂ngx_http_upstream_process_headers方法调用到可由HTTP模块实现的rewrite redirect方法
    ngx_int_t                      (*rewrite_redirect)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h, size_t prefix);//ngx_http_upstream_rewrite_location中执行
    ngx_int_t                      (*rewrite_cookie)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h);

    ngx_msec_t                       timeout;
    //用于表示上游响应的错误码、包体长度等信息
    ngx_http_upstream_state_t       *state; //从r->upstream_states分配获取，见ngx_http_upstream_init_request

    //当使用cache的时候，ngx_http_upstream_cache中设置
    ngx_str_t                        method; //不使用文件缓存时没有意义 //GET,HEAD,POST
    //schema和uri成员仅在记录日志时会用到，除此以外没有意义
    ngx_str_t                        schema; //就是前面的http,https,mecached://  fastcgt://(ngx_http_fastcgi_handler)等。
    ngx_str_t                        uri;

#if (NGX_HTTP_SSL)
    ngx_str_t                        ssl_name;
#endif
    //目前它仅用于表示是否需要清理资源，相当于一个标志位，实际不会调用到它所指向的方法
    ngx_http_cleanup_pt             *cleanup;
    //是否指定文件缓存路径的标志位 
    //xxx_store(例如scgi_store)  on | off |path   只要不是off,store都为1，赋值见ngx_http_fastcgi_store
//制定了存储前端文件的路径，参数on指定了将使用root和alias指令相同的路径，off禁止存储，此外，参数中可以使用变量使路径名更明确：fastcgi_store /data/www$original_uri;
    unsigned                         store:1; //ngx_http_upstream_init_request赋值
    //后端应答数据在ngx_http_upstream_process_request->ngx_http_file_cache_update中进行缓存  
    //ngx_http_test_predicates用于可以检测xxx_no_cache,从而决定是否需要缓存后端数据 

   /*如果Cache-Control参数值为no-cache、no-store、private中任意一个时，则不缓存...不缓存...  后端携带有"x_accel_expires:0"头  参考http://blog.csdn.net/clh604/article/details/9064641
    部行也可能置0，参考ngx_http_upstream_process_accel_expires，不过可以通过fastcgi_ignore_headers忽略这些头部，从而可以继续缓存*/ 
    //此外，如果没有使用fastcgi_cache_valid proxy_cache_valid 设置生效时间，则默认会把cacheable置0，见ngx_http_upstream_send_response
    unsigned                         cacheable:1; //是否启用文件缓存 参考http://blog.csdn.net/clh604/article/details/9064641
    unsigned                         accel:1;
    unsigned                         ssl:1; //是否基于SSL协议访问上游服务器
#if (NGX_HTTP_CACHE)
    unsigned                         cache_status:3; //NGX_HTTP_CACHE_BYPASS 等
#endif

    /*
    upstream有3种处理上游响应包体的方式，但HTTP模块如何告诉upstream使用哪一种方式处理上游的响应包体呢？
    当请求的ngx_http_request_t结构体中subrequest_in_memory标志位为1时，将采用第1种方式，即upstream不转发响应包体到下游，由HTTP模
        块实现的input_filter方法处理包体；
    当subrequest_in_memory为0时，upstream会转发响应包体。
    当ngx_http_upstream_conf t配置结构体中的buffering标志位为1时，将开启更多的内存和磁盘文件用于缓存上游的响应包体，这意味上游网速更快；
        会先buffer后端FCGI发过来的数据，等达到一定量（比如buffer满）再传送给最终客户端
    当buffering为0时，将使用固定大小的缓冲区（就是上面介绍的buffer缓冲区）来转发响应包体。
    
    在向客户端转发上游服务器的包体时才有用。
    当buffering为1时，表示使用多个缓冲区以及磁盘文件来转发上游的响应包体。
    当Nginx与上游间的网速远大于Nginx与下游客户端间的网速时，让Nginx开辟更多的内存甚至使用磁盘文件来缓存上游的响应包体，
        这是有意义的，它可以减轻上游服务器的并发压力。
    当buffering为0时，表示只使用上面的这一个buffer缓冲区来向下游转发响应包体 从上游接收多少就向下游发送多少，不缓存，这样上游发送速率与下游速率相等
    */ //fastcgi赋值见ngx_http_fastcgi_handler u->buffering = flcf->upstream.buffering; //见xxx_buffering如fastcgi_buffering  是否缓存后端服务器应答回来的包体
    //该参数也可以通过后端返回的头部字段: X-Accel-Buffering:no | yes来设置是否开启，见ngx_http_upstream_process_buffering
    /*
     如果开启缓冲，那么Nginx将尽可能多地读取后端服务器的响应数据，等达到一定量（比如buffer满）再传送给最终客户端。如果关闭，
     那么Nginx对数据的中转就是一个同步的过程，即从后端服务器接收到响应数据就立即将其发送给客户端。

     buffering标志位为1时，将开启更多的内存和磁盘文件用于缓存上游的响应包体，这意味上游网速更快；当buffering
     为0时，将使用固定大小的缓冲区（就是上面介绍的buffer缓冲区）来转发响应包体。
     */ //buffering方式和非buffering方式在函数ngx_http_upstream_send_response分叉
      //见xxx_buffering如fastcgi_buffering  proxy_buffering  是否缓存后端服务器应答回来的包体
    unsigned                         buffering:1; //向下游转发上游的响应包体时，是否开启更大的内存及临时磁盘文件用于缓存来不及发送到下游的响应
    //为1说明本次和后端的连接使用的是缓存cache(keepalive配置)connection的TCP连接，也就是使用的是之前已经和后端建立好的TCP连接ngx_connection_t
    //在缓存和后端的连接的时候使用(也就是是否配置了keepalive con-num配置项)，为1表示使用的是缓存的TCP连接，为0表示新建的和后端的TCP连接，见ngx_http_upstream_free_keepalive_peer
    //此外，在后端服务器交互包体后，如果头部行指定没有包体，则会u->keepalive = !u->headers_in.connection_close;例如ngx_http_proxy_process_header
    unsigned                         keepalive:1;//只有在开启keepalive con-num才有效，释放后端tcp连接判断在ngx_http_upstream_free_keepalive_peer
    unsigned                         upgrade:1; //后端返回//HTTP/1.1 101的时候置1  

/*
request_sent表示是否已经向上游服务器发送了请求，当request_sent为1时，表示upstream机制已经向上游服务器发送了全部或者部分的请求。
事实上，这个标志位更多的是为了使用ngx_output_chain方法发送请求，因为该方法发送请求时会自动把未发送完的request_bufs链表记录下来，
为了防止反复发送重复请求，必须有request_sent标志位记录是否调用过ngx_output_chain方法
*/
    unsigned                         request_sent:1; //ngx_http_upstream_send_request_body中发送请求包体到后端的时候置1
/*
将上游服务器的响应划分为包头和包尾，如果把响应直接转发给客户端，header_sent标志位表示包头是否发送，header_sent为1时表示已经把
包头转发给客户端了。如果不转发响应到客户端，则header_sent没有意义
*/
    unsigned                         header_sent:1; //表示头部已经扔给协议栈了，
};
```

### 5.1.2 设置upstream的限制性参数

### 5.1.3 设置需要访问的第三方服务器地址

```c
//resolved结构体，用来保存上游服务器的地址
typedef struct { //创建空间和赋值见ngx_http_fastcgi_eval
    ngx_str_t                        host; //sockaddr对应的地址字符串,如a.b.c.d
    in_port_t                        port; //端口
    ngx_uint_t                       no_port; /* unsigned no_port:1 */

    ngx_uint_t                       naddrs; //地址个数，
    ngx_addr_t                      *addrs; 

    struct sockaddr                 *sockaddr; //上游服务器的地址
    socklen_t                        socklen; //sizeof(struct sockaddr_in);

    ngx_resolver_ctx_t              *ctx;
} ngx_http_upstream_resolved_t;
```

### 5.1.4 设置回调方法

### 5.1.5 如何启动upstream机制

## 5.2 回调方法的执行场景

