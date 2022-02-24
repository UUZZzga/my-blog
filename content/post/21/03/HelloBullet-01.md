+++
title = "HelloBullet 01"
date = "2021-03-23T22:59:48+08:00"
tags = ["c++","Bullet"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "Bullet 流程 分析"
+++

## 初始化
>
### 碰撞配置
```cpp
//默认 碰撞配置
btDefaultCollisionConfiguration* collisionConfiguration = new btDefaultCollisionConfiguration();
//使用默认 碰撞调度器
btCollisionDispatcher* dispatcher = new btCollisionDispatcher(collisionConfiguration);
```
碰撞调度器所在 src/CollisionDispatch
>
### Broad-Phase
```cpp
//通用的Broad-Phase
btBroadphaseInterface* overlappingPairCache = new btDbvtBroadphase();
```
https://zhuanlan.zhihu.com/p/113415779
如果把所有物体 两两计算碰撞 计算量 O(N^2)
Broad-Phase 就是把明显不相关的 计算排除掉
>
### 约束求解器
```cpp
//默认约束求解器
btSequentialImpulseConstraintSolver* solver = new btSequentialImpulseConstraintSolver;
```
约束求解器 src/BulletDynamics/ConstraintSolver
>
### 创建一个力学世界
```cpp
btDiscreteDynamicsWorld* dynamicsWorld = new btDiscreteDynamicsWorld(dispatcher, overlappingPairCache, solver, collisionConfiguration);
```
>
### 设置重力方向
```cpp
dynamicsWorld->setGravity(btVector3(0, -10, 0));
```
>
## 创建 刚体
>
### 初始化内存 管理工具
```cpp
btAlignedObjectArray<btCollisionShape*> collisionShapes;
```
>
### 创建一个立方体
```cpp
btCollisionShape* groundShape = new btBoxShape(btVector3(btScalar(50.), btScalar(50.), btScalar(50.)));
```
>
### 交由内存管理工具管理
```cpp
collisionShapes.push_back(groundShape);
```
>
### 创建 变换矩阵
```cpp
btTransform groundTransform;
groundTransform.setIdentity();
groundTransform.setOrigin(btVector3(0, -56, 0));
```
>
### 局部 惯量
```cpp
btVector3 localInertia(0, 0, 0);
if (isDynamic)
	groundShape->calculateLocalInertia(mass, localInertia);
```
只有有重量的物体才能是非静态、有局部惯量
>
### 运动轨迹 插值
```cpp
btDefaultMotionState* myMotionState = new btDefaultMotionState(groundTransform);
```
使用默认的插值 同步“活动”对象
>
### 创建刚体
```cpp
btRigidBody::btRigidBodyConstructionInfo rbInfo(mass, myMotionState, groundShape, localInertia);
btRigidBody* body = new btRigidBody(rbInfo);
```
>
### 将刚体添加到力学世界
```cpp
dynamicsWorld->addRigidBody(body);
```
>
## 模拟计算
>
### 步进模拟
```cpp
dynamicsWorld->stepSimulation(1.f / 60.f, 10);
```
头文件定义
```cpp
virtual int stepSimulation( btScalar timeStep,int maxSubSteps=1, btScalar fixedTimeStep=btScalar(1.)/btScalar(60.));
```
timeStep 剩余时间，maxSubSteps 最大执行步骤，fixedTimeStep 一步多长时间
>
### 得到模拟结果
```cpp

for(int i = 0;i < dynamicsWorld->getNumCollisionObjects(); ++i){
	btCollisionObject* obj = dynamicsWorld->getCollisionObjectArray()[i];
	btRigidBody* body = btRigidBody::upcast(obj);
	btTransform trans;
	if (body && body->getMotionState())
	{
		body->getMotionState()->getWorldTransform(trans);
	}
	else
	{
		trans = obj->getWorldTransform();
	}
	//trans 就是物体的 变换矩阵
}
```
得到具体坐标
```cpp
trans.getOrigin().getX();
trans.getOrigin().getY();
trans.getOrigin().getZ();
```
>
## 清理结果
>
### 从力学世界移除物体
```cpp
for (i = dynamicsWorld->getNumCollisionObjects() - 1; i >= 0; i--)
{
	btCollisionObject* obj = dynamicsWorld->getCollisionObjectArray()[i];
	btRigidBody* body = btRigidBody::upcast(obj);
	if (body && body->getMotionState())
	{
		delete body->getMotionState();
	}
	dynamicsWorld->removeCollisionObject(obj);
	delete obj;
}
```
>
### 清空内存管理器管理的物体
```cpp
for (int j = 0; j < collisionShapes.size(); j++)
{
	btCollisionShape* shape = collisionShapes[j];
	collisionShapes[j] = 0;
	delete shape;
}

collisionShapes.clear();
```
>
### 删除
```cpp
//删除力学世界
delete dynamicsWorld;

//删除解算器
delete solver;

//删除 broad-phase
delete overlappingPairCache;

//删除 调度器
delete dispatcher;

//删除 碰撞配置
delete collisionConfiguration;
```