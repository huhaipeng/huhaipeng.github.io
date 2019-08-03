```c
if (ngx_add_conn) { //ngx_add_conn为向epoll注册新事件的函数
    if (ngx_add_conn(c) == NGX_ERROR) {
        goto failed;
    }
}

ngx_log_debug3(NGX_LOG_DEBUG_EVENT, pc->log, 0,
               "connect to %V, fd:%d #%uA", pc->name, s, c->number);

rc = connect(s, pc->sockaddr, pc->socklen);//connect to peer

if (rc == -1) {
    err = ngx_socket_errno;
    ...
}

```
在异步编程主动连接的时候，最好把向epoll注册新事件的步骤放于connect函数前，因为有可能connect会立即返回成功；若把epoll注册事件放在connect之后，有可能会丢失这个事件的通知。（实测好像不会丢失。。。）