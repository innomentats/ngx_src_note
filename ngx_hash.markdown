#Nginx基础数据结构之ngx_hash

## 涉及到的结构体

```c
typedef struct {
    void             *value;       // value
    u_short           len;         // name长度
    u_char            name[1];     // key
} ngx_hash_elt_t;


typedef struct {
    ngx_hash_elt_t  **buckets;     // 实际散列表的地址   
    ngx_uint_t        size;        // 桶的个数
} ngx_hash_t;


typedef struct {
    ngx_hash_t        hash;        
    void             *value;       // 在ngx_hash_wildcard_init可以见到使用场景， 低2为可有特殊含义： 00 表示 value是一个数据指针，11，value指向通配符hash表
} ngx_hash_wildcard_t;

// k-v 
typedef struct {
    ngx_str_t         key;              
    ngx_uint_t        key_hash;
    void             *value;
} ngx_hash_key_t;


typedef ngx_uint_t (*ngx_hash_key_pt) (u_char *data, size_t len);


typedef struct {
    ngx_hash_t            hash;
    ngx_hash_wildcard_t  *wc_head;
    ngx_hash_wildcard_t  *wc_tail;
} ngx_hash_combined_t;

```


```c

typedef struct {
    ngx_hash_t       *hash;                // 实际的hash结构体
    ngx_hash_key_pt   key;                 //hash方法

    ngx_uint_t        max_size;            // 最大元素个数
    ngx_uint_t        bucket_size;         // 桶的大小
 
    char             *name;
    ngx_pool_t       *pool;
    ngx_pool_t       *temp_pool;
} ngx_hash_init_t;
```

## 基本操作
```c
#define NGX_HASH_ELT_SIZE(name)                                               \
    (sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))
```
因为
```c 
#define ngx_align(d, a)     (((d) + (a - 1)) & ~(a - 1))
```
可以看出来，a取2的幂，返回>=d的a的倍数，nginx通过ngx_cpuinfo获得系统二级缓存的大小，然后又利用align将分配的内存大小设置为cpu二级缓存读写行`ngx_cacheline_size`的整数倍，一次性性将频繁访问的数据读入内存，减少CPU高级缓存与低级缓存、内存的数据交换。NGX_HASH_ELT_SIZE返回的就是元素一个关键字所占的空间，加上sizeof(void*)是因为一个桶最后会用一个NULL来标记

nginx的hash是只读的，因此在init做了很多恶心的工作。反过来想下，怎么设计一个只读的hash表？ 
* 占用内存不会动态变化，所以希望init的时候分配合理：选择合适的桶的个数和大小
* 查询如何才能最高效？

带着问题我们来看看`ngx_hash_init`.


```c

ngx_int_t
ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
{
    u_char          *elts;
    size_t           len;
    u_short         *test;
    ngx_uint_t       i, n, key, size, start, bucket_size;
    ngx_hash_elt_t  *elt, **buckets;

    //首先保证桶的大小能放下所有的元素
    for (n = 0; n < nelts; n++) {
        if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *))
        {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build the %s, you should "
                          "increase %s_bucket_size: %i",
                          hinit->name, hinit->name, hinit->bucket_size);
            return NGX_ERROR;
        }
    }
    test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
    if (test == NULL) {
        return NGX_ERROR;
    }

    // 实际桶的大小    
    bucket_size = hinit->bucket_size - sizeof(void *);

    //  估计单个桶的大小
    start = nelts / (bucket_size / (2 * sizeof(void *)));
    start = start ? start : 1;

    if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        start = hinit->max_size - 1000;
    }

    for (size = start; size < hinit->max_size; size++) {

        ngx_memzero(test, size * sizeof(u_short));

        for (n = 0; n < nelts; n++) {
            if (names[n].key.data == NULL) {
                continue;
            }
            //一个桶里面所有的元素都按照一个ngx_hash_key_t数组放在一起            
            key = names[n].key_hash % size;
            test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));

        if (test[key] > (u_short) bucket_size) {
                goto next;
            }
        }

        goto found;

    next:

        continue;
    }

    ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                  "could not build the %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size);

    ngx_free(test);

    return NGX_ERROR;

found:

    for (i = 0; i < size; i++) {
        test[i] = sizeof(void *);
    }

    //test存放每个桶的大小
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    }

    len = 0;

    // 计算max_size
    for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }

        test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        len += test[i];
    }

    //初始化实际的内存地址，存放hash表，为什么要多一个 ngx_hash_wildcard_t？多的一个value指针用来干吗？

    if (hinit->hash == NULL) {
        hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        if (hinit->hash == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }

        buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

    } else {
        buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
        if (buckets == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }
    }

    elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    if (elts == NULL) {
        ngx_free(test);
        return NGX_ERROR;
    }

    elts = ngx_align_ptr(elts, ngx_cacheline_size);

    //初始化每个桶的在内存中的实际初始位置  
    for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }

        buckets[i] = (ngx_hash_elt_t *) elts;
        elts += test[i];

    }

    for (i = 0; i < size; i++) {
        test[i] = 0;
    }

    //插入每个元素   
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        elt->value = names[n].value;
        elt->len = (u_short) names[n].key.len;

        ngx_strlow(elt->name, names[n].key.data, names[n].key.len);

        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    }

    //每个桶最后加一个NULL  
    for (i = 0; i < size; i++) {
        if (buckets[i] == NULL) {
            continue;
        }

        elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);

        elt->value = NULL;
    }

    ngx_free(test);

    hinit->hash->buckets = buckets;
    hinit->hash->size = size;

    return NGX_OK;
}
```