#nginx基础数据结构之ngx_array

nginx既然有自己的内存池，那么不可避免的需要使用配套的数组这一基础数据结构。ngx_array_s源码放在`core/ngx_array.c`下面。

1， 首先看看array的结构体

```c
struct ngx_array_s {
    void        *elts;       // 元素地址
    ngx_uint_t   nelts;      // 元素个数
    size_t       size;       // 单个元素的大小
    ngx_uint_t   nalloc;     // 总共分配的元素个数
    ngx_pool_t  *pool;
};
```

2，基本操作

* ngx_array_create  创建n个size大小的数组
* ngx_array_destroy 销毁数组
* ngx_array_push 如果nalloc == nelts，表示数组已经存满，当前内存池还能放下array的元素，就对nalloc++，否则直接申请目前数组2被大小的内存。最后返回新的元素的位置。
* ngx_array_push_n 返回n个新元素的起始位置
