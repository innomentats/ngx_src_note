#nginx基础数据结构之ngx_queue

个人感觉ngx的队列是非常有意思的一个数据结构。首先这个队列的结构体
```c
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```
可以看到是由双向链表组成，但是却没有元素节点。这样的结构，跟一般的队列实现不一样，由元素自己去维护自己的队列信息。


1，初始化队列

```c
#define ngx_queue_init(q)  \                                                   
    (q)->prev = q;         \                                                  
    (q)->next = q
```

2,ngx_queue_data(q, type, link)，type是目标结构体，link是type结构体中的ngx_queue_s类型成员，q是当前需要获得的**目标结构体**中ngx_queue_s的实际内存位置，那么要获得目标结构体的位置，
```c

(type*)((uchar*)q - offsetof(type,link))

```


在实际使用场景中，经常看到queue如此打扮出场

```c
ngx_queue_insert_head(h,x)
...
q = ngx_queue_last(queue)
c = ngx_queue_data(q, type, queue);
```
由于元素是头部插入，作者利用LRU思想，取出最后队列尾部的元素，进行回收/移除等操作
