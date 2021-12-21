# FPS

本节件数如何构造一个计算 FPS 的类
## 类声明

        class FramesPerSecondCounter
        {
        public:
        	explicit FramesPerSecondCounter(float avgInterval = 0.5f)
        	:avgInterval_(avgInterval)
        	{
        		assert(avgInterval > 0.0f);
        	}

        	bool tick(float deltaSeconds, bool frameRendered = true)
        	{
        		if (frameRendered)
        			numFrames_++;

        		accumulatedTime_ += deltaSeconds;

        		if (accumulatedTime_ > avgInterval_)
        		{
        			currentFPS_ = static_cast<float>(numFrames_ / accumulatedTime_);
        			if (printFPS_)
        				printf("FPS: %.1f\n", currentFPS_);
        			numFrames_ = 0;
        			accumulatedTime_ = 0;
        			return true;
        		}

        		return false;
        	}

        	inline float getFPS() const { return currentFPS_; }

        	bool printFPS_ = true;

        private:
        	const float avgInterval_ = 0.5f;
        	unsigned int numFrames_ = 0;
        	double accumulatedTime_ = 0;
        	float currentFPS_ = 0.0f;
        };

    这里主要说明下几个私有属性的意义：
    1. avgInterval_，统计时间窗口，默认 0.5s 统计一次；
    2. numFrames_，统计窗口内，当前绘制了多少帧；
    3. accumulatedTime_，统计窗口内，当前已经过了多少时间；
    4. currentFPS_，计算出来的 FPS 是多少；

    对于 tick 方法，主要是：
    1. 累加 numFrames_ 窗口内的渲染帧数；
    2. 累加 accumulatedTime_ 窗口渲染的时间；
    3. 如果 accumulatedTime_ 超过了窗口的大小，则计算 FPS，并更新 currentFPS_，同时重置 numFrames_、accumulatedTime_，进入下一次计算周期；

## 如何使用
1. 声明好当前时间、时间间隔和 FPS 类

        double timeStamp = glfwGetTime();
	    float deltaSeconds = 0.0f;

	    FramesPerSecondCounter fpsCounter(0.5f);

2. 在主循环中，更新时间间隔，并调用 tick 方法

        while (!glfwWindowShouldClose(window))
	    {
            fpsCounter.tick(deltaSeconds);
    
		    positioner.update(deltaSeconds, mouseState.pos, mouseState.pressedLeft);
    
		    const double newTimeStamp = glfwGetTime();
		    deltaSeconds = static_cast<float>(newTimeStamp - timeStamp);
		    timeStamp = newTimeStamp;
    