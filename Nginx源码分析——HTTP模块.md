# Nginx源码分析——HTTP模块

## HTTP配置快的嵌套

<img src="images\Snipaste_2022-05-02_11-18-19.png" style="zoom: 50%;" />

在HTTP之前，成为main配置快，比如user、event等配置项

http、server、location这是http模块定义的，在处理一个请求的时候，首先按照域名找到一个server快，然后根据每个url找到具体的location，然后处理每个location下具体的指令。



在配置项中，会出现在http、server、location中冲突的情况

指令在多个配置快内可以出现合并，但具体按指定的方法进行合并，（一般为存储配置项的值，比如root、access_log或gzip）。

而动作类指令（指定行为） 如：rewrite、proxy_pass等。

### 存储值的指令继承规则：向上覆盖

- 子配置项不存在时，直接使用父配置项
- 子配置存在时，直接覆盖父配置快

### HTTP模块合并配置的实现

- 指令在哪个块下生效
- 指令允许出现在哪些块下
- 在server块内生效，从http向server合并指令
- 配置缓存在内存



## Listen指令

只能出现在sever上下文中。

从一个连接建立起，到收到请求，如何处理？

<img src="images\Snipaste_2022-05-02_11-47-10.png" style="zoom:67%;" />

当用户发送一个SYN，内核发送SYN+ACK，然后用户发送一个ACK，进行连接。

内存池分为连接内存池和请求内存池，分配connection_pool_size大小的内存池，当accept一个方法时，会使用ngx_http_init_connection回调方法。

当用户发送一个GET、PUT方法，把内核中的发送data读到nginx内核态。

<img src="images\Snipaste_2022-05-02_11-51-55.png" style="zoom:60%;" />



### 处理HTTP头部的请求





## Nginx中的正则表达式

正则表达式用法（元字符）

- `.`匹配除换行符以外的任意字符
- `\w`匹配字母或数字或下划线或汉字
- `\s`匹配任意的空白符
- `\d`匹配数字
- `\b`匹配单词的开始或结束
- `^`匹配字符串的开始
- `$`匹配字符串的结束

正则表达式用法（重复）

- `*`重复零次或更多次
- `+`重复一次或更多次
- `?`重复零次或一次
- `{n}`重复n次   只出现四次，`\d{4}`
- `{n,}`重复n次或更多次
- `{n,m}`重复n到m次



## 如何找到处理请求的server指令块

一个请求被哪个server处理，靠的是`server_name`

### server_name指令

指令后可以跟多个域名，第一个是主域名

也支持泛域名：仅支持在最前或者最后 例如：server_name *.taohui.tech

还支持正则表达式 例如：server_name www.taohui.tech ~^www\d+\\.taohui\\.tech$

支持用正则表达式创建变量：

```nginx
server{
    server_name ~^(www\.)?(.+)$;
    location/ {root/site/$2;}
}

server{
    server_name ~^(www\.)?(?<domain>.+)$; #命名变量
    location/ {root/site/$domain;}
}
```

------

### server_name匹配顺序

1. 精准匹配  一个完整的域名字符串，与域名顺序无关
2. *在前的泛域名 
3. *在后的泛域名
4. 按文件中的顺序匹配正则表达式域名
5. `default server` 指定第一个，listen端口指定默认



## HTTP请求的11个阶段

|                |                        |                                            |
| :------------: | :--------------------: | :----------------------------------------: |
|   POST_READ    | 读到所有的请求头部之后 |         realip，获取到一些原始的值         |
| SERVER_REWRITE |                        |                  rewrite                   |
|  FIND_CONFIG   |                        |                                            |
|    REWRITE     |                        |                  rewrite                   |
|  POST_REWRITE  |                        |                                            |
|   PREACCESS    |     确认访问权限，     | limit_conn,limit_req根据连接数、速度限制等 |
|     ACCESS     |                        |       auth_basic,access,auth_request       |
|  POST_ACCESS   |                        |                                            |
|   PRECONTENT   |                        |                try_files，                 |
|    CONTENT     |                        |           index,autoindex,concat           |
|      LOG       |                        |               打印access_log               |

所有请求按照这11个阶段顺序依次调用HTTP模块进行请求。

![](images\Snipaste_2022-05-02_16-06-02.png)

### postread阶段

用于发现用户请求的真实ip地址，利于后续处理中根据ip地址限速、限流。

如何拿到真实的用于IP地址

- TCP连接四元组（src ip, src port, dst ip, dst port） 
- HTTP头部X-Formwarded-For 用于传递IP
- HTTP头部X-Real-IP用于传递用户IP
- 网络中存在许多反向代理



### rewrite阶段: return指令

```nginx
return code [text];
return code URL;
return URL;
```

返回状态码：

- Nginx自定义
  - 444： 关闭连接
- HTTP 1.0标准
  - 301： http1.0永久重定向
  - 302：临时重定向，禁止被缓存
- HTTP 1.1标准
  - 303：临时重定向，允许改变方法，禁止被缓存
  - 307：临时重定向，不允许改变方法，禁止被缓存
  - 308：永久重定向，不允许改变方法