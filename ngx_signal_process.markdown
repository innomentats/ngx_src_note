#Nginx 创建worker进程

##热身

这里顺便说下信号处理

 1， `ngx_signal_process`
*    解析出进程pid
* 调用ngx_os_signal_process


2, `ngx_os_signal_process`

*  从`ngx_signal_t  signals[] `找到要处理的信号
* 发送信号 `kill(pid, sig->signo)`

----
## 创建worker进程
下面对worker进程的创建进行分析，先看看全局进程ngx_processes的结构
```c
typedef struct {
    // 进程ID
    ngx_pid_t           pid;  
    // 由waitpid系统调用获取到的进程状态          
    int                 status;        
    /*
    这是由socketpair系统调用产生出的用于进程间通信的socket句柄，这一对socket句柄可以互相通信，
    目前用于master 父进程与worker子进程间的通信。
    */
    ngx_socket_t        channel[2];     

    // 子进程的循环执行方法，当父进程调用ngx_spawn_process 生成子进程时使用
    ngx_spawn_proc_pt   proc;          
    /*
    上面的ngx_spawn_proc_pt方法中第二个参数需要传递一个指针，它是可选的。例如worker子进程就不需要，
    而cache manage进程就需要ngx_cache_manager_ctx上下文成员。这时data一般与ngx_spawn_proc_pt方法中第二个参数是等价的。
    */
    void               *data;
    // 进程名称。操作系统中显示的进程名称与name相同
    char               *name;

    // 标志位，为1表示在重新生成子进程
    unsigned            respawn:1;
    // 标志位，为1表示正在生成子进程
    unsigned            just_spawn:1;
    // 标志位，为1表示在进行父，子进程分离
    unsigned            detached:1;
    // 标志位，为1表示进程正在退出
    unsigned            exiting:1;
    // 标志位，为1表示进程已经退出
    unsigned            exited:1;
} ngx_process_t;
```


1, `ngx_master_process_cycle`
* 屏蔽一系列信号`sigprocmask`，防止创建子进程过程被打扰
* 根据core模块的`worker_processes`创建worker个数
* 进入`ngx_start_worker_processes`重新生成子进程(NGX_PROCESS_RESPAWN)

> 
* 找到一个空闲的slot来存放新创建的子进程
* 如果不是热加载，进行通道的建立和属性设置

>>   
* socketpair 创建一对无名的套接字描述符 
* 设置句柄为非阻塞，异步
* 设置句柄的附属进程为当前master进程
* 关闭句柄
* 设置 子进程句柄ngx_channel，ngx_process_slot是当前子进程在ngx_processes中的索引
* fork

>>>
* 对子进程，设置ngx_pid ，调用传递进来的`ngx_spawn_proc_pt   proc; `进行初始化。
* 设置当前子进程的标志位



