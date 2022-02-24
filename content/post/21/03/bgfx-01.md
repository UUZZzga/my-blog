+++
title = "Bgfx 01"
date = "2021-03-30T09:45:15+08:00"
tags = ["c++","bgfx"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "Bgfx 示例 HelloWorld 分析"
+++

## ExampleHelloWorld 类
### init 方法
```cpp
bgfx::Init init;
init.type     = args.m_type;
init.vendorId = args.m_pciId;
init.resolution.width  = m_width;
init.resolution.height = m_height;
init.resolution.reset  = m_reset;
bgfx::init(init);
```
对bgfx 进行初始化
可以指定渲染引擎
`init.type = bgfx::RendererType::Vulkan;`

```cpp
m_debug  = BGFX_DEBUG_TEXT;
...
bgfx::setDebug(m_debug);
```
设置debug信息
其他可选项有
1. BGFX_DEBUG_IFH 跳过所有呈现调用。这在分析CPU和GPU之间的潜在瓶颈时非常有用。
2. BGFX_DEBUG_PROFILER 启用探查器(profiler)。
3. BGFX_DEBUG_STATS 显示内部统计信息。
4. BGFX_DEBUG_TEXT 显示调试文本。
5. BGFX_DEBUG_WIREFRAME 线框渲染。所有渲染原语都将渲染为线。

```cpp
// Set view 0 clear state.
bgfx::setViewClear(0
	, BGFX_CLEAR_COLOR|BGFX_CLEAR_DEPTH
	, 0x303030ff
	, 1.0f
	, 0
	);
```
[in] _id: View id.
[in] _flags: 清除标志。使用 BGFX_CLEAR_NONE 移除任何清除操作。请参见： BGFX_CLEAR_*.
[in] _rgba: 颜色清除值。
[in] _depth: 深度清除值。
[in] _stencil: Stencil 清除值。
设置 BGFX_CLEAR_COLOR|BGFX_CLEAR_DEPTH 清除时使用 颜色和深度 清除后颜色为 303030 深度为1.0

```cpp
imguiCreate();
```
使用了imgui 做窗口

### shutdown 方法
```cpp
virtual int shutdown() override
{
	imguiDestroy();

	// Shutdown bgfx.
	bgfx::shutdown();

	return 0;
}
```
先关闭窗口再关闭渲染

### update 方法
```cpp
if (!entry::processEvents(m_width, m_height, m_debug, m_reset, &m_mouseState) )
{
	...
}
return false;
```
processEvents 无阻判断是否 发生事件 
处理事件 
Axis 手柄 轴事件
Char,
Exit 退出事件 发生后 要手动检查是否退出
Gamepad 手柄事件
Key 键盘事件
Mouse 鼠标事件
Size 窗口大小改变事件
Window 窗口事件
Suspend 暂停事件
DropFile 拖拽文件事件

```cpp
bgfx::setViewRect(0, 0, 0, uint16_t(m_width), uint16_t(m_height) );
```
为 id为0的 view 设置视图矩形。绘制图元外部视图将被剪裁。

```cpp
bgfx::touch(0);
```
如果没有其他绘制调用提交到视图0，则此虚拟绘制调用用于确保视图0被清除。

```cpp
bgfx::dbgTextClear();
```
清除内部调试文本缓冲区。
```cpp
bgfx::dbgTextImage(
	  bx::max<uint16_t>(uint16_t(m_width /2/8 ), 20)-20
	, bx::max<uint16_t>(uint16_t(m_height/2/16),  6)-6
	, 40
	, 12
	, s_logo
	, 160
	);
```
_x , _y:  左上角 2D位置.
_width , _height: 图像宽度和高度。
_data: 原始图像数据（字符/属性原始编码）。
_pitch: 图像间距（字节）。

```cpp
bgfx::dbgTextPrintf(0, 1, 0x0f, "Color can be changed with ANSI \x1b[9;me\x1b[10;ms\x1b[11;mc\x1b[12;ma\x1b[13;mp\x1b[14;me\x1b[0m code too.");

bgfx::dbgTextPrintf(80, 1, 0x0f, "\x1b[;0m    \x1b[;1m    \x1b[; 2m    \x1b[; 3m    \x1b[; 4m    \x1b[; 5m    \x1b[; 6m    \x1b[; 7m    \x1b[0m");
bgfx::dbgTextPrintf(80, 2, 0x0f, "\x1b[;8m    \x1b[;9m    \x1b[;10m    \x1b[;11m    \x1b[;12m    \x1b[;13m    \x1b[;14m    \x1b[;15m    \x1b[0m");

const bgfx::Stats* stats = bgfx::getStats();
bgfx::dbgTextPrintf(0, 2, 0x0f, "Backbuffer %dW x %dH in pixels, debug text %dW x %dH in characters."
	, stats->width
	, stats->height
	, stats->textWidth
	, stats->textHeight
	);
```
打印到内部调试文本字符缓冲区
_x , _y:  左上角 单位 字符
_attr: 调色板。其中前4位表示背景索引，底部4位表示标准VGA文本调色板（ANSI转义代码）的前景色。
_format: printf 风格的 格式化字符串. 可使用 ANSI转义代码

```cpp
//前进到下一帧。渲染线程将被踢到处理提交的渲染原语。
bgfx::frame();
return true;
```
