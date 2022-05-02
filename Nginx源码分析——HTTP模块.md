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
- `{n}`重复n次
- `{n,}`重复n次或更多次
- `{n,m}`重复n到m次