# NGINX HTTP模块源码解析

从本节开始，我们将进入HTTP模块实现原理的解析，关于HTTP模块，第一部分我们来了解它是如何解析配置文件nginx.conf的，并如何保存http块、server块和location块的配置数据；其次，当一个配置信息在http块、server块和location块多次出现的时候，如何管理并以哪个为准~~，本文第一小节主要讲解http块中的各个模块数据的存储方式。~~

## http核心配置模块的存储方式

为了让我们对HTTP框架又深刻的认识，本小节以一个贯彻始终的nginx.conf配置范例来说明框架的行为，如下所示：

```c
http {
    mytest_num 1;
    server {
        server_name A;
        listen 127.0.0.1:8000;
        listen 80;
        
        mytest_num 2;
        
        location /L1 {
            mytest_num 3;
            ...
        }
        location /L2 {
            mytest_num 4;
            ...
        }
    }
    server {
        server_name B;
        listen 80;
        listen 8080;
        listen 173.39.160.51:8000;
        
        mytest_num 5;
        
        location /L1 {
            mytest_num 6;
            ...
        }
        
        location /L3 {
            mytest_num 7;
            ...
        }
    }
}
```

配置信息规则如下:

- HTTP框架支持http{}块内拥有多个server{}、location{}配置快的。
- server代表域名，选用哪一个server虚拟主机块是取决于server_name
- location代表URI，选用哪一个location是与用于请求的URI匹配后决定的
- 同一个配置项可以出现在任意的http{}、server{}、location{}等配置块中。

Nginx对ngx_http_module核心模块定义了新的模块类型，这个模块中的ctx上下文使用了不同于核心模块、事件模块的新接口`ngx_http_module_t`。在介绍`ngx_http_module_t`接口之前，首先对不同级别的HTTP配置项做定义：

- 直接属于http{}块内的配置项称为main配置项
- 直接属于server{}块内的配置项称为srv配置项
- 直接属于location{}块内的配置项称为loc配置项

每一个http模块都必须实现`ngx_http_module_t`接口，结构体如下所示

```c
//nginx主配置文件分为4部分，main（全局配置）、server（主机设置）、upstream（负载均衡服务器设）和location（URL匹配特定位置的设置），这四者关系为：server继承main，location继承server，upstream既不会继承其他设置也不会被继承。
//成员中的create一般在解析前执行函数，merge在函数后执行
typedef struct { //注意和ngx_http_conf_ctx_t结构配合        初始化赋值执行，如果为"http{}"中的配置，在ngx_http_block中, ,所有的NGX_HTTP_MODULE模块都在ngx_http_block中执行
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf); //解析配置文件前调用
    //一般用来把对应的模块加入到11个阶段对应的阶段去ngx_http_phases,例如ngx_http_realip_module的ngx_http_realip_init
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf); //完成配置文件的解析后调用  

/*当需要创建数据结构用于存储main级别（直属于http{...}块的配置项）的全局配置项时，可以通过create_main_conf回调方法创建存储全局配置项的结构体*/
    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);//常用于初始化main级别配置项

/*当需要创建数据结构用于存储srv级别（直属于虚拟主机server{...}块的配置项）的配置项时，可以通过实现create_srv_conf回调方法创建存储srv级别配置项的结构体*/
    void       *(*create_srv_conf)(ngx_conf_t *cf);
// merge_srv_conf回调方法主要用于合并main级别和srv级别下的同名配置项
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    /*当需要创建数据结构用于存储loc级别（直属于location{...}块的配置项）的配置项时，可以实现create_loc_conf回调方法*/
    void       *(*create_loc_conf)(ngx_conf_t *cf);

    /*
    typedef struct {
           void * (*create_loc_conf) (ngx_conf_t *cf) ;
          char*(*merge_loc_conf) (ngx_conf_t *cf, void *prev,
    }ngx_http_module_t
        上面这段代码定义了create loc_conf方法，意味着HTTP框架会建立loc级别的配置。
    什么意思呢？就是说，如果没有实现merge_loc_conf方法，也就是在构造ngx_http_module_t
    时将merge_loc_conf设为NULL了，那么在4.1节的例子中server块或者http块内出现的
    配置项都不会生效。如果我们希望在server块或者http块内的配置项也生效，那么可以通过
    merge_loc_conf方法来实现。merge_loc_conf会把所属父配置块的配置项与子配置块的同名
    配置项合并，当然，如何合并取决于具体的merge_loc_conf实现。
        merge_loc_conf有3个参数，第1个参数仍然是ngx_conf_t *cf，提供一些基本的数据
    结构，如内存池、日志等。我们需要关注的是第2、第3个参数，其中第2个参数void *prev
    悬指解析父配置块时生成的结构体，而第3个参数void:-leconf则指出的是保存子配置块的结
    构体。
    */
    // merge_loc_conf回调方法主要用于合并srv级别和loc级别下的同名配置项
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf); //nginx提供了10个预设合并宏，见上面
} ngx_http_module_t; //
```



在每次nginx运行过程中，都会有一个`ngx_cycle_t`用来保存整个运行周期中的上下文，其中有一个属性`conf_ctx`，这个属性用来存储nginx所有模块的一个三维数组（每个元素指向一个模块指针），

```c
struct ngx_cycle_s
{
    //********
    void                  ****conf_ctx; //有多少个模块就会有多少个指向这些模块的指针，见ngx_init_cycle   ngx_max_module
    //**********
};
```



![](images\Snipaste_2021-04-10_22-41-25.png)

其中标注的`http`表示在`ngx_modules`数组中，第7个模块是`ngx_http_module`模块，在这里这个指针被定义为指向`ngx_http_conf_ctx_t`结构体。

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
```

在nginx.conf配置文件中，在http块下配置有server块，而在server快下也配置有location块，此外location块还可以在location块中嵌套定义，而`ngx_http_conf_ctx_t`结构体的作用就是存储所有这些配置对应的结构体数据。==那么问题来了，http{}配置项是main级别，明明有了create_main_conf生成的结构体保存全局配置参数，为什么还要调用create_srv_conf、create_loc_conf方法建立结构体呢？思考下这个问题，下面解答。==


需要明确的一点是，在nginx.conf配置文件中，配置项都是由一个个模块定义的，一个模块可以定义多个配置项，对于这些配置项的解析工作都是由这个模块所定义的方法进行的。但是，一般的，一个模块只会定义一个结构体，这个结构体中的各个属性则对应于该模块所定义的各个配置项的数据，也就是说，通过各个模块所定义的方法，其会将其所定义的配置项对应的配置转换为该模块所定义的结构体。这里所说的结构体就对应于上面的`main_conf`、`srv_conf`和`loc_conf`中的配置。


从上面的定义就可以看出，这三个属性的类型都是指针类型的数组，而数组的长度就对应于模块的个数，准确来讲，是对应于http模块的各个。在解析各个http模块的配置之前，nginx会对各个http模块在当前类型的模块（http模块）中进行相对位置进行标记，每个http模块的相对位置就对应于上面三个属性的数组下标。前面已经讲到，每个http模块都只会有一个配置结构体存储该模块所定义的所有配置数据，而这些配置结构体就是存储在上面的三个数组中的。这样，我们就能够理解了，其实上面的结构体的三个属性，每一个属性的数组都对应了一个http模块的配置结构体。

既然这里每个模块都有一个结构体存储在数组的对应索引位置，那这里为什么需要三个数组呢？比如说，对于`ngx_http_core_module`，其相对位置在http模块是第一个，也就是说`main_conf[0]`、`srv_conf[0]`和`loc_conf[0]`存储的都是`ngx_http_core_module`的配置结构体，为什么需要三个结构体。这里我们需要说明的是，对于每个http模块，其会根据需要将配置项按照可使用范围划分为三类：仅用于http块，可以用于http块和server块，以及可以用于http块、server块和location块。每一类配置项都使用的是一个不同的结构体，比如`ngx_http_core_module`就定义了`ngx_http_core_main_conf_t`用于存储仅用于http块的配置项，定义了`ngx_http_core_srv_conf_t`用于存储用于http块和server块的配置项，定义了`ngx_http_core_loc_conf_t`用于存储用于http块、server块和location块的配置项。对应于上面的数组就是，`main_conf[0]`的结构体类型为`ngx_http_core_main_conf_t`，`srv_conf[0]`的结构体类型为`ngx_http_core_srv_conf_t`，`loc_conf[0]`对应的结构体类型为`ngx_http_core_loc_conf_t`。说到这里，我们就必须要厘清一个问题了，比如，对于某个配置项，其配置在了http块中，但是其类型是可以用于http块、server块和location块的，那么其就会被存储在`loc_conf[0]`中，也就是说，上面的一整个结构体，从目前来看，存储的都是在http块中解析出来的各个配置项的数据。那么nginx是如何标记一个配置项是这三种类型中的哪一种呢？这主要是通过`ngx_command_t`结构体来定义的，如下所示为三个典型的配置：

```c
{
  ngx_string("variables_hash_max_size"),
  NGX_HTTP_MAIN_CONF | NGX_CONF_TAKE1,
  ngx_conf_set_num_slot,
 	NGX_HTTP_MAIN_CONF_OFFSET,
 	offsetof(ngx_http_core_main_conf_t, variables_hash_max_size),
 	NULL
},
{
  ngx_string("listen"),
 	NGX_HTTP_SRV_CONF | NGX_CONF_1MORE,
 	ngx_http_core_listen,
 	NGX_HTTP_SRV_CONF_OFFSET,
 	0,
 	NULL
},
{
  ngx_string("root"),
 	NGX_HTTP_MAIN_CONF | NGX_HTTP_SRV_CONF | NGX_HTTP_LOC_CONF | NGX_HTTP_LIF_CONF
 	  | NGX_CONF_TAKE1,
 	ngx_http_core_root,
 	NGX_HTTP_LOC_CONF_OFFSET,
 	0,
 	NULL
},
```

这里我们以`variables_hash_max_size`、`listen`和`root`三个指令为例，这三个指令都是`ngx_http_core_module`模块定义的配置项，但是它们存储的位置则是完全不同的。我们需要注意的就是每个指令的第四个属性的定义：`NGX_HTTP_MAIN_CONF_OFFSET`、`NGX_HTTP_SRV_CONF_OFFSET`和`NGX_HTTP_LOC_CONF_OFFSET`。这三个类型的定义有两重含义，一个是表示这个配置项是仅用于http块，还是可以用于http块和server块，再或者是可以用于http块、server块和location块；另一重含义是定义了这个配置项在上面讲的`ngx_http_conf_ctx_t`中的偏移量，所谓的偏移量指的就是，在知道`ngx_http_conf_ctx_t`结构体对象的指针地址时，通过这里的偏移量就可以计算出当前配置项所存储的数组。这里我们就需要展示一段代码，即在`ngx_conf_parse()`方法中，其主要是用于解析nginx.conf配置文件的，在解析了某个配置项之后，就会在所有的模块中，找到该配置项的定义，如果找到了配置项，就会尝试获取存储该配置项所对应的结构体，并且会调用该配置项指定的方法进行配置项数据的解析。这里尝试获取该配置项所对应的结构体时，就需要用上上面的偏移量。如下是获取该配置项的方法：

```c
// 查找配置对象，NGX_DIRECT_CONF常量单纯用来指定配置存储区的寻址方法，只用于core模块
if (cmd->type & NGX_DIRECT_CONF) {
  conf = ((void **) cf->ctx)[cf->cycle->modules[i]->index];

  // NGX_MAIN_CONF常量有两重含义，其一是指定指令的使用上下文是main（其实还是指core模块），
  // 其二是指定配置存储区的寻址方法。
} else if (cmd->type & NGX_MAIN_CONF) {
  conf = &(((void **) cf->ctx)[cf->cycle->modules[i]->index]);

  // 除开core模块，其他类型的模块都会使用第三种配置寻址方式，也就是根据cmd->conf的值
  // 从cf->ctx中取出对应的配置。举http模块为例，cf->conf的可选值是NGX_HTTP_MAIN_CONF_OFFSET、
  // NGX_HTTP_SRV_CONF_OFFSET、NGX_HTTP_LOC_CONF_OFFSET，
  // 分别对应“http{}”、“server{}”、“location{}”这三个http配置级别。

  // 这个if判断的作用主要是，cf->ctx的类型是ngx_http_conf_ctx_t，而cmd->conf主要的值可选
  // NGX_HTTP_MAIN_CONF_OFFSET、NGX_HTTP_SRV_CONF_OFFSET、NGX_HTTP_LOC_CONF_OFFSET，
  // 可以看到ngx_http_conf_ctx_t的属性有main_conf、srv_conf和loc_conf，
  // 其实这里就是在计算当前的配置对象是存储在这三个数组中的哪一个数组中，以default_type指令为例，
  // 其ngx_command_t的配置为：
  // {ngx_string("default_type"),
  //     NGX_HTTP_MAIN_CONF | NGX_HTTP_SRV_CONF | NGX_HTTP_LOC_CONF | NGX_CONF_TAKE1,
  //     ngx_conf_set_str_slot,
  //     NGX_HTTP_LOC_CONF_OFFSET,
  //     offsetof(ngx_http_core_loc_conf_t, default_type),
  //     NULL},
  // 可以看到，其conf属性的值为NGX_HTTP_LOC_CONF_OFFSET，则说明其是存储在loc_conf数组中的，
  // 而该数组中的元素类型为ngx_http_core_loc_conf_t，因而可以看到，后面ngx_command_t
  // 中offset属性的值就指定为了offsetof(ngx_http_core_loc_conf_t, default_type)，
  // 这就是在计算default_type属性在ngx_http_core_loc_conf_t结构体中的位置。
  // 通过下面的if判断第一步confp = *(void **) ((char *) cf->ctx + cmd->conf);，就可以
  // 计算出当前所使用的结构体是在main_conf、srv_conf
  // 和loc_conf的哪一个数组中，而通过第二步conf = confp[cf->cycle->modules[i]->ctx_index];
  // 的计算，就可以计算出该结构体在数组中的具体位置，并且获取该结构体数据。
  // 需要注意的是，这种计算方式只适用于http模块的配置项获取，因为只有http模块的配置结构体是
  // ngx_http_conf_ctx_t类型的
} else if (cf->ctx) {
  confp = *(void **) ((char *) cf->ctx + cmd->conf);

  if (confp) {
    conf = confp[cf->cycle->modules[i]->ctx_index];
  }
}
```



从调用`ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)`开始

```c
struct ngx_conf_s {
    char                 *name;
    ngx_array_t          *args;

    ngx_cycle_t          *cycle;
    ngx_pool_t           *pool;
    ngx_pool_t           *temp_pool;
    ngx_conf_file_t      *conf_file;
    ngx_log_t            *log;

    void                 *ctx;
    ngx_uint_t            module_type;
    ngx_uint_t            cmd_type;

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
};
```



```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```



第一步传入参数，初始化`ngx_http_conf_ctx_t`中的参数，并分配好空间

![](images\Snipaste_2021-04-13_17-36-38.png)

![](images\Snipaste_2021-04-13_17-36-02.png)



第二步调用各个http模块的create_main_conf、create_srv_conf、create_loc_conf函数

首先是第一个http模块，`ngx_http_core_module_ctx`

![](images\Snipaste_2021-04-13_17-41-19.png)

其中各个函数指针分别是：

![](images\Snipaste_2021-04-13_17-46-30.png)



###### 调用其`ngx_http_core_create_main_conf`函数，初始化`ngx_http_core_main_conf_t`结构体

![](images\Snipaste_2021-04-13_17-49-58.png)

其中`servers`为指向一个包含`ngx_http_core_srv_conf_t`的动态数组

![](images\Snipaste_2021-04-13_17-52-19.png)



###### 调用其`ngx_http_core_create_srv_conf`函数，初始化`ngx_http_core_srv_conf_t`结构体

![](images\Snipaste_2021-04-13_17-56-58.png)



###### 调用其`ngx_http_core_create_loc_conf`函数，初始化`ngx_http_core_loc_conf_t`结构体

![](images\Snipaste_2021-04-13_18-01-27.png)

###### 调用其ngx_http_core_preconfiguratioin函数，

```c
static ngx_int_t
ngx_http_core_preconfiguration(ngx_conf_t *cf)
{
    return ngx_http_variables_add_core_vars(cf);
}
//ngx_http_variables_add_core_vars把ngx_http_core_variables中的各种变量信息存放到cmcf->variables_keys中
```

![](images\Snipaste_2021-04-13_20-30-14.png)



引用：

[https://cloud.tencent.com/developer/article/1596843]: 
[《深入理解Nginx 模块开发与架构解析》]: 



