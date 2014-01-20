#master进程工作流程

## QA
* work进程是否继承master的环境变量？

## 过程

1, 初始化错误描叙信息： ngx_strerror_init

2, 获得命令行参数，并且设置对应的操作标记：ngx_get_options。如果是
* ngx_show_version 根据ngx_show_help 显示帮组信息，或者根据ngx_show_configure现实编译的配置信息。

3, ngx_max_sockets = -1

4, ngx_time_init 获得当前时间

5,ngx_regex_init ... 初始化pcre库

6，初始化日志ngx_log_init，init_cycle.pool以及init_cycle.log

7， 保存命令行参数，

8， 初始化conf_file,prefix,conf_prefix

9, 初始化系统相关变量，如内存页面大小ngx_pagesize，ngx_cacheline_size,ngx_max_sockets等

10，将继承来的socket放到init_cycle的listenning是数组

11，初始化模块索引

12，对ngx_cycle结构进行初始化，详细看[ngx_cycle](/ngx_cycle.html)这一节

13，检查ngx_signal，如果有信号要处理，直接转向`ngx_signal_process`,详细见[这里](/ngx_signal_process.html)

14,