+++
title = "c++20 coroutines-ts 1"
date = "2021-09-01T11:01:40+08:00"
tags = ["c++","c++20","c++2a","coroutines-ts"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "c++20 coroutines-ts 协程"
+++
#目标
目标正常执行以下代码
```c++
task<int> fibon(int x)
{
  if (x == 0)
  {
    co_return 0;
  }
  if (x == 1)
  {
    co_return 1;
  }
  co_return co_await fibon(x - 1) + co_await fibon(x - 2);
}
```
为什么会出错
##编译器遇到co_await的处理
```c++
	co_await MyCoroutine(args...);
```
以上代码会被处理为
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
        coro.await_resume();
    } catch (...) {
        promise.unhandled_exception();
    }
    co_await promise.final_suspend();//注意
    //析构 promise
    //析构 coroutine 结构体里的参数
    //析构 coroutine coro
}
```
可以看到 final_suspend 这里没有被try包裹
所以 final_suspend 返回的 类/结构体 所有方法必须是 noexcept 的
如std::suspend_never
```c++
struct suspend_never {
    [[nodiscard]] constexpr bool await_ready() const noexcept {
        return true;
    }

    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};
```
其中的 `[[nodiscard]]` 是 C++17/c++20 加入的 属性记号
详见 https://zh.cppreference.com/w/cpp/feature_test

对于我们的目标会被编译器处理成什么样
```c++
co_await fibon(2);

...

task<int> fibon(int x)
{
    //创建 coroutine coro
    //复制参数到 coroutine 结构体
    promise_type promise;

    coro.promise=promise;
    auto return_object = promise.get_return_object();
    try {
        co_await promise.initial_suspend();
		
		//创建 coroutine coro
		//复制参数到 coroutine 结构体
		promise_type promise;
		coro.promise=promise;
		auto return_object = promise.get_return_object();
		try {
			co_await promise.initial_suspend();
			
			//创建 coroutine coro
			//复制参数到 coroutine 结构体
			promise_type promise;
			coro.promise=promise;
			auto return_object = promise.get_return_object();
			try {
				co_await promise.initial_suspend();
				
				promise.return_value(1);//注意
			} catch (...) {
				promise.unhandled_exception();
			}
			co_await promise.final_suspend();
			//析构 promise
			//析构 coroutine 结构体里的参数
			//析构 coroutine coro
			
			
			//创建 coroutine coro
			//复制参数到 coroutine 结构体
			promise_type promise;
			coro.promise=promise;
			auto return_object = promise.get_return_object();
			try {
				co_await promise.initial_suspend();
				
				promise.return_value(0);//注意
			} catch (...) {
				promise.unhandled_exception();
			}
			co_await promise.final_suspend();
			//析构 promise
			//析构 coroutine 结构体里的参数
			//析构 coroutine coro
			
			// ...
		
        if(!coro.await_ready()){
            coro.await_suspend();
        }
        coro.await_resume();
    } catch (...) {
        promise.unhandled_exception();
    }
    co_await promise.final_suspend();
    //析构 promise
    //析构 coroutine 结构体里的参数
    //析构 coroutine coro
}
```
我是根据执行先后顺序来推测的
里面有两个协程 执行了 initial_suspend 后 没有执行 coro的方法 直接执行return_value
设置值、执行final_suspend结束了
```c++
task<int> fibon(int x)
{
  if (x == 0)
  {
    co_return 0;//这里
  }
  if (x == 1)
  {
    co_return 1;//这里
  }
  co_return co_await fibon(x - 1) + co_await fibon(x - 2);
}
```

这会出现什么问题？
如果你在await_suspend中把 handle 存到数组里，并在循环中执行，如
```c++
//...
void await_suspend(std::coroutine_handle<> handle)
{
  auto loop = getEventLoopOfCurrentThread();
  loop->addCoroHandle(handle);
}
//...

void EventLoop::coroHandle(){
	for(auto handle:reday_coro_){ //注意
		handle();//等效于 handle.resume();
	}
	reday_coro_.clear();
}
```
当x为4时，猜猜执行如下代码会怎样
```c++
task<int> fibon(int x)
{
  if (x == 0)
  {
    co_return 0;
  }
  if (x == 1)
  {
    co_return 1;
  }
  co_return co_await fibon(x - 1)/*位置1*/ + co_await fibon(x - 2)/*位置2*/;
}
```
由于我们是在 await_suspend 中加入数组 但是这是在递归 会在执行完 位置1 执行位置2触发 await_suspend加入到尾部，但原来的尾部是什么？co_return,它会先返回在执行 位置2
所以需要 设计 前后关系
##前后关系
在 await_suspend 建立关系 并在 final_suspend 中处理前后关系
由于其中一段被编译器展开,在初始化时已经执行过 final_suspend 了
然后你就会发现程序什么都没执行

因为 执行顺序是
x=4 进入位置1,
	x=3 进入位置1,
		x=2 进入位置1,
			x=1 返回1, 在初始化时就执行过 final_suspend
		x=2 进入位置2, 要完成 'x=2 进入位置1'
			x=0 返回0, 在初始化时就执行过 final_suspend
	x=3 进入位置2, 要完成 'x=3 进入位置1'
		x=1 返回1,
x=4 进入位置2
...

当在执行 'x=2 进入位置2' 对应的await_suspend时，在确定前后关系之前就执行过 final_suspend 'x=1 返回1' 了，我们的代码是建立在 所有协程先执行 await_suspend 确定前后关系，在执行 final_suspend 执行 前后关系表 对应的 coro.resume(), 但是在初始化时 执行'x=1 返回1' 我们此时 前后关系表 没有建立、为空，
所以什么都没执行
##解决办法
final_suspend 执行 前后关系表 对应的 coro.resume()时如果没找到 对应的 则存到数组里 事件循环 数组遍历 执行对应的coro.resume()
