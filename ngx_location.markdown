# nginx location设计

## 核心数据结构

### 1， ngx_http_core_loc_conf_s  

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

## location tree 的构建
 解析完一个http{}块的配置后，进入ngx_http_block函数，进行location tree的创建
 

```c
static char *
ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ...
    
    /* create location trees */

    for (s = 0; s < cmcf->servers.nelts; s++) {

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


    