#nginx 配置解析

## ngx_conf_parse 流程

```c

char *
ngx_conf_parse(ngx_conf_t *cf, ngx_str_t *filename)
{
    char             *rv;
    ngx_fd_t          fd;
    ngx_int_t         rc;
    ngx_buf_t         buf;
    ngx_conf_file_t  *prev, conf_file;
    enum {
        parse_file = 0,
        parse_block,
        parse_param
    } type;
     
    //如果filename不为空    
     if (filename) {

        /* open configuration file */
        fd = ngx_open_file(filename->data, NGX_FILE_RDONLY, NGX_FILE_OPEN, 0);
        if (fd == NGX_INVALID_FILE) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, ngx_errno,
                               ngx_open_file_n " \"%s\" failed",
                               filename->data);
            return NGX_CONF_ERROR;
        }
        
        //保存当前文件配置文件信息， 为要保存？ 
        prev = cf->conf_file;
        cf->conf_file = &conf_file;
        //获得配置文件的统计信息(fstat)
         if (ngx_fd_info(fd, &cf->conf_file->file.info) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cf->log, ngx_errno,
                          ngx_fd_info_n " \"%s\" failed", filename->data);
        }
        
        //初始化buffer
        cf->conf_file->buffer = &buf;

        buf.start = ngx_alloc(NGX_CONF_BUFFER, cf->log);
        if (buf.start == NULL) {
            goto failed;
        }
        buf.pos = buf.start;
        buf.last = buf.start;
        buf.end = buf.last + NGX_CONF_BUFFER;
        buf.temporary = 1;
        // 保存配置文件的基本信息
        cf->conf_file->file.fd = fd;
        cf->conf_file->file.name.len = filename->len;
        cf->conf_file->file.name.data = filename->data;
        cf->conf_file->file.offset = 0;
        cf->conf_file->file.log = cf->log;
        cf->conf_file->line = 1;
        //标记为文件解析
        type = parse_file;
    } else if (cf->conf_file->file.fd != NGX_INVALID_FILE) {
        //标记为块解析，例如http{} ,server{},location{}
        type = parse_block;
    } else {
        //参数解析
        type = parse_param;
    }
    
    //循环解析
    for ( ;; ) {
        //读入一个token，一般是一行
	    //读到的配置参数放到: (ngx_str_t*)(*((*cf).args)).elt        
        rc = ngx_conf_read_token(cf);
         /*
         * ngx_conf_read_token() may return
         *
         *    NGX_ERROR             there is error
         *    NGX_OK                the token terminated by ";" was found
         *    NGX_CONF_BLOCK_START  the token terminated by "{" was found
         *    NGX_CONF_BLOCK_DONE   the "}" was found
         *    NGX_CONF_FILE_DONE    the configuration file is done
         */
         
        //***// 这一部分是rc的值，和当前的type进行合法性校验。 并且检查是否解析完毕
        //如果是rc == NGX_OK || rc=NGX_CONF_BLOCK_START,判断cf是否有handler回调，如果没有，直接使用ngx_conf_handler统一处理，否则进行handler调用。目前还没发现哪里有做这个调用，求高手指点
        
         if (cf->handler) {

            /*
             * the custom handler, i.e., that is used in the http's
             * "types { ... }" directive
             */
            
            //使用handler处理
            rv = (*cf->handler)(cf, NULL, cf->handler_conf);
            if (rv == NGX_CONF_OK) {
                continue;
            }

            if (rv == NGX_CONF_ERROR) {
                goto failed;
            }

            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, rv);

            goto failed;
        }

        //否则进入一般处理， 调用ngx_conf_handler对当前的token进行处理：
        /*
         * 1，获得token的名字，找出其在ngx_modules中对应的command，对command的type进行检查
         * 2，根据type找出当前token所在的块的配置信息，调用command里面用户自定义的set回调，如果，nginx提供了14个回到方法，如： ngx_conf_set_num_slot，ngx_conf_set_str_array_slot等等
         *
         */
        rc = ngx_conf_handler(cf, rc);

        if (rc == NGX_ERROR) {
            goto failed;
        }
    }
    
failed:

    rc = NGX_ERROR;

done:

    if (filename) {
        if (cf->conf_file->buffer->start) {
            ngx_free(cf->conf_file->buffer->start);
        }

        if (ngx_close_file(fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_ALERT, cf->log, ngx_errno,
                          ngx_close_file_n " %s failed",
                          filename->data);
            return NGX_CONF_ERROR;
        }

        cf->conf_file = prev;
    }

    if (rc == NGX_ERROR) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
```

        