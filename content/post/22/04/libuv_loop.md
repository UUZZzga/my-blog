---
title: "libuv loop流程分析"
date: 2022-04-16T20:21:29+08:00
draft: true
tags:
    - libuv
author: "uzzzzz"
description: ""
image: ""
comments: true	# set false to hide Disqus
share: true	# set false to hide share buttons
menu: ""		# set "main" to add this content to the main menu
---
以前看过陈硕写的muduo那本书，现在想写自己的网络库，决定再分析下libuv。
分析先从事件循环开始
代码片段位于`src\unix\core.c`的`uv_run`

```c
while (r != 0 && loop->stop_flag == 0) {
  uv__update_time(loop);
  uv__run_timers(loop);
  ran_pending = uv__run_pending(loop);
  uv__run_idle(loop);
  uv__run_prepare(loop);
  timeout = 0;
  if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
    timeout = uv_backend_timeout(loop);
  uv__io_poll(loop, timeout);
  uv__metrics_update_idle_time(loop);
  uv__run_check(loop);
  uv__run_closing_handles(loop);
  if (mode == UV_RUN_ONCE) {
    uv__update_time(loop);
    uv__run_timers(loop);
  }
  r = uv__loop_alive(loop);
  if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
    break;
}
```

`uv__update_time`
更新当前事件循环的时间，避免频繁系统调用来获取时间。

`uv__run_timers`
有了当前时间就可以知道那些计时器超时了，执行超时的那些计时器的回调函数。

`uv__run_pending`
执行每轮循环的事件 。为空返回0，不为空返回1。
对应muduo的queueInLoop，libuv与muduo不同的是muduo的pending在事件循环末尾。

`uv__run_idle`
执行空转句柄。名字有误导性，每次循环都执行，不仅是真正空转时。
代码在`src\unix\loop_watcher.c`以宏函数的形式写的。

`uv__run_prepare`
执行准备句柄。I/O轮询前
代码在`src\unix\loop_watcher.c`以宏函数的形式写的。

`uv_backend_timeout`
计算阻塞时间。
停止（uv_stop）或，以上都不是则返回下一次计时器触发所用时间。
TODO: 没写完

`uv__io_poll`
执行io轮询操作，阻塞在此

`uv__metrics_update_idle_time`
TODO:

`uv__run_check`
执行检查句柄。每次循环都执行
代码在`src\unix\loop_watcher.c`以宏函数的形式写的。

`uv__run_closing_handles`
正在关闭句柄以链表形式，为每一个句柄执行关闭。
uv__run_closing_handles -> uv__finish_close
SIGNAL
如果管道内有信号被捕获 执行 uv__make_close_pending，重新加入正在关闭的句柄链表
PIPE（unix域）、TCP、TTY
执行uv__stream_destroy
UDP
执行uv__udp_finish_close
最后如果存在close_cb则执行

`uv__loop_alive`
返回活着的句柄数
