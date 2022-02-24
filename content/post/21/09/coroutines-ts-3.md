+++
title = "c++20 coroutines-ts 3"
date = "2021-09-01T11:01:40+08:00"
tags = ["c++","c++20","c++2a","coroutines-ts"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "c++20 coroutines-ts 协程"
+++

##  异常
对于以下代码
```c++
	try{
		co_await MyCoroutine(args...);
	}catch(XXX e){
	
	}
```
会被处理为
```c++
return_type MyCoroutine(args...)
{
    //创建 coroutine coro
    //复制参数到 coroutine 结构体
    promise_type promise;

    coro.promise=promise;
    auto return_object = promise.get_return_object();

    try {
        co_await promise.initial_suspend();
        if(!coro.await_ready()){
            coro.await_suspend();
        }
		if(promise.cancellation_requested())
			goto end-label;
        ret coro.await_resume();
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
	final_suspend:
    co_await promise.final_suspend();

    //析构 promise
    //析构 coroutine 结构体里的参数
    //析构 coroutine coro
}
```

未完
