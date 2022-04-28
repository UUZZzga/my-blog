---
title: "libuv stream流程分析"
date: 2022-04-17T18:50:00+08:00
draft: true
author: "uzzzzz"
description: ""
image: ""
comments: true	# set false to hide Disqus
share: true	# set false to hide share buttons
menu: ""		# set "main" to add this content to the main menu
tags:
    - libuv
---

## 创建

### 与epoll的交互
是这么执行的
`w->cb(loop, w, pe->events);`

stream通过
```c
uv__io_init(&stream->io_watcher, uv__stream_io, -1);
```

## uv__stream_io

### 对连接特殊处理

```c
if (stream->connect_req) {
  uv__stream_connect(stream);
  return;
}
```

执行 uv__stream_connect

### 读取和关闭

```c
/* Ignore POLLHUP here. Even if it's set, there may still be data to read. */
if (events & (POLLIN | POLLERR | POLLHUP))
  uv__read(stream);
```
`events`存在 `POLLIN` | `POLLERR` | `POLLHUP`
则执行`uv__read`

### 关闭后

```c
if (uv__stream_fd(stream) == -1)
  return;  /* read_cb closed stream. */
```

关闭后则直接返回

### 写入

```c
if (events & (POLLOUT | POLLERR | POLLHUP)) {
  uv__write(stream);
  uv__write_callbacks(stream);

  /* Write queue drained. */
  if (QUEUE_EMPTY(&stream->write_queue))
    uv__drain(stream);
}
```

## 连接

在一个非阻塞的TCP套接字上调用connect时，Connect将立即返回一个EINPROGRESS错误
连接建立可写，连接建立遇到错误可读可写。
连接已经建立并有来自对端的数据到达，这种情况也是可读可写。

### 获取错误码

```c
if (stream->delayed_error) {
  /* To smooth over the differences between unixes errors that
   * were reported synchronously on the first connect can be delayed
   * until the next tick--which is now.
   */
  error = stream->delayed_error;
  stream->delayed_error = 0;
} else {
  /* Normal situation: we need to get the socket error from the kernel. */
  assert(uv__stream_fd(stream) >= 0);
  getsockopt(uv__stream_fd(stream),
             SOL_SOCKET,
             SO_ERROR,
             &error,
             &errorsize);
  error = UV__ERR(error);
}
```

主动获取错误

### 忽略EINPROGRESS

```c
if (error == UV__ERR(EINPROGRESS))
  return;
```

### 处理错误

```c
if (error < 0 || QUEUE_EMPTY(&stream->write_queue)) {
  uv__io_stop(stream->loop, &stream->io_watcher, POLLOUT);
}
```

`uv__io_stop`[详见](../libuv_epoll/#watcherqueue)

### 连接回执

存在cb 则传入error执行cb

#### cb

uv_tcp_connect -> uv__tcp_connect
代码片

```c
int uv__tcp_connect(/* ... , */
                    uv_connect_cb cb) {
  // ...
  req->cb = cb;
  // ...
}
```

以 uv_connect() 开启连接完成后的回调函数。 status 若成功将是0，否则 < 0。

### 发生错误清空写入队列

```c
if (uv__stream_fd(stream) == -1)
  return;

if (error < 0) {
  uv__stream_flush_write_queue(stream, UV_ECANCELED);
  uv__write_callbacks(stream);
}
```

