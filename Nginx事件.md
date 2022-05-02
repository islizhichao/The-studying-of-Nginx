读事件

请求建立TCP连接事件 

TCP连接可读事件

TCP连接关闭事件



读事件

TCP连接可写事件





具体抓包事件

Wireshark网络抓包器

第三次握手发送ACK响应时，操作系统才会同志Nginx产生读事件，建立新连接。



epoll

epoll底层使用eventpoll，里面有两个核心数据结构

**rdlist**

**rbr**





阻塞、非阻塞、异步与同步

调用一个方法，可能使当前的进程进入sleep方法。

非阻塞：有你的代码决定是否切换新任务，





Nginx模块

`ngx_modules.c` 中包含一个所有编译进Nginx中的模块

如何定义：

`ngx_module_t`Nginx中每个模块必须说明的结构体，是一个通用的接口。



 





