+++
title = "HelloBullet 02"
date = "2021-03-25T16:49:26+08:00"
tags = ["c++","Bullet"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "uzzzzz"
description = "Bullet 示例 AppBasicExample/AppBasicExampleGui 分析"
+++

## 物理计算 类的关系
>
### 抽象类 CommonExampleInterface
>
#### 初始化 和 结束
```cpp
virtual void initPhysics() = 0;
virtual void exitPhysics() = 0;
```
>
#### 计算物理
```cpp
virtual void stepSimulation(float deltaTime) = 0;
```
>
#### 图像GUI相关
```cpp
virtual void updateGraphics() {}
virtual void renderScene() = 0;
```
>
窗口回执
```cpp
virtual bool mouseMoveCallback(float x, float y) = 0;
virtual bool mouseButtonCallback(int button, int state, float x, float y) = 0;
virtual bool keyboardCallback(int key, int state) = 0;
```
>
#### debug相关
```cpp
virtual void physicsDebugDraw(int debugFlags) = 0;  //目前，我们在 Bullet/src/LinearMath/btIDebugDraw.h
//只有在切换演示时才调用 resetCamera。这样你可以重新启动（initPhysics）并更容易地在特定的位置观看
virtual void resetCamera(){};
```
>
#### vr相关
```cpp
virtual void vrControllerMoveCallback(int controllerId, float pos[4], float orientation[4], float analogAxis, float auxAnalogAxes[10]) {}
virtual void vrControllerButtonCallback(int controllerId, int button, int state, float pos[4], float orientation[4]) {}
virtual void vrHMDMoveCallback(int controllerId, float pos[4], float orientation[4]) {}
virtual void vrGenericTrackerMoveCallback(int controllerId, float pos[4], float orientation[4]) {}
```
>
#### 处理命令行参数
```cpp
virtual void processCommandLineArgs(int argc, char* argv[]){};
```
>
#### 自定义 类的接口
```cpp
typedef class CommonExampleInterface*(CreateFunc)(CommonExampleOptions& options);
```
满足这个类型即可
>
### CommonRigidBodyBase 共有基类
>
#### 属性
```cpp
//使用内存管理工具 管理 btCollisionShape指针 物体的物理形状
btAlignedObjectArray<btCollisionShape*> m_collisionShapes;

//Broad-Phase
btBroadphaseInterface* m_broadphase;

//碰撞调度器
btCollisionDispatcher* m_dispatcher;

//约束解算器
btConstraintSolver* m_solver;

//碰撞配置
btDefaultCollisionConfiguration* m_collisionConfiguration;

//力学世界
btDiscreteDynamicsWorld* m_dynamicsWorld;
```
>
#### 观察视角相关
##### 属性
```cpp
//data for picking objects
class btRigidBody* m_pickedBody;
class btTypedConstraint* m_pickedConstraint;
int m_savedState;
btVector3 m_oldPickingPos;
btVector3 m_hitPos;
btScalar m_oldPickingDist;
```
##### 方法
1. mouseMoveCallback 鼠标移动回执
```cpp
virtual bool mouseMoveCallback(float x, float y)
{
	CommonRenderInterface* renderer = m_guiHelper->getRenderInterface();

	if (!renderer)
		{
		btAssert(0);
		return false;
	}

	btVector3 rayTo = getRayTo(int(x), int(y));
	btVector3 rayFrom;
	renderer->getActiveCamera()->getCameraPosition(rayFrom);
	movePickedBody(rayFrom, rayTo);

	return false;
}
```
执行getRayTo 把屏幕坐标转换成世界坐标也就是移动的目标点，通过gui的输入工具,获取相机内的物体的坐标也就是原来的坐标 执行 movePickedBody(rayFrom, rayTo); 移动物体
2. movePickedBody
```cpp
virtual bool movePickedBody(const btVector3& rayFromWorld, const btVector3& rayToWorld)
{
	if (m_pickedBody && m_pickedConstraint)
	{
		btPoint2PointConstraint* pickCon = static_cast<btPoint2PointConstraint*>(m_pickedConstraint);
		if (pickCon)
		{
			//keep it at the same picking distance

			btVector3 newPivotB;

			btVector3 dir = rayToWorld - rayFromWorld;
			dir.normalize();
			dir *= m_oldPickingDist;

			newPivotB = rayFromWorld + dir;
			pickCon->setPivotB(newPivotB);
			return true;
		}
	}
	return false;
}
```
如果选择物体 移动选择物体。rayFromWorld原来的坐标，rayToWorld目标坐标，dir移动的方向，normalize坐标和原点的线段方向不变长度为1，m_oldPickingDist 为移动距离的长度，setPivotB设置关节在第2个刚体坐标系中的位置
3. mouseButtonCallback 鼠标点击回执
```cpp
virtual bool mouseButtonCallback(int button, int state, float x, float y)
{
	CommonRenderInterface* renderer = m_guiHelper->getRenderInterface();

	if (!renderer)
	{
		btAssert(0);
		return false;
	}

	CommonWindowInterface* window = m_guiHelper->getAppInterface()->m_window;

// ...

	if (state == 1)
	{
		if (button == 0 && (!window->isModifierKeyPressed(B3G_ALT) && !window->isModifierKeyPressed(B3G_CONTROL)))
		{
			btVector3 camPos;
			renderer->getActiveCamera()->getCameraPosition(camPos);

			btVector3 rayFrom = camPos;
			btVector3 rayTo = getRayTo(int(x), int(y));

			pickBody(rayFrom, rayTo);
		}
	}
	else
	{
		if (button == 0)
		{
			removePickingConstraint();
			//remove p2p
		}
	}

	//printf("button=%d, state=%d\n",button,state);
	return false;
}
```
button 按钮，state状态。按下左键时，获取相机指向的坐标，窗口坐标转换成世界坐标，执行pickBody
4. pickBody 
```cpp
virtual bool pickBody(const btVector3& rayFromWorld, const btVector3& rayToWorld)
{
	if (m_dynamicsWorld == 0)
		return false;

	btCollisionWorld::ClosestRayResultCallback rayCallback(rayFromWorld, rayToWorld);

	rayCallback.m_flags |= btTriangleRaycastCallback::kF_UseGjkConvexCastRaytest;
	m_dynamicsWorld->rayTest(rayFromWorld, rayToWorld, rayCallback);
	if (rayCallback.hasHit())
	{
		btVector3 pickPos = rayCallback.m_hitPointWorld;
		btRigidBody* body = (btRigidBody*)btRigidBody::upcast(rayCallback.m_collisionObject);
		if (body)
		{
			//other exclusions?
			if (!(body->isStaticObject() || body->isKinematicObject()))
			{
				m_pickedBody = body;
				m_savedState = m_pickedBody->getActivationState();
				m_pickedBody->setActivationState(DISABLE_DEACTIVATION);
				//printf("pickPos=%f,%f,%f\n",pickPos.getX(),pickPos.getY(),pickPos.getZ());
				btVector3 localPivot = body->getCenterOfMassTransform().inverse() * pickPos;
				btPoint2PointConstraint* p2p = new btPoint2PointConstraint(*body, localPivot);
				m_dynamicsWorld->addConstraint(p2p, true);
				m_pickedConstraint = p2p;
				btScalar mousePickClamping = 30.f;
				p2p->m_setting.m_impulseClamp = mousePickClamping;
				//very weak constraint for picking
				p2p->m_setting.m_tau = 0.001f;
			}
		}

		m_oldPickingPos = rayToWorld;
		m_hitPos = pickPos;
		m_oldPickingDist = (pickPos - rayFromWorld).length();
	}
	return false;
}
```
ClosestRayResultCallback 用于获取鼠标指向最近物体的工具，设置flag 使用 凸面射线 gjk算法，
`m_dynamicsWorld->rayTest(rayFromWorld, rayToWorld, rayCallback);`
使用rayTest尝试获取 最近物体的工具，rayCallback.hasHit() 判断是否存在，
`btVector3 pickPos = rayCallback.m_hitPointWorld;`
获取选中 相对于物体原点的位置，
`btRigidBody* body = (btRigidBody*)btRigidBody::upcast(rayCallback.m_collisionObject);`
尝试将父类btCollisionObject 转换成子类btRigidBody upcast内部是用状态机和c风格转换实现的，
转换成功后 排除 静态物体（没有质量）和 KinematicObject （其实他有isStaticOrKinematicObject方法），
设置选中物体，保存物体原来状态，更改物体状态为 DISABLE_DEACTIVATION 失去活动 ，获取重心的变换矩阵 的反向 乘 pickPos 矩阵 点乘 向量，
添加一个 点到点的约束 给 世界里的 被选中物体，设置夹住的效果 给约束 力量为 30，给选中物体非常弱的约束
>
5. removePickingConstraint
```cpp
virtual void removePickingConstraint()
{
	if (m_pickedConstraint)
	{
		m_pickedBody->forceActivationState(m_savedState);
		m_pickedBody->activate();
		m_dynamicsWorld->removeConstraint(m_pickedConstraint);
		delete m_pickedConstraint;
		m_pickedConstraint = 0;
		m_pickedBody = 0;
	}
}
```
如果夹住的约束存在，恢复物体状态，激活，移除约束，释放 约束占用的内存，清空夹住的物体指针
#### 物理引擎相关
>
##### 方法
1. createEmptyDynamicsWorld
```cpp
///collision configuration contains default setup for memory, collision setup
m_collisionConfiguration = new btDefaultCollisionConfiguration();
//m_collisionConfiguration->setConvexConvexMultipointIterations();

///use the default collision dispatcher. For parallel processing you can use a diffent dispatcher (see Extras/BulletMultiThreaded)
m_dispatcher = new btCollisionDispatcher(m_collisionConfiguration);

m_broadphase = new btDbvtBroadphase();

///the default constraint solver. For parallel processing you can use a different solver (see Extras/BulletMultiThreaded)
btSequentialImpulseConstraintSolver* sol = new btSequentialImpulseConstraintSolver;
m_solver = sol;

m_dynamicsWorld = new btDiscreteDynamicsWorld(m_dispatcher, m_broadphase, m_solver, m_collisionConfiguration);

m_dynamicsWorld->setGravity(btVector3(0, -10, 0));
```
默认 碰撞配置
默认 碰撞调度器
通用的Broad-Phase
默认 约束求解器
创建一个力学世界
设置重力方向
2. createBoxShape
```cpp
btBoxShape* box = new btBoxShape(halfExtents);
return box;
```
3. createRigidBody
```cpp
btRigidBody* createRigidBody(float mass, const btTransform& startTransform, btCollisionShape* shape, const btVector4& color = btVector4(1, 0, 0, 1))
{
	btAssert((!shape || shape->getShapeType() != INVALID_SHAPE_PROXYTYPE));

	//rigidbody is dynamic if and only if mass is non zero, otherwise static
	bool isDynamic = (mass != 0.f);

	btVector3 localInertia(0, 0, 0);
	if (isDynamic)
		shape->calculateLocalInertia(mass, localInertia);

	//using motionstate is recommended, it provides interpolation capabilities, and only synchronizes 'active' objects

	btDefaultMotionState* myMotionState = new btDefaultMotionState(startTransform);

	btRigidBody::btRigidBodyConstructionInfo cInfo(mass, myMotionState, shape, localInertia);

	btRigidBody* body = new btRigidBody(cInfo);
	//body->setContactProcessingThreshold(m_dfaultContactProcessingThre

	body->setUserIndex(-1);
	m_dynamicsWorld->addRigidBody(body);
	return body;
}
```

4. deleteRigidBody
```cpp
void deleteRigidBody(btRigidBody* body)
{
	int graphicsUid = body->getUserIndex();
	m_guiHelper->removeGraphicsInstance(graphicsUid);

	m_dynamicsWorld->removeRigidBody(body);
	btMotionState* ms = body->getMotionState();
	delete body;
	delete ms;
}
```
5. exitPhysics
```cpp
virtual void exitPhysics()
{
	removePickingConstraint();
	//cleanup in the reverse order of creation/initialization

	//remove the rigidbodies from the dynamics world and delete them

	if (m_dynamicsWorld)
	{
		int i;
		for (i = m_dynamicsWorld->getNumConstraints() - 1; i >= 0; i--)
		{
			m_dynamicsWorld->removeConstraint(m_dynamicsWorld->getConstraint(i));
		}
		for (i = m_dynamicsWorld->getNumCollisionObjects() - 1; i >= 0; i--)
		{
			btCollisionObject* obj = m_dynamicsWorld->getCollisionObjectArray()[i];
			btRigidBody* body = btRigidBody::upcast(obj);
			if (body && body->getMotionState())
			{
				delete body->getMotionState();
			}
			m_dynamicsWorld->removeCollisionObject(obj);
			delete obj;
		}
	}
	//delete collision shapes
	for (int j = 0; j < m_collisionShapes.size(); j++)
	{
		btCollisionShape* shape = m_collisionShapes[j];
		delete shape;
	}
	m_collisionShapes.clear();

	delete m_dynamicsWorld;
	m_dynamicsWorld = 0;

	delete m_solver;
	m_solver = 0;

	delete m_broadphase;
	m_broadphase = 0;

	delete m_dispatcher;
	m_dispatcher = 0;

	delete m_collisionConfiguration;
	m_collisionConfiguration = 0;
}
```
逆序移除约束
逆序移除碰撞对象 有使用MotionState的话 移除 MotionState
清空内存管理器 m_collisionShapes 管理的物体
删除世界
。。。
>
### BasicExample 应用
#### 方法
##### initPhysics
设置高度轴
```cpp
m_guiHelper->setUpAxis(1);
```
1y 2z,只支持Y Z

```cpp
createEmptyDynamicsWorld();
```
执行 createEmptyDynamicsWorld(); 初始化

```cpp
//m_dynamicsWorld->setGravity(btVector3(0,0,0));
m_guiHelper->createPhysicsDebugDrawer(m_dynamicsWorld);

if (m_dynamicsWorld->getDebugDrawer())
	m_dynamicsWorld->getDebugDrawer()->setDebugMode(btIDebugDraw::DBG_DrawWireframe + btIDebugDraw::DBG_DrawContactPoints);
```
设置debug

```cpp
///create a few basic rigid bodies
btBoxShape* groundShape = createBoxShape(btVector3(btScalar(50.), btScalar(50.), btScalar(50.)));

//groundShape->initializePolyhedralFeatures();
//btCollisionShape* groundShape = new btStaticPlaneShape(btVector3(0,1,0),50);

m_collisionShapes.push_back(groundShape);

btTransform groundTransform;
groundTransform.setIdentity();
groundTransform.setOrigin(btVector3(0, -50, 0));

{
	btScalar mass(0.);
	createRigidBody(mass, groundTransform, groundShape, btVector4(0, 0, 1, 1));
}
```
创建 地面

```cpp
{
	//create a few dynamic rigidbodies
	// Re-using the same collision is better for memory usage and performance

	btBoxShape* colShape = createBoxShape(btVector3(.1, .1, .1));

	//btCollisionShape* colShape = new btSphereShape(btScalar(1.));
	m_collisionShapes.push_back(colShape);

	/// Create Dynamic Objects
	btTransform startTransform;
	startTransform.setIdentity();

	btScalar mass(1.f);

	//rigidbody is dynamic if and only if mass is non zero, otherwise static
	bool isDynamic = (mass != 0.f);

	btVector3 localInertia(0, 0, 0);
	if (isDynamic)
		colShape->calculateLocalInertia(mass, localInertia);

	for (int k = 0; k < ARRAY_SIZE_Y; k++)
	{
		for (int i = 0; i < ARRAY_SIZE_X; i++)
		{
			for (int j = 0; j < ARRAY_SIZE_Z; j++)
			{
				startTransform.setOrigin(btVector3(
					btScalar(0.2 * i),
					btScalar(2 + .2 * k),
					btScalar(0.2 * j)));

				createRigidBody(mass, startTransform, colShape);
			}
		}
	}
}

m_guiHelper->autogenerateGraphicsObjects(m_dynamicsWorld);
```
创建5x5x5 的方块，使用 autogenerateGraphicsObjects 把力学世界的 物体 生成 显示对象
##### renderScene
```cpp
void BasicExample::renderScene()
{
	CommonRigidBodyBase::renderScene();
}
```

##### resetCamera
```cpp
void resetCamera()
{
	float dist = 4;
	float pitch = -35;
	float yaw = 52;
	float targetPos[3] = {0, 0, 0};
	m_guiHelper->resetCamera(dist, yaw, pitch, targetPos[0], targetPos[1], targetPos[2]);
}
```
设置相机所在位置 看向位置
>
## 图形相关 类的关系
### GUIHelperInterface 抽象类
### OpenGLGuiHelper 实现