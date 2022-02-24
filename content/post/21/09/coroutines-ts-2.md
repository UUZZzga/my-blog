+++
title = "c++20 coroutines-ts 2"
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
目标 设计一个scheduler调度器 正常执行以下代码
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

未完
