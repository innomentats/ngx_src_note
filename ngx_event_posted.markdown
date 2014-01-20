#ngx_event_posted 

## 插入元素

```c
#define ngx_locked_post_event(ev, queue)                                      \
                                                                              \
    if (ev->prev == NULL) {                                                   \
        ev->next = (ngx_event_t *) *queue;                                    \
        ev->prev = (ngx_event_t **) queue;                                    \
        *queue = ev;                                                          \
                                                                              \
        if (ev->next) {                                                       \
            ev->next->prev = &ev->next;                                       \
        }                                                                     \
                                                                              \
        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0, "post event %p", ev);  \
                                                                              \
    } else  {                                                                 \
        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0,                        \
                       "update posted event %p", ev);                         \
    }
    
#define ngx_post_event(ev, queue)                                             \
                                                                              \
    ngx_mutex_lock(ngx_posted_events_mutex);                                  \
    ngx_locked_post_event(ev, queue);                                         \
    ngx_mutex_unlock(ngx_posted_events_mutex);

```

&emsp; 由于post队列是一个进程中的全局队列，因此，插入之前进行加锁，但是ngx并没有开启多线程模式。
元素每次都插入队头，prev指向的是上一个元素的next指针的地址，next指针指向下一个元素的内存地址。
队首元素的prev指针指向后一个元素。


## 删除元素 
```c
#define ngx_delete_posted_event(ev)                                           \
                                                                              \
    *(ev->prev) = ev->next;                                                   \
                                                                              \
    if (ev->next) {                                                           \
        ev->next->prev = ev->prev;                                            \
    }                                                                         \
                                                                              \
    ev->prev = NULL;   
 ```
    
&emsp; 将上一个元素的next指针指向下一个元素，下一个元素的prev指针指向本元素的prev指针指向的地址。


##问题 ：
  
1, prev 为什么需要二维指针？
> 
* 记录的是上一元素的next指针的地址，能够更加快捷的做队列相关的操作。

    
2,怎么判断一个ngx_event_s元素在队列中？
    
>
* 判断元素prev是否为NULL
