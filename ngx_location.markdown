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
    // 无法完全匹配的location组成的树
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
&emsp;&emsp;递归初始化ngx_http_core_loc_conf_s中的static_location 和regex_location

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




    