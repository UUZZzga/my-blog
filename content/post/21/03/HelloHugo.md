+++
date = "2021-03-23T09:19:06+08:00"
title = "HelloHugo"
# slug = "post-title"
tags = ["test"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "测试 hugo"
+++

# 测试
## test
----

#### 无序列表
- 1
* a
+ A

#### 有序列表
1. 1.a
2. 2.b
3. 3.c
9. 9.d 注意

#### 区块引用
> 区块引用测试
> > 再次嵌套
> > > 再次再次嵌套

#### 行内式
[测试](#test)跳转到测试锚点

#### 图片
bing logo![测试](https://cn.bing.com/sa/simg/favicon-2x.ico)bing

#### 删除线
~~aa~~

#### 代码
`printf("Hello World!\n");`

```c
printf("Hello World!\n");
```

```cpp
using namespace std;
cout<<"Hello World!"<<endl;
```