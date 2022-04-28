---
title: "libuv epoll流程分析"
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
使用epoll处理io的代码在`src/unix/epoll.c`下的`uv__io_poll`函数

## watcher_queue

对watcher_queue内所有成员执行epoll_ctl

如TCP关闭流程
uv_close -> uv__tcp_close -> uv__stream_close -> uv__io_close -> uv__io_stop
uv__io_stop的代码片

```c
// ...
if (w->pevents == 0) {
  QUEUE_REMOVE(&w->watcher_queue);
  QUEUE_INIT(&w->watcher_queue);
  w->events = 0;

  if (w == loop->watchers[w->fd]) {
    assert(loop->nfds > 0);
    loop->watchers[w->fd] = NULL;
    loop->nfds--;
  }
}
else if (QUEUE_EMPTY(&w->watcher_queue))
  QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);
```

libuv 执行`uv_close`并不会立刻执行epoll_ctl
而是在事件修改时放入watcher_queue，目的应该是解耦

## 在wait期间是否阻塞SIGPROF信号

代码片

```c
sigmask = 0;
if (loop->flags & UV_LOOP_BLOCK_SIGPROF) {
  sigemptyset(&sigset);
  sigaddset(&sigset, SIGPROF);
  sigmask |= 1 << (SIGPROF - 1);
}
```

## wait
代码片
```c
if (sigmask != 0 && no_epoll_pwait != 0)
  if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
    abort();

if (no_epoll_wait != 0 || (sigmask != 0 && no_epoll_pwait == 0)) {
  nfds = epoll_pwait(loop->backend_fd,
                     events,
                     ARRAY_SIZE(events),
                     timeout,
                     &sigset);
  if (nfds == -1 && errno == ENOSYS) {
    uv__store_relaxed(&no_epoll_pwait_cached, 1);
    no_epoll_pwait = 1;
  }
} else {
  nfds = epoll_wait(loop->backend_fd,
                    events,
                    ARRAY_SIZE(events),
                    timeout);
  if (nfds == -1 && errno == ENOSYS) {
    uv__store_relaxed(&no_epoll_wait_cached, 1);
    no_epoll_wait = 1;
  }
}

if (sigmask != 0 && no_epoll_pwait != 0)
  if (pthread_sigmask(SIG_UNBLOCK, &sigset, NULL))
    abort();
```

`no_epoll_wait`、`no_epoll_pwait`、`no_epoll_wait_cached`、`no_epoll_pwait_cached`
是用来判断使用epoll_pwait还是 pthread_sigmask + epoll_wait的
`XXX_cached`是原子变量用来缓存，使用 relaxed 进行加载，在多线程情况下不需要考虑内存可见性，
最多是多执行几次系统调用。

events 是epoll_event数组 长1024

## 更新时间

执行完阻塞操作必须更新循环内的时间，
就算是无阻塞也不能排除操作系统在系统调用时进程调度
```c
/* Update loop->time unconditionally. It's tempting to skip the update when
 * timeout == 0 (i.e. non-blocking poll) but there is no guarantee that the
 * operating system didn't reschedule our process while in the syscall.
 */
SAVE_ERRNO(uv__update_time(loop));
```

## 超时判断

代码片

```c
if (nfds == 0) {
  assert(timeout != -1);
  if (reset_timeout != 0) {
    timeout = user_timeout;
    reset_timeout = 0;
  }
  if (timeout == -1)
    continue;
  if (timeout == 0)
    return;
  /* We may have been inside the system call for longer than |timeout|
   * milliseconds so we need to update the timestamp to avoid drift.
   */
  goto update_timeout;
}
```

`reset_timeout`重置timeout
`timeout == -1` 重新到wait无限阻塞
TODO 为什么会出现
`timeout == 0` 没有发生事件且无阻塞，直接返回
`goto update_timeout;`跳到[重新计算超时时间](#重新计算超时时间)

## 处理错误

`errno == ENOSYS`没有系统调用 重试
`errno != EINTR` 执行失败
其余逻辑和[超时判断](#超时判断)一样

## 处理触发的事件

代码片

```c
for (i = 0; i < nfds; i++) {
  pe = events + i;
  fd = pe->data.fd;
  /* Skip invalidated events, see uv__platform_invalidate_fd */
  if (fd == -1)
    continue;
  assert(fd >= 0);
  assert((unsigned) fd < loop->nwatchers);
  w = loop->watchers[fd];
  if (w == NULL) {
    /* File descriptor that we've stopped watching, disarm it.
     *
     * Ignore all errors because we may be racing with another thread
     * when the file descriptor is closed.
     */
    epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
    continue;
  }
  /* Give users only events they're interested in. Prevents spurious
   * callbacks when previous callback invocation in this loop has stopped
   * the current watcher. Also, filters out events that users has not
   * requested us to watch.
   */
  pe->events &= w->pevents | POLLERR | POLLHUP;
  /* Work around an epoll quirk where it sometimes reports just the
   * EPOLLERR or EPOLLHUP event.  In order to force the event loop to
   * move forward, we merge in the read/write events that the watcher
   * is interested in; uv__read() and uv__write() will then deal with
   * the error or hangup in the usual fashion.
   *
   * Note to self: happens when epoll reports EPOLLIN|EPOLLHUP, the user
   * reads the available data, calls uv_read_stop(), then sometime later
   * calls uv_read_start() again.  By then, libuv has forgotten about the
   * hangup and the kernel won't report EPOLLIN again because there's
   * nothing left to read.  If anything, libuv is to blame here.  The
   * current hack is just a quick bandaid; to properly fix it, libuv
   * needs to remember the error/hangup event.  We should get that for
   * free when we switch over to edge-triggered I/O.
   */
  if (pe->events == POLLERR || pe->events == POLLHUP)
    pe->events |=
      w->pevents & (POLLIN | POLLOUT | UV__POLLRDHUP | UV__POLLPRI);
  if (pe->events != 0) {
    /* Run signal watchers last.  This also affects child process watchers
     * because those are implemented in terms of signal watchers.
     */
    if (w == &loop->signal_io_watcher) {
      have_signals = 1;
    } else {
      uv__metrics_update_idle_time(loop);
      w->cb(loop, w, pe->events);
    }
    nevents++;
  }
}
```
如果满足`w == &loop->signal_io_watcher`则延后到[处理信号](#处理信号)
否则 是这么执行的
`w->cb(loop, w, pe->events);`

## 判断重置超时

代码片

```c
if (reset_timeout != 0) {
  timeout = user_timeout;
  reset_timeout = 0;
}
```

## 处理信号

```c
if (have_signals != 0) {
  uv__metrics_update_idle_time(loop);
  loop->signal_io_watcher.cb(loop, &loop->signal_io_watcher, POLLIN);
}

// ...

if (have_signals != 0)
  return;  /* Event loop should cycle now so don't poll again. */
```

## 处理完成退出逻辑

代码片

```c
if (nevents != 0) {
  if (nfds == ARRAY_SIZE(events) && --count != 0) {
    /* Poll for more events but don't block this time. */
    timeout = 0;
    continue;
  }
  return;
}
if (timeout == 0)
  return;
if (timeout == -1)
  continue;
```

## 重新计算超时时间

代码片

```c
real_timeout -= (loop->time - base);
if (real_timeout <= 0)
  return;

timeout = real_timeout;
```
