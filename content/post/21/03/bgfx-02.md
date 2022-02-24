+++
title = "Bgfx 02"
date = "2021-03-30T16:36:40+08:00"
tags = ["c++","bgfx"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "Bgfx 示例 Cubes 分析"
+++

## PosColorVertex 类
### ms_layout 静态属性
```cpp
static bgfx::VertexLayout ms_layout;
```

### init 静态方法
```cpp
ms_layout
	.begin()
	.add(bgfx::Attrib::Position, 3, bgfx::AttribType::Float)
	.add(bgfx::Attrib::Color0,   4, bgfx::AttribType::Uint8, true)
	.end();
```
初始化 ms_layout
ms_layout 类型储存着 其他属性 的 类型顺序

### 其他属性
```cpp
float m_x;
float m_y;
float m_z;
uint32_t m_abgr;
```

## ExampleCubes 类
### init 方法
```cpp
// Create static vertex buffer.
m_vbh = bgfx::createVertexBuffer(
	// Static data can be passed with bgfx::makeRef
	  bgfx::makeRef(s_cubeVertices, sizeof(s_cubeVertices) )
	, PosColorVertex::ms_layout
	);
```
bgfx::makeRef 构造一个 指针
构造一个 vbh 数据 s_cubeVertices 长度 s_cubeVertices的长度 ，数组结构顺序 PosColorVertex::ms_layout
vbh 对应着opengl 的vbo

```cpp
// Create static index buffer for triangle list rendering.
m_ibh[0] = bgfx::createIndexBuffer(
	// Static data can be passed with bgfx::makeRef
	bgfx::makeRef(s_cubeTriList, sizeof(s_cubeTriList) )
	);

...

// Create static index buffer for point list rendering.
m_ibh[4] = bgfx::createIndexBuffer(
	// Static data can be passed with bgfx::makeRef
	bgfx::makeRef(s_cubePoints, sizeof(s_cubePoints) )
	);
```
储存着vbh的下标，用下标来代替vbh
ibh 对应着opengl 的EBO 

```cpp
// Create program from shaders.
m_program = loadProgram("vs_cubes", "fs_cubes");
```
创建并加载 着色器