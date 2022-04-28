---
title: "Pilot_tick_input"
date: 2022-04-26T22:03:31+08:00
author: "uzzzzz"
description: ""
image: ""
comments: true	# set false to hide Disqus
share: true	# set false to hide share buttons
menu: ""		# set "main" to add this content to the main menu
tags:
    - Pilot
---
## tick中的input操作

Engine 的 tick 方法 执行的操作， 更新时间 、逻辑tick、渲染tick。

使用的底层窗口封装是glfw

设计上只有Renderer能操作glfw(InputSystem 只使用glfw的key相关的宏定义)

tick中的输入使用的glfw的输入，流程如下

PilotRenderer::tick ->
SurfaceIO::tick_NotQuit

tick_NotQuit代码片

```cpp
if (!glfwWindowShouldClose(m_window))
{
    glfwPollEvents();
    return true;
}
return false;
```

内容是 Window 没关闭就 glfwPollEvents

既然有glfwPollEvents那么必定有 注册glfw的回调，就在initialize内

## SurfaceIO::initialize 初始化

先设置 Client API，我们先看看官方文档描述都支持哪些Client API。

> **GLFW_CLIENT_API** indicates the client API provided by the window's context; either **GLFW_OPENGL_API**, **GLFW_OPENGL_ES_API** or **GLFW_NO_API**.

这里可以看到Client API是指Open GL 、 OpenGL ES相关，Pilot使用的是Vulkan所以使用的是**GLFW_NO_API**。

让后执行`glfwCreateWindow`创建窗口。
使用`glfwSetWindowUserPointer`设置用户数据为this，有set设置用户数据相应的有get来获取设置用户数据`glfwGetWindowUserPointer`。

分别执行
`glfwSetKeyCallback`、
`glfwSetCharCallback`、
`glfwSetCharModsCallback`、
`glfwSetMouseButtonCallback`、
`glfwSetCursorPosCallback`、
`glfwSetCursorEnterCallback`、
`glfwSetScrollCallback`、
`glfwSetDropCallback`、
`glfwSetWindowSizeCallback`、
`glfwSetWindowCloseCallback`
设置回调，回调内会执行SurfaceIO内对应事件处理，也就是on开头的方法，对应方法执行监听相应事件的回调。
这里聊一下`windowSizeCallback`
```cpp
Pilot::SurfaceIO* app = (Pilot::SurfaceIO*)glfwGetWindowUserPointer(window);
app->m_width          = width;
app->m_height         = height;
app->reset(app->m_reset);
app->onWindowSize(width, height);
```
我们可以看到他是先执行`reset`再执行onWindowSize的，
reset 会把参数设置为m_reset（我把我改成我自己）再执行onReset
。

最后执行
```cpp
glfwSetInputMode(m_window, GLFW_RAW_MOUSE_MOTION, GLFW_FALSE);
```
关闭raw格式输入，
这里我想说[官方文档描述](https://www.glfw.org/docs/latest/input_guide.html#raw_mouse_motion)

> It is disabled by default.

默认关闭。

consumingBufferShift

## SurfaceIO 其余杂项

`getWidth`、
`getHeight`、
`setSize`、
`getTitle`、
`setTitle`
这些就不用说了看名字就知道。

`m_is_editor_mode`、
`m_is_focus_mode`、
`setEditorMode`、
`setFocusMode`
这些以后写。

`toggleFullscreen`
以后再写。
TODO: 未完成

`isKeyDown`、`isMouseButtonDown`
判断键盘按下、判断鼠标按下。
不再传入值按键范围直接返回false，
执行`glfwGetKey`、`glfwGetMouseButton`返回值和GLFW_PRESS比较

## tick_NotQuit之后

tick_NotQuit之后没关闭就可以开始渲染了，代码片

```cpp
if (framebuffer && framebuffer->m_scene && framebuffer->m_scene->m_loaded)
{
    m_rhi->tick_pre(framebuffer, release_handles);
    m_ui->tick_pre(uistate);

    m_ui->tick_post(uistate);
    m_rhi->tick_post(framebuffer);
}
```

## 总结

Pilot的 input 暂时划分在 Render 中

<!--
rhi 全称 Render Hardware Interface 中文渲染硬件接口。
代码逻辑就是有要渲染的资源，将要渲染的资源提交给显卡。

为什么要先对rhi执行tick_pre，再对ui执行。代码片如下

```cpp
void SurfaceRHI::tick_pre(const FrameBuffer* framebuffer, SceneReleaseHandles& release_handles)
{
    // prepare material and mesh assets
    m_vulkan_manager->syncScene(*framebuffer->m_scene, m_pilot_renderer, release_handles);

    m_vulkan_manager->beginFrame();
}
```

syncScene是和scene系统交互的桥梁。
既然tick_pre会调用 beginFrame，那么 rhi的tick_post呢

```cpp
void SurfaceRHI::tick_post(const FrameBuffer* framebuffer) {
    m_vulkan_manager->endFrame();
}
``` -->
