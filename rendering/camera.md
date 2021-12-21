# 摄像机基础

## Camera
Camera 类中持有 CameraPositionerInterface 类的实例，用来切换不同的摄像机；
    #pragma once

    #include <assert.h>
    #include <algorithm>

    #include "shared/UtilsMath.h"
    #include "shared/Trackball.h"

    #include "glm/gtx/euler_angles.hpp"

    class CameraPositionerInterface
    {
    public:
    	virtual ~CameraPositionerInterface() = default;
    	virtual glm::mat4 getViewMatrix() const = 0;
    	virtual glm::vec3 getPosition() const = 0;
    };

    class Camera final
    {
    public:
    	explicit Camera(CameraPositionerInterface& positioner)
    		: positioner_(&positioner)
    	{}

    	Camera(const Camera&) = default;
    	Camera& operator = (const Camera&) = default;

    	glm::mat4 getViewMatrix() const { return positioner_->getViewMatrix(); }
    	glm::vec3 getPosition() const { return positioner_->getPosition(); }

    private:
    	const CameraPositionerInterface* positioner_;
    };

## CameraPositioner_FirstPerson
第一人称摄像机，最常使用的一种，下面来具体分析

    class CameraPositioner_FirstPerson final: public CameraPositionerInterface
    {
    public:
    	CameraPositioner_FirstPerson() = default;
    	CameraPositioner_FirstPerson(const glm::vec3& pos, const glm::vec3& target, const glm::vec3& up)
    	: cameraPosition_(pos)
    	, cameraOrientation_(glm::lookAt(pos, target, up))
    	, up_(up)
    	{}

    	void update(double deltaSeconds, const glm::vec2& mousePos, bool mousePressed)
    	{
    		if (mousePressed)
    		{
    			const glm::vec2 delta = mousePos - mousePos_;
    			const glm::quat deltaQuat = glm::quat(glm::vec3(-mouseSpeed_ * delta.y, mouseSpeed_ * delta.x, 0.0f));
    			cameraOrientation_ = deltaQuat * cameraOrientation_;
    			cameraOrientation_ = glm::normalize(cameraOrientation_);
    			setUpVector(up_);
    		}
    		mousePos_ = mousePos;

    		const glm::mat4 v = glm::mat4_cast(cameraOrientation_);

    		const glm::vec3 forward = -glm::vec3(v[0][2], v[1][2], v[2][2]);
    		const glm::vec3 right = glm::vec3(v[0][0], v[1][0], v[2][0]);
    		const glm::vec3 up = glm::cross(right, forward);

    		glm::vec3 accel(0.0f);

    		if (movement_.forward_) accel += forward;
    		if (movement_.backward_) accel -= forward;

    		if (movement_.left_) accel -= right;
    		if (movement_.right_) accel += right;

    		if (movement_.up_) accel += up;
    		if (movement_.down_) accel -= up;

    		if (movement_.fastSpeed_) accel *= fastCoef_;

    		if (accel == glm::vec3(0))
    		{
    			// decelerate naturally according to the damping value
    			moveSpeed_ -= moveSpeed_ * std::min((1.0f / damping_) * static_cast<float>(deltaSeconds), 1.0f);
    		}
    		else
    		{
    			// acceleration
    			moveSpeed_ += accel * acceleration_ * static_cast<float>(deltaSeconds);
    			const float maxSpeed = movement_.fastSpeed_ ? maxSpeed_ * fastCoef_ : maxSpeed_;
    			if (glm::length(moveSpeed_) > maxSpeed) moveSpeed_ = glm::normalize(moveSpeed_) * maxSpeed;
    		}

    		cameraPosition_ += moveSpeed_ * static_cast<float>(deltaSeconds);
    	}

    	virtual glm::mat4 getViewMatrix() const override
    	{
    		const glm::mat4 t = glm::translate(glm::mat4(1.0f), -cameraPosition_);
    		const glm::mat4 r = glm::mat4_cast(cameraOrientation_);
    		return r * t;
    	}

    	virtual glm::vec3 getPosition() const override
    	{
    		return cameraPosition_;
    	}

    	void setPosition(const glm::vec3& pos)
    	{
    		cameraPosition_ = pos;
    	}

    	void resetMousePosition(const glm::vec2& p) { mousePos_ = p; };

    	void setUpVector(const glm::vec3& up)
    	{
    		const glm::mat4 view = getViewMatrix();
    		const glm::vec3 dir = -glm::vec3(view[0][2], view[1][2], view[2][2]);
    		cameraOrientation_ = glm::lookAt(cameraPosition_, cameraPosition_ + dir, up);
    	}

    	inline void lookAt(const glm::vec3& pos, const glm::vec3& target, const glm::vec3& up) {
    		cameraPosition_ = pos;
    		cameraOrientation_ = glm::lookAt(pos, target, up);
    	}

    public:
    	struct Movement
    	{
    		bool forward_ = false;
    		bool backward_ = false;
    		bool left_ = false;
    		bool right_ = false;
    		bool up_ = false;
    		bool down_ = false;
    		//
    		bool fastSpeed_ = false;
    	} movement_;

    public:
    	float mouseSpeed_ = 4.0f;
    	float acceleration_ = 150.0f;
    	float damping_ = 0.2f;
    	float maxSpeed_ = 10.0f;
    	float fastCoef_ = 10.0f;

    private:
    	glm::vec2 mousePos_ = glm::vec2(0);
    	glm::vec3 cameraPosition_ = glm::vec3(0.0f, 10.0f, 10.0f);
    	glm::quat cameraOrientation_ = glm::quat(glm::vec3(0));
    	glm::vec3 moveSpeed_ = glm::vec3(0.0f);
    	glm::vec3 up_ = glm::vec3(0.0f, 0.0f, 1.0f);
    };

先看属性：  
1. 定义了自定义结构类型 Movement，用来和具体按下的按键解耦；
2. 定义了 mouseSpeed_ 等属性，用来控制摄像机的运动参数；
3. 定义了 cameraPosition_ 等内部属性，用来计算 ViewMatrix；
再看方法：
1. 构造函数

		CameraPositioner_FirstPerson(const glm::vec3& pos, const glm::vec3& target, const glm::vec3& up)
		: cameraPosition_(pos)
		, cameraOrientation_(glm::lookAt(pos, target, up))
		, up_(up)
		{}
	
	使用 glm::lookAt 函数，更新 cameraOrientation_ 摄像机朝向；
2. update，更新 camera 位置

		void update(double deltaSeconds, const glm::vec2& mousePos, bool mousePressed)
		{
			if (mousePressed)
			{
				const glm::vec2 delta = mousePos - mousePos_;
				const glm::quat deltaQuat = glm::quat(glm::vec3(-mouseSpeed_ * delta.y, mouseSpeed_ * delta.x, 0.0f));
				cameraOrientation_ = deltaQuat * cameraOrientation_;
				cameraOrientation_ = glm::normalize(cameraOrientation_);
				setUpVector(up_);
			}
			mousePos_ = mousePos;

			const glm::mat4 v = glm::mat4_cast(cameraOrientation_);

			const glm::vec3 forward = -glm::vec3(v[0][2], v[1][2], v[2][2]);
			const glm::vec3 right = glm::vec3(v[0][0], v[1][0], v[2][0]);
			const glm::vec3 up = glm::cross(right, forward);

			glm::vec3 accel(0.0f);

			if (movement_.forward_) accel += forward;
			if (movement_.backward_) accel -= forward;

			if (movement_.left_) accel -= right;
			if (movement_.right_) accel += right;

			if (movement_.up_) accel += up;
			if (movement_.down_) accel -= up;

			if (movement_.fastSpeed_) accel *= fastCoef_;

			if (accel == glm::vec3(0))
			{
				// decelerate naturally according to the damping value
				moveSpeed_ -= moveSpeed_ * std::min((1.0f / damping_) * static_cast<float>(deltaSeconds), 1.0f);
			}
			else
			{
				// acceleration
				moveSpeed_ += accel * acceleration_ * static_cast<float>(deltaSeconds);
				const float maxSpeed = movement_.fastSpeed_ ? maxSpeed_ * fastCoef_ : maxSpeed_;
				if (glm::length(moveSpeed_) > maxSpeed) moveSpeed_ = glm::normalize(moveSpeed_) * maxSpeed;
			}

			cameraPosition_ += moveSpeed_ * static_cast<float>(deltaSeconds);
		}
	update 方法每一帧都会被调用，传入帧与帧之间间隔的时间、当前鼠标的位置，以及鼠标是否被按下：
	1. 当鼠标被按下时，会根据当前传入的鼠标位置，与上一帧之间的插值，构造旋转用的四元数，这里有三点需要注意：
		1. 四元数的构造函数 glm::quat(angle.x, angle.y, angle.z)，这里需要分别传入绕 x 轴旋转的角度、绕 y 轴旋转的角度和绕 z 轴旋转的角度，注意这里需要传入 radiance ，也就是弧度值，另外需要引用欧拉角的头文件 `#include "glm/gtx/euler_angles.hpp"`；
		2. 四元数乘以一个三维向量，直接就是得到该三维向量旋转后的结果，这里的乘法运算被重载了；

				//rotate vector
				vec3 qrot(vec4 q, vec3 v)
				{
				    return v + 2.0*cross(q.xyz, cross(q.xyz,v) + q.w*v);
				}
				//rotate vector (alternative)
				vec3 qrot_2(vec4 q, vec3 v)
				{
				    return v*(q.w*q.w - dot(q.xyz,q.xyz)) + 2.0*q.xyz*dot(q.xyz,v) +    
				          2.0*q.w*cross(q.xyz,v);
				}

		3. 四元数乘以一个四元数，得到的是两个四元数旋转合并后的四元数；
	2. 调用 setUpVector，保证摄像机不会出现上下翻转的情况；
	3. 更新鼠标位置；
	4. 通过 glm::mat4_cast 获取旋转矩阵，根据旋转矩阵，解析出 forword、up、right 方向；
	5. 下面一段是模拟加减速的过程，不再赘述；

综上，一个基本的 Camera 类就配置完成了，下面我们看一下应该怎么使用：
1. 首先，声明 Camera 与具体实现类，并且保存当前的鼠标状态；

		CameraPositioner_FirstPerson positioner( vec3(0.0f), vec3(0.0f, 0.0f, -1.0f), vec3(0.0f, 1.0f, 0.0f));	
		//positioner.setPosition(vec3(0.0f, 0.0f, 0.0f));  
		Camera camera(positioner);  

		struct MouseState  
		{  
			glm::vec2 pos = glm::vec2(0.0f);  
			bool pressedLeft = false;  
		} mouseState;

2. 设置监听窗口鼠标事件的回调函数，更新鼠标状态

		glfwSetCursorPosCallback(
			window,
			[](auto* window, double x, double y)
			{
				int width, height;
				glfwGetFramebufferSize(window, &width, &height);
				mouseState.pos.x = static_cast<float>(x / width);
				mouseState.pos.y = static_cast<float>(y / height);
			}
		);

		glfwSetMouseButtonCallback(
			window,
			[](auto* window, int button, int action, int mods)
			{
				if (button == GLFW_MOUSE_BUTTON_LEFT)
					mouseState.pressedLeft = action == GLFW_PRESS;
			}
		);

3. 设置监听按键的回调函数，更新摄像机的前进方向

		glfwSetKeyCallback(
			window,
			[](GLFWwindow* window, int key, int scancode, int action, int mods)
			{
				const bool pressed = action != GLFW_RELEASE;
				if (key == GLFW_KEY_ESCAPE && pressed)
					glfwSetWindowShouldClose(window, GLFW_TRUE);
				if (key == GLFW_KEY_W)
					positioner.movement_.forward_ = pressed;
				if (key == GLFW_KEY_S)
					positioner.movement_.backward_ = pressed;
				if (key == GLFW_KEY_A)
					positioner.movement_.left_ = pressed;
				if (key == GLFW_KEY_D)
					positioner.movement_.right_ = pressed;
				if (key == GLFW_KEY_1)
					positioner.movement_.up_ = pressed;
				if (key == GLFW_KEY_2)
					positioner.movement_.down_ = pressed;
				if (mods & GLFW_MOD_SHIFT)
					positioner.movement_.fastSpeed_ = pressed;
				if (key == GLFW_KEY_SPACE)
					positioner.setUpVector(vec3(0.0f, 1.0f, 0.0f));
			}
		);
	
4. 在渲染循环的开始，更新摄像机的状态

		while (!glfwWindowShouldClose(window))
		{
			positioner.update(deltaSeconds, mouseState.pos, mouseState.pressedLeft);
		
5. 获取更新后的视图矩阵，并更新 Uniform 变量

		const mat4 p = glm::perspective(45.0f, ratio, 0.1f, 1000.0f);
		const mat4 view = camera.getViewMatrix();

		const PerFrameData perFrameData = { .view = view, .proj = p, .cameraPos = glm::vec4(camera.getPosition(), 1.0f) };
		glNamedBufferSubData(perFrameDataBuffer, 0, kUniformBufferSize, &perFrameData);
	
	
