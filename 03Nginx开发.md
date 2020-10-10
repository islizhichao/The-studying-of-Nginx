# Nginx开发

## 一个简单的模块

在编写模块时，我们先对这个模块提出三个问题：

- 是否需要在配置文件中进行配置？配置指令是什么？有什么样的参数
- 如果使用Nginx框架？怎样设置访问参数？怎样处理TCP/HTTP请求？
- 如何编译集成进入Nginx中？

而我们的回答是：

- 模块名称是ndg_test_module
- 模块指令是ndg_test on|off，开关模块只能在location中进行定义
- 使用ngx_command_t和相关函数解析配置指令
- 使用ngx_http_module_t定义功能函数，创建配置数据并初始化
- 使用ngx_module_t定义模块
- 不直接处理HTTP请求，只在URL重写阶段执行
- 根据配置指令的on|off决定输出字符串的内容
- 编写config脚本，用"--add-module"静态连接选项集成到Nginx中

**配置解析**

ndg_test_module在Nginx配置文件里的形式是：

```c
location /test{
    ndg_test on;
    ...
}
```

解析这个配置指令需要定义配置数据结构、指令数组和管理函数。

**配置数据结构**

定义一个含有对应信息的结构体，用来存储配置数据

```c
struct NdgTestConf final
{
	ndg_flag_t enabled = ngx_nil;  //保存配置文件的信息
};
```

**配置的解析**

Nginx提供结构ngx_command_t来实现配置指令解析，每条指令对应一个ngx_command_t对象。所有的ngx_command_t对象放在一个数组里，最后需要一个ngx_null_command表示数组定义结束：

```c
/*
commands数组用于定义模块的配置文件参数，每一个数组元素都是ngx_command_t类型，数组的结尾用ngx_null_command表示。Nginx在解析配置
文件中的一个配置项时首先会遍历所有的模块，对于每一个模块而言，即通过遍历commands数组进行，另外，在数组中检查到ngx_null_command时，
会停止使用当前模块解析该配置项。每一个ngx_command_t结构体定义了自己感兴趣的一个配置项：
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

```c
static ngx_command_t ndg_test_cmds[] = 
{
    {
        ngx_string("ndg_tets"),				//指令的名字，也就是配置文件中出现的指令
        NGX_HTTP_LOC_CONF | NGX_CONF_FLAG,	//指令的作用域和类型，指令只能出现在location中，并且参数只能是on|off
        ngx_conf_set_flag_slot,				//解析函数指针,使用Nginx标准函数ngx_conf_set_flag_slot解析指令
        NGX_HTTP_LOC_CONF_OFFSET,			//数据的存储位置，整个数据结构存储在http/location作用域
        offset(NdgTsetConf,enabled),		//数据的具体存储变量，使用宏offsetof获取变量的地址，供解析函数使用
        nullptr
    },
    ngx_null_command
};
```

**创建配置数据**

Nginx要求模块自己分配配置数据结构的内存，所以我们还需要编写一个create( )函数，在配置解析时使用内存池创建对象：

```c
static void* create(ngx_conf_t* cf)
{
    return NgxPool(cf).alloc<NdgTestConf>();
}
```

**处理函数**

函数handler( )先读取配置参数，然后根据参数向控制台输出字符串：

```c
static ngx_int_t handler(ngx_http_request_t *r)
{
    auto cf = reinterpret_cast<NdgTestConf*>(ngx_http_get_module_loc_conf(r,ndg_test_module));
    NgxLogError(r).print("hello c++");
    if(cf->enabled)
    {
        std::cout<< "hello nginx"<<std::endl;
    }
    else
    {
        std::cout<<"hello disabled"<<std::endl;
    }
    return NGX_DECLINED;
}
```

注册处理函数

函数init()将在Nginx配置解析阶段调用，向Nginx框架注册我们自己的处理函数：

```c
static ngx_int_t init(ngx_conf_t* cf)
{
    auto cmcf = reinterpret_cast<ngx_http_core_main_conf_t*>(
    	ngx_http_conf_get_module_main_conf(
        	cf,ngx_http_core_module
        )
    );
    NgxArray<ngx_http_handler_pt> arr(
    	cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers
    );
    arr.push(handler);
    return NGX_OK;
}
```

**模块集成**

Nginx提供两个数据结构ngx_http_module_t和ngx_module_t，用来集成配置指令解析和处理函数。

**集成配置函数**

```c
static ngx_http_module_t ndg_test_ctx = 
{
    nullptr;	//preconfiguration
    init,		//postconfiguration
    nullptr,	//create_main_conf
    nullptr, 	//init_main_conf
    nullptr,	//create_srv_conf
    nullptr,	//init_srv_conf
    create,		//create_loc_conf
    nullptr,	//merge_loc_conf
};
```

**集成配置指令**

ngx_module_t是Nginx真正定义模块的数据结构，它集成ngx_http_module和ngx_command_t数组

```c
ngx_module_t ndg_test_module = 
{
    NGX_MODULE_V1,
    &ndg_test_ctx,
    ndg_test_cmds,
    NGX_HTTP_MODULE,
    nullptr,
    nullptr,
    nullptr,
    nullptr,
    nulltpr,
    nullptr,
    nullptr,
    NGX_MODULE_V1_PADDING
};
```

