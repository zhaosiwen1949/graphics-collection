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
	
	
