#Nginx location转发顺序

## 匹配规则
### 原型 
 **location [=|~|~*|^~|@] /uri/ { … }**
>
### 普通字符串匹配<a name=str>?</a>
>>
1 "="   ：精确匹配，如果匹配到跳出匹配过程
>>
2 "^~"  : 最大前缀匹配， 如果匹配跳出匹配匹配过程
>>
3 不带任何前缀： 最大前缀匹配，例如location /{} 跳表以/开头的字符串搜索匹配，再没有正则表达式匹配的情况才进行这个匹配，优先级最低

>
#### 正则表达式匹配<a name=reg>?</a>

>>
4 "~[*]"  : 大小写相关[无关]的正则匹配，  
>>
5 "@" ： named location，不是普通的location匹配，而是location内部重定向(internally redirected)的变量


### 匹配规则

*  [普通字符串匹配](#str)优先执行，然后使用[正则表达式匹配](#reg)
*  使用正则表达式匹配会覆盖字符串匹配的结果，但是不包括上面的**1**,**2**的情况
*  无法直接表达不匹配某个表达式，因此可以先把不匹配的正则表达式内容部分留空，然后用/来处理

### 源码解析
```c
static char *
ngx_http_core_location(ngx_conf_t *cf, ngx_command_t *cmd, void *dummy)
{
    char                      *rv; 
    u_char                    *mod;
    size_t                     len; 
    ngx_str_t                 *value, *name;
    ngx_uint_t                 i;   
    ngx_conf_t                 save;
    ngx_http_module_t         *module;
    ngx_http_conf_ctx_t       *ctx, *pctx;
    ngx_http_core_loc_conf_t  *clcf, *pclcf;

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }    

    pctx = cf->ctx;
    ctx->main_conf = pctx->main_conf;
    ctx->srv_conf = pctx->srv_conf;
    
    //初始化loc_conf
    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }    
    
     for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_HTTP_MODULE) {
            continue;
        }    

        module = ngx_modules[i]->ctx;
        // 创建存储location级别的全局配置
        if (module->create_loc_conf) {
            ctx->loc_conf[ngx_modules[i]->ctx_index] =
                                                   module->create_loc_conf(cf);
            if (ctx->loc_conf[ngx_modules[i]->ctx_index] == NULL) {
                 return NGX_CONF_ERROR;
            }
        }
    }
    
    clcf = ctx->loc_conf[ngx_http_core_module.ctx_index];
    clcf->loc_conf = ctx->loc_conf;
    //获取 location 行解析结果，数组类型，如：["location", "^~", "/test/"]
    value = cf->args->elts;
    if (cf->args->nelts == 3) {

        len = value[1].len;
        mod = value[1].data;
        name = &value[2];
        //完全匹配
        if (len == 1 && mod[0] == '=') {

            clcf->name = *name;
            clcf->exact_match = 1;
        //普通字符串匹配-最大前缀匹配
        } else if (len == 2 && mod[0] == '^' && mod[1] == '~') {

            clcf->name = *name;
            clcf->noregex = 1;
        // 区分大小写的正则匹配
        } else if (len == 1 && mod[0] == '~') {
           
            if (ngx_http_core_regex_location(cf, clcf, name, 0) != NGX_OK) {
                return NGX_CONF_ERROR;
            }
         //不区分大小写的正则匹配
        } else if (len == 2 && mod[0] == '~' && mod[1] == '*') {
            if (ngx_http_core_regex_location(cf, clcf, name, 1) != NGX_OK) {
                return NGX_CONF_ERROR;
            }

            } else {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid location modifier \"%V\"", &value[1]);
                return NGX_CONF_ERROR;
            }
     } else {
         ...   
     }
     
     // 指向ngx_http_core_module的全局location配置数组
     pclcf = pctx->loc_conf[ngx_http_core_module.ctx_index];
     if (pclcf->name.len) {
        //精确匹配和named location里面不能再嵌套location块        
        if (pclcf->exact_match) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the exact location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (pclcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the named location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (clcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "named location \"%V\" can be "
                               "on the server level only",
                               &clcf->name);
            return NGX_CONF_ERROR;
        }
        
        ...
        //将当前location插入父location的队列里面
        if (ngx_http_add_location(cf, &pclcf->locations, clcf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    //保存当前配置信息，解解析块内内容
         save = *cf;
    cf->ctx = ctx;
    cf->cmd_type = NGX_HTTP_LOC_CONF;

    rv = ngx_conf_parse(cf, NULL);

    *cf = save;

    return rv;
}
    

```

上面是解析location配置，下面是穿件location tree的时候，对locations队列的排序，由此可以获得location的转发顺序

```c

static ngx_int_t
ngx_http_cmp_locations(const ngx_queue_t *one, const ngx_queue_t *two)
{
    ngx_int_t                   rc;
    ngx_http_core_loc_conf_t   *first, *second;
    ngx_http_location_queue_t  *lq1, *lq2;

    lq1 = (ngx_http_location_queue_t *) one;
    lq2 = (ngx_http_location_queue_t *) two;

    first = lq1->exact ? lq1->exact : lq1->inclusive;
    second = lq2->exact ? lq2->exact : lq2->inclusive;
    
    
    if (first->noname && !second->noname) {
        /* shift no named locations to the end */
        return 1;
    }

    if (!first->noname && second->noname) {
        /* shift no named locations to the end */
        return -1;
    }

    if (first->noname || second->noname) {
        /* do not sort no named locations */
        return 0;
    }

    if (first->named && !second->named) {
        /* shift named locations to the end */
        return 1;
    }

    if (!first->named && second->named) {
        /* shift named locations to the end */
        return -1;
     
    if (first->named && second->named) {
        return ngx_strcmp(first->name.data, second->name.data);
    }

#if (NGX_PCRE)

    if (first->regex && !second->regex) {
        /* shift the regex matches to the end */
        return 1;
    }

    if (!first->regex && second->regex) {
        /* shift the regex matches to the end */
        return -1;
    }

    if (first->regex || second->regex) {
        /* do not sort the regex matches */
        return 0;
    }

#endif
    rc = ngx_strcmp(first->name.data, second->name.data);

    if (rc == 0 && !first->exact_match && second->exact_match) {
        /* an exact match must be before the same inclusive one */
        return 1;
    }

    return rc;
}

```
