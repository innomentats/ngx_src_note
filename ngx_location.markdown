# nginx location设计

## 核心数据结构

### 1， ngx_http_core_loc_conf_s  
&emsp;&emsp; ngx_http_core_loc_conf_s完整的描叙了一个location块的信息，以及匹配这个location的请求的http包的一些资源信息
```c
struct ngx_http_core_loc_conf_s {
    ngx_str_t name ;   //location的名称，就是nginx.conf 里面location后面的表达式
    #if (NGX_PCRE)
    ngx_http_regex_t  *regex;
#endif

    unsigned      noname:1;   /* "if () {}" block or limit_except */
    unsigned      lmt_excpt:1;  /*标记location的method是否有限制*/ 
    unsigned      named:1;    

    unsigned      exact_match:1;
    unsigned      noregex:1;

    unsigned      auto_redirect:1;
#if (NGX_HTTP_GZIP)
    unsigned      gzip_disable_msie6:2;
#if (NGX_HTTP_DEGRADATION)
    unsigned      gzip_disable_degradation:2;
#endif
#endif
    //静态location树
    ngx_http_location_tree_node_t   *static_locations;
#if (NGX_PCRE)
    ngx_http_core_loc_conf_t       **regex_locations;
#endif
    
    void        **loc_conf;  //指向的是ngx_http_conf_ctx_t结构体的loc_conf指针数组,保存当前http块内所有的location

    //记录的就是带method限制的location
    uint32_t      limit_except;
    void        **limit_except_loc_conf;

    ngx_http_handler_pt  handler;

    ...

    /*
    将同一个server块内多个表达location块的 ngx_http_core_loc_conf_t 结构体以及双向链表方式组合起来，
    该locations指针将指向ngx_http_location_queue_t 结构体
    */
    ngx_queue_t  *locations;
}
```
### 2，ngx_http_location_queue_t

&emsp;&emsp; location queue将所有的相同前缀(结构体中的name)的location组织在一个队列中。

```c
typedef struct {
    ngx_queue_t                      queue;
    ngx_http_core_loc_conf_t        *exact;  //精确匹配的location数组
    ngx_http_core_loc_conf_t        *inclusive;  //前缀包含匹配的location数组
    ngx_str_t                       *name;    // 
    u_char                          *file_name;
    ngx_uint_t                       line;
    ngx_queue_t                      list;
}ngx_http_location_queue_t;
```

### 3, ngx_http_location_tree_node_s

```c
struct ngx_http_location_tree_node_s {
    // 左子树
    ngx_http_location_tree_node_t   *left;
    // 右子树
    ngx_http_location_tree_node_t   *right;
    // 完全匹配的location组成的树，包括exact和inclusive
    ngx_http_location_tree_node_t   *tree;

    /*
    如果location对应的URI匹配字符串属于能够完全匹配的类型，则exact指向其对应的ngx_http_core_loc_conf_t结构体，否则为NULL空指针
    */
    ngx_http_core_loc_conf_t        *exact;

    /*
    如果location对应的URI匹配字符串属于无法完全匹配的类型，则inclusive指向其对应的ngx_http_core_loc_conf_t 结构体，否则为NULL空指针
    */
    ngx_http_core_loc_conf_t        *inclusive;

    // 自动重定向标志
    u_char                           auto_redirect;

    // name字符串的实际长度
    u_char                           len;

    // name指向location对应的URI匹配表达式
    u_char                           name[1];
};
```

## location tree 的构建

&emsp;&emps;解析完一个http{}块的配置后，进入ngx_http_block函数，进行location tree的创建
 
1,  ngx_http_block是http模块的配置解析函数
```c
static char *
ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ...
    
    /* create location trees */
    // 对每个server块，构建location搜索树
    for (s = 0; s < cmcf->servers.nelts; s++) {
        //取得当前模块的location在全局location数组的位置
        clcf = cscfp[s]->ctx->loc_conf[ngx_http_core_module.ctx_index];
        
        if (ngx_http_init_locations(cf, cscfp[s], clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }

        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }

  ...
}
```

2， ngx_http_init_locations  
&emsp;&emsp;取出pclcf->locations,分类放在cscf->named_location 和pclcf->regex_locations里面
因此这里就初始化了ngx_http_core_loc_conf_s的regex_locations。

```c

static ngx_int_t
ngx_http_init_locations(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ...
    //指向当前location数组
    locations = pclcf->locations;

    if (locations == NULL) {
        return NGX_OK;
    }

    //对location数组排序
    ngx_queue_sort(locations, ngx_http_cmp_locations);

    ...

    // 开始对location进行分类
    for (q = ngx_queue_head(locations);
        q != ngx_queue_sentinel(locations);
        q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        //对当前location递归处理，函数调用结束之后，clcf的location是已经排好序的location数组了
        clcf = lq->exact ? lq->exact : lq->inclusive;
        if (ngx_http_init_locations(cf, NULL, clcf) != NGX_OK) {
            return NGX_ERROR;
        }

#if (NGX_PCRE)
        //regex记录的是正则路径的开始元素，r是正则路径的长度
        if (clcf->regex) {
            r++;

            if (regex == NULL) {
                regex = q;
            }

            continue;
        }

#endif

        if (clcf->named) {
            n++;

            if (named == NULL) {
                named = q;
            }

            continue;
        }

        if (clcf->noname) {
            break;
        }
    }

    //从q分裂队列
    if (q != ngx_queue_sentinel(locations)) {
        ngx_queue_split(locations, q, &tail);
    }

    // 取出named location，然后存在srv_conf的named_location里面
    if (named) {
        clcfp = ngx_palloc(cf->pool,
                           (n + 1) * sizeof(ngx_http_core_loc_conf_t **));
        if (clcfp == NULL) {
            return NGX_ERROR;
        }

        cscf->named_locations = clcfp;

        for (q = named;
             q != ngx_queue_sentinel(locations);
             q = ngx_queue_next(q))
        {
            lq = (ngx_http_location_queue_t *) q;

            *(clcfp++) = lq->exact;
        }

        *clcfp = NULL;

        ngx_queue_split(locations, named, &tail);
    }
    ...
}


3， ngx_http_init_static_location_trees 初始化pclcf->static_location。 这才是构造搜索树的开始。也只有static location 才构建搜索树。

```c

static ngx_int_t
ngx_http_init_static_location_trees(ngx_conf_t *cf,
    ngx_http_core_loc_conf_t *pclcf)
{

    ...

    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;
        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_ERROR;
        }
    }

```

```c
static ngx_int_t
ngx_http_init_static_location_trees(ngx_conf_t *cf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ...

    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;

        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_ERROR;
        }
    }


    if (ngx_http_join_exact_locations(cf, locations) != NGX_OK) {
        return NGX_ERROR;
    }

    //建立location搜索树
    ngx_http_create_locations_list(locations, ngx_queue_head(locations));

    pclcf->static_locations = ngx_http_create_locations_tree(cf, locations, 0);
    if (pclcf->static_locations == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```
/*
 *  如果exact_match的location和普通字符串匹配的location的名字相同，则合并到location的inclusive队列里面
 */
```c
static ngx_int_t
ngx_http_join_exact_locations(ngx_conf_t *cf, ngx_queue_t *locations)
{

    ngx_queue_t                *q, *x;
    ngx_http_location_queue_t  *lq, *lx;

    q = ngx_queue_head(locations);

    while (q != ngx_queue_last(locations)) {

        x = ngx_queue_next(q);

        lq = (ngx_http_location_queue_t *) q;
        lx = (ngx_http_location_queue_t *) x;

        if (ngx_strcmp(lq->name->data, lx->name->data) == 0) {

            if ((lq->exact && lx->exact) || (lq->inclusive && lx->inclusive)) {
                ngx_log_error(NGX_LOG_EMERG, cf->log, 0,
                              "duplicate location \"%V\" in %s:%ui",
                              lx->name, lx->file_name, lx->line);

                return NGX_ERROR;
            }
            //注意，lx不可能是exact_match，为什么呢？自己想想
            lq->inclusive = lx->inclusive;

            ngx_queue_remove(x);

            continue;
        }

        q = ngx_queue_next(q);
    }

    return NGX_OK;
}
```
/*
 *将具有相同的前缀的location移动到q->list里面
 */
```c

static void
ngx_http_create_locations_list(ngx_queue_t *locations, ngx_queue_t *q)
{
    u_char                     *name;
    size_t                      len;
    ngx_queue_t                *x, tail;
    ngx_http_location_queue_t  *lq, *lx;

    if (q == ngx_queue_last(locations)) {
        return;
    }

    lq = (ngx_http_location_queue_t *) q;

    //如果不是inclusive类型，直接进入下一个元素
    if (lq->inclusive == NULL) {
        ngx_http_create_locations_list(locations, ngx_queue_next(q));
        return;
    }

    len = lq->name->len;
    name = lq->name->data;

    //找出所有前缀是name的location，除了lq本身
    for (x = ngx_queue_next(q);
         x != ngx_queue_sentinel(locations);
         x = ngx_queue_next(x))
    {
        lx = (ngx_http_location_queue_t *) x;

        if (len > lx->name->len
            || (ngx_strncmp(name, lx->name->data, len) != 0))
        {
            break;
        }
    }

    q = ngx_queue_next(q);

    //没有找到前缀为name的location，直接进入下一个
    if (q == x) {
        ngx_http_create_locations_list(locations, x);
        return;
    }

    //在q这里分裂location，将具有name前缀的location插入lq->list
    ngx_queue_split(locations, q, &tail);
    ngx_queue_add(&lq->list, &tail);

    //如果location都处理完了，开始处理lq->list,例如处理完a,ab,abc,abd,name=a,  现在就是进行ab,abc,abd的处理
    if (x == ngx_queue_sentinel(locations)) {
        ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));
        return;
    }

    //将前缀不是name的location插入locations队列
    ngx_queue_split(&lq->list, x, &tail);
    ngx_queue_add(locations, &tail);

    //接续递归处理lq->list 和 locations建立搜索树
    ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));

    ngx_http_create_locations_list(locations, x);
}
```

### 创建location树
&emsp;&emsp; 终于进入树的构建了，前期对location队列做了排序和分类，那么接下来构建树的过程也就是一个递归过程了。

#### 实例分析

&emsp;&emsp;假设有这么一些排好序的 locations 队列：
    a ab abc abd ac acd ae bc bcd be，那么经过 ngx_http_create_locations_list  处理后，构造出来的结构大家如下图：

![location](location_tree.jpg),

可以看到，list队列就是一个排好序的单向链表，而左右子树仍然是放在双向队列里面。这样的设计，非常适合高效的检索。

#### 1，ngx_http_create_locations_tree (cf,locations,prefix)
locations是当前要处理的location队列，prefix是当前location队列的共同前缀。

```c

/*
 * to keep cache locality for left leaf nodes, allocate nodes in following
 * order: node, left subtree, right subtree, inclusive subtree
 */

static ngx_http_location_tree_node_t *
ngx_http_create_locations_tree(ngx_conf_t *cf, ngx_queue_t *locations,
    size_t prefix)
{
    size_t                          len;
    ngx_queue_t                    *q, tail;
    ngx_http_location_queue_t      *lq;
    ngx_http_location_tree_node_t  *node;

    //取出队列中间元素
    q = ngx_queue_middle(locations);

    // 处理出去共同前缀的部分
    lq = (ngx_http_location_queue_t *) q;
    len = lq->name->len - prefix;

    node = ngx_palloc(cf->pool,
                      offsetof(ngx_http_location_tree_node_t, name) + len);
    if (node == NULL) {
        return NULL;
    }

    node->left = NULL;
    node->right = NULL;
    node->tree = NULL;
    node->exact = lq->exact;
    node->inclusive = lq->inclusive;

    node->auto_redirect = (u_char) ((lq->exact && lq->exact->auto_redirect)
                           || (lq->inclusive && lq->inclusive->auto_redirect));

    node->len = (u_char) len;
    ngx_memcpy(node->name, &lq->name->data[prefix], len);

    //从中间分裂locations
    ngx_queue_split(locations, q, &tail);

    if (ngx_queue_empty(locations)) {
        /*
         * ngx_queue_split() insures that if left part is empty,
         * then right one is empty too
         */
        goto inclusive;
    }

    node->left = ngx_http_create_locations_tree(cf, locations, prefix);
    if (node->left == NULL) {
        return NULL;
    }

    ngx_queue_remove(q);

    if (ngx_queue_empty(&tail)) {
        goto inclusive;
    }

    node->right = ngx_http_create_locations_tree(cf, &tail, prefix);
    if (node->right == NULL) {
        return NULL;
    }

inclusive:

    if (ngx_queue_empty(&lq->list)) {
        return node;
    }

    node->tree = ngx_http_create_locations_tree(cf, &lq->list, prefix + len);
    if (node->tree == NULL) {
        return NULL;
    }

    return node;
}
```
