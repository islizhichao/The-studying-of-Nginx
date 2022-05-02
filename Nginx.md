# Nginx

## Nginx入门

nginx有一个主进程和多个从进程。

主进程的共能是读取、分析配置文件并且管理从进程。

从进程用来实际处理请求。



### 启动、停止、重启配置文件

- `stop` — fast shutdown
- `quit` — graceful shutdown优雅关闭
- `reload` — reloading the configuration file
- `reopen` — reopening the log files

