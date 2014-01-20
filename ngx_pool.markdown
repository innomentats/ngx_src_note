# nginx内存管理器——**ngx_pool**

## 基本的结构体

### nginx 内存通过自身维护的内存池进行管理。ngx将所有的内存块通过链表的形式组织起来。源码在`core/ngx_palloc.c` 

下面是ngx_pool_s 的结构

```c
    struct ngx_pool_s {  
        ngx_pool_data_t       d;        //实际内存地址   
        size_t                max;      //普通内存块中的最大内存
        ngx_pool_t           *current;  //当前内存数据块
        ngx_chain_t          *chain;    //
        ngx_pool_large_t     *large;    //大块内存链表指针
        ngx_pool_cleanup_t   *cleanup;  //内存回收句柄链表
        ngx_log_t            *log;      
    };
  ```

  * ngx_pool_data_t是实际的内存块。
  >
  ```c
    typedef struct {
        u_char               *last;   //当前分配位置
        u_char               *end;    //本块内存的尾部
        ngx_pool_t           *next;   //下一块内存数据地址
        ngx_uint_t            failed; //分配失败的次数
    } ngx_pool_data_t;  
  ```
    ngx会记录本内存块分配失败的次数，这个失败是什么意思呢？ 稍后在`ngx_palloc_block`可以看到.
    
  * ngx_pool_large_t 是管理大块内存的结构体。也是一个单链表结构。
 >
 ```c
   struct ngx_pool_large_s {
       ngx_pool_large_t     *next;    //下一块指针
       void                 *alloc;   //内存地址
   };
```
 
##基本操作介绍

   &emsp;&emsp;总共有14个函数。ngx_alloc和ngx_calloc都是直接使用c的内存分配函数分配一块size大小的内存，但是ngx_calloc会对分配的内存做一次清0: `ngx_memzero`。
   
   &emsp;&emsp;ngx_memalign直接调用调用系统函数memalign(alignment,size),分配一块地址是alignment的倍数，大小为size的内存，其中alignment是2的幂。nginx使用NGX_POOL_ALIGNMENT=16来对齐。
    
   &emsp;&emsp;ngx_create_pool大致过程： 更加传入的参数size，申请一块内存，然后将这块内存的前面sizeof(ngx_pool_t)部分分配给自己:`p->d.last = (u_char *) p + sizeof(ngx_pool_t);`,将size修改为`size - sizeof(ngx_pool_t)`,将current指向自己`p->current = p;`。完毕。
   
   &emsp;&emsp;再看ngx_palloc。它是从当前内存池pool里面释放分配一个size大小的内存。首先判断size是否大于当前普通内存池的最大内存大小：max，如果大于，就直接走ngx_palloc_large，否则，从当前内存池中沿着链表一直走，找一块满足满足size大小的内存，如果找到结尾还没有找到，就得求助于`ngx_palloc_block`,进行申请。
   
   &emsp;&emsp;ngx_palloc_block，申请一块一块跟当前pool大小一样的内存，初始化当前内存块的ngx_pool_data_t。然后从当前内存块开始，将所有的内存块的分配失败次数加1，如果失败次数超过4，将current指针往后移动，最后将新申请的内存块挂到链表尾部。
   
   &emsp;&emsp;ngx_pnalloc跟ngx_palloc的流程是一样的，但是区别在于，在pool中申请内存时是否做地址对齐，ngx_pnalloc直接从last开始，不做对齐，因此分配比较紧凑，牺牲了寻址速度，但减少了内存碎块。
   
   &emsp;&emsp;ngx_palloc_large是分配大块内存。这里需要注意的一点是，分配的大块内存直接放在alloc，首先去大块内存链表去寻找alloc的挂载点，超过三次还没找到，就直接去pool里面申请一个ngx_pool_large_t，用来挂载alloc。
   
   &emsp;&emsp;ngx_pfree进行大块内存的释放。ngx_reset_pool释放所有大块内存，将所有普通内存块的last指针指向本块的开始位置（减去去存放ngx_pool_t的大小哦）。ngx_destroy_pool首先会调用cleanup对所有的数据块进行清理（*<font size="3" color="red">究竟是干什么呢</font>* ？）,释放大块内存，最后释放普通内存块。直接一次2次链表就就搞定。