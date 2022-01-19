# 引擎架构之应用封装

## 整体架构
MyApp 继承 CameraApp，CameraApp 继承 VulkanApp，这三者的作用各有不同：
1. VulkanApp 封装了窗口以及 Vulkan 的公共资源，包括 device、queue、commond 和 swapchain 这些绘制中基本不变的基础设施，并将绘制的过程进行封装，具体定制部分，由外面传入的 Renderer 负责；
2. CameraApp 在 Vulkan 的基础上提供了 Camera 的实现；
3. MyApp 的作用在于，将需要的 Renderer 装入 VulkanApp 的绘制过程中，其中主要使用了 MultiRenderer（具体分析见[引擎架构之 Vulkan Renderer 封装](engine/vulkanRenderer.md)）

## 源码分析 - VulkanApp

        struct RenderItem {
        	Renderer& renderer_;
        	bool enabled_ = true;
        	bool useDepth_ = true;
        	explicit RenderItem(Renderer& r, bool useDepth = true)
        	: renderer_(r)
        	, useDepth_(useDepth)
        	{}
        };

        struct VulkanRenderContext
        {
        	VulkanInstance vk;
        	VulkanRenderDevice vkDev;
        	VulkanContextCreator ctxCreator;
        	VulkanResources resources;

        	VulkanRenderContext(void* window, uint32_t screenWidth, uint32_t screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures()):
        		ctxCreator(vk, vkDev, window, screenWidth, screenHeight, ctxFeatures),
        		resources(vkDev),

        		depthTexture(resources.addDepthTexture(vkDev.framebufferWidth, vkDev.framebufferHeight, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL)),

        		screenRenderPass(resources.addFullScreenPass()),
        		screenRenderPass_NoDepth(resources.addFullScreenPass(false)),

        		finalRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ = false, .clearDepth_ = false, .flags_ = eRenderPassBit_Last  })),
        		clearRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ =  true, .clearDepth_ =  true, .flags_ = eRenderPassBit_First })),

        		swapchainFramebuffers(resources.addFramebuffers(screenRenderPass.handle, depthTexture.image.imageView)),
        		swapchainFramebuffers_NoDepth(resources.addFramebuffers(screenRenderPass_NoDepth.handle))
        	{
        	}

        	void updateBuffers(uint32_t imageIndex);
        	void composeFrame(VkCommandBuffer commandBuffer, uint32_t imageIndex);

        	// For Chapter 8 & 9
        	inline PipelineInfo pipelineParametersForOutputs(const std::vector<VulkanTexture>& outputs) const {
        		return PipelineInfo {
        			.width = outputs.empty() ? vkDev.framebufferWidth : outputs[0].width,
        			.height = outputs.empty() ? vkDev.framebufferHeight : outputs[0].height,
        			.useBlending = false
        		};
        	}

        	std::vector<RenderItem> onScreenRenderers_;

        	VulkanTexture depthTexture;

        	// Framebuffers and renderpass for on-screen rendering
        	RenderPass screenRenderPass;
        	RenderPass screenRenderPass_NoDepth;

        	RenderPass clearRenderPass;
        	RenderPass finalRenderPass;

        	std::vector<VkFramebuffer> swapchainFramebuffers;
        	std::vector<VkFramebuffer> swapchainFramebuffers_NoDepth;

        	void beginRenderPass(VkCommandBuffer cmdBuffer, VkRenderPass pass, size_t currentImage, const VkRect2D area,
        		VkFramebuffer fb = VK_NULL_HANDLE,
        		uint32_t clearValueCount = 0, const VkClearValue* clearValues = nullptr)
        	{
        		const VkRenderPassBeginInfo renderPassInfo = {
        			.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        			.renderPass = pass,
        			.framebuffer = (fb != VK_NULL_HANDLE) ? fb : swapchainFramebuffers[currentImage],
        			.renderArea = area,
        			.clearValueCount = clearValueCount,
        			.pClearValues = clearValues
        		};

        		vkCmdBeginRenderPass( cmdBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE );
        	}
        };

        struct VulkanApp
        {
        	VulkanApp(int screenWidth, int screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures())
        		: window_(initVulkanApp(screenWidth, screenHeight, &resolution_)),
        		ctx_(window_, resolution_.width, resolution_.height, ctxFeatures),
        		onScreenRenderers_(ctx_.onScreenRenderers_)
        	{
        		glfwSetWindowUserPointer(window_, this);
        		assignCallbacks();
        	}

        	~VulkanApp()
        	{
        		glslang_finalize_process();
        		glfwTerminate();
        	}

            // 虚函数，在每帧更新的 updateBuffers 函数中调用
        	virtual void drawUI() {}
        	virtual void draw3D() = 0;

            // 主循环
        	void mainLoop();

        	// Check if none of the ImGui widgets were touched so our app can process mouse events
        	inline bool shouldHandleMouse() const { return !ImGui::GetIO().WantCaptureMouse; }

            // 鼠标与按键回调
        	virtual void handleKey(int key, bool pressed) = 0;
        	virtual void handleMouseClick(int button, bool pressed)
        	{
        		if (button == GLFW_MOUSE_BUTTON_LEFT)
        			mouseState_.pressedLeft = pressed;
        	}

        	virtual void handleMouseMove(float mx, float my)
        	{
        		mouseState_.pos.x = mx;
        		mouseState_.pos.y = my;
        	}

            // 虚函数，每帧更新的时候调用，会传入每帧之间的间隔
        	virtual void update(float deltaSeconds) = 0;

        	inline float getFPS() const { return fpsCounter_.getFPS(); }

        protected:
        	struct MouseState
        	{
        		glm::vec2 pos = glm::vec2(0.0f);
        		bool pressedLeft = false;
        	} mouseState_;

        	Resolution resolution_;
        	GLFWwindow* window_ = nullptr;
        	VulkanRenderContext ctx_;
        	std::vector<RenderItem>& onScreenRenderers_;
        	FramesPerSecondCounter fpsCounter_;

        private:
        	void assignCallbacks();

        	void updateBuffers(uint32_t imageIndex);
        };

VulkanApp 主要分为以下几部分：  
1. `initVulkanApp` 初始化 `window` 变量；
2. `VulkanRenderContext ctx_` 初始化 Vulkan 上下文相关部分；
3. `mainloop` 函数，进入渲染循环；  

下面逐一分析

### initVulkanApp

        GLFWwindow* initVulkanApp(int width, int height, Resolution* resolution)
        {
        	glslang_initialize_process();

        	volkInitialize();

            // 1、初始化 GLFW
        	if (!glfwInit())
        		exit(EXIT_FAILURE);

        	if (!glfwVulkanSupported())
        		exit(EXIT_FAILURE);

        	glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
        	glfwWindowHint(GLFW_RESIZABLE, GL_FALSE);

            // 2、设置分辨率
        	if (resolution)
        	{
        		*resolution = detectResolution(width, height);
        		width  = resolution->width;
        		height = resolution->height;
        	}

            // 3、 创建窗口
        	GLFWwindow* result = glfwCreateWindow(width, height, "VulkanApp", nullptr, nullptr);
        	if (!result)
        	{
        		glfwTerminate();
        		exit(EXIT_FAILURE);
        	}

        	return result;
        }

`initVulkanApp` 函数用来初始化 GLFW 的窗口对象，主要做了以下几件事：
1. `glslang_initialize_process();` 初始化 glslang 工具；
2. `volkInitialize();` 初始化 volk 库；
3. `glfwInit()` 初始化 GLFW 库；
4. 检测分辨率；
5. 创建窗口；

### VulkanRenderContext

        struct VulkanRenderContext
        {
        	VulkanInstance vk;
        	VulkanRenderDevice vkDev;
        	VulkanContextCreator ctxCreator;
        	VulkanResources resources;

        	VulkanRenderContext(void* window, uint32_t screenWidth, uint32_t screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures()):
        		ctxCreator(vk, vkDev, window, screenWidth, screenHeight, ctxFeatures),
        		resources(vkDev),

        		depthTexture(resources.addDepthTexture(vkDev.framebufferWidth, vkDev.framebufferHeight, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL)),

        		screenRenderPass(resources.addFullScreenPass()),
        		screenRenderPass_NoDepth(resources.addFullScreenPass(false)),

        		finalRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ = false, .clearDepth_ = false, .flags_ = eRenderPassBit_Last  })),
        		clearRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ =  true, .clearDepth_ =  true, .flags_ = eRenderPassBit_First })),

        		swapchainFramebuffers(resources.addFramebuffers(screenRenderPass.handle, depthTexture.image.imageView)),
        		swapchainFramebuffers_NoDepth(resources.addFramebuffers(screenRenderPass_NoDepth.handle))
        	{
        	}

        	void updateBuffers(uint32_t imageIndex);
        	void composeFrame(VkCommandBuffer commandBuffer, uint32_t imageIndex);

        	// For Chapter 8 & 9
        	inline PipelineInfo pipelineParametersForOutputs(const std::vector<VulkanTexture>& outputs) const {
        		return PipelineInfo {
        			.width = outputs.empty() ? vkDev.framebufferWidth : outputs[0].width,
        			.height = outputs.empty() ? vkDev.framebufferHeight : outputs[0].height,
        			.useBlending = false
        		};
        	}

        	std::vector<RenderItem> onScreenRenderers_;

        	VulkanTexture depthTexture;

        	// Framebuffers and renderpass for on-screen rendering
        	RenderPass screenRenderPass;
        	RenderPass screenRenderPass_NoDepth;

        	RenderPass clearRenderPass;
        	RenderPass finalRenderPass;

        	std::vector<VkFramebuffer> swapchainFramebuffers;
        	std::vector<VkFramebuffer> swapchainFramebuffers_NoDepth;

        	void beginRenderPass(VkCommandBuffer cmdBuffer, VkRenderPass pass, size_t currentImage, const VkRect2D area,
        		VkFramebuffer fb = VK_NULL_HANDLE,
        		uint32_t clearValueCount = 0, const VkClearValue* clearValues = nullptr)
        	{
        		const VkRenderPassBeginInfo renderPassInfo = {
        			.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        			.renderPass = pass,
        			.framebuffer = (fb != VK_NULL_HANDLE) ? fb : swapchainFramebuffers[currentImage],
        			.renderArea = area,
        			.clearValueCount = clearValueCount,
        			.pClearValues = clearValues
        		};

        		vkCmdBeginRenderPass( cmdBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE );
        	}
        };

先看一下 `VulkanRenderContext` 的属性：
1. `VulkanInstance vk;` 类型定义如下，主要是存储 Vulkan 实例以及相关的扩展：

        struct VulkanInstance final
        {
        	VkInstance instance;
        	VkSurfaceKHR surface;
        	VkDebugUtilsMessengerEXT messenger;
        	VkDebugReportCallbackEXT reportCallback;
        };
    
    包括 Vulkan 实例、Surface 表面对象、调试信息回调的对象等；
2. `VulkanRenderDevice vkDev;` 类型定义如下，主要存储整个 APP 都会用的 Vulkan 对象，主要是 device 设备、swapchain 交换链、queue 提交队列和 command 命令缓冲：

        struct VulkanRenderDevice final
        {
        	// framebuffer 的宽高
            uint32_t framebufferWidth; 
        	uint32_t framebufferHeight;
            
        	VkDevice device; // Vulkan Device 接口
        	VkQueue graphicsQueue; // Vulkan 的图像队列
        	VkPhysicalDevice physicalDevice; // Vulkan  PhysicalDevice 接口

        	uint32_t graphicsFamily; // Vulkan 的图像队列索引

        	VkSwapchainKHR swapchain; // 图像的交换链
        	VkSemaphore semaphore; // 信号量
        	VkSemaphore renderSemaphore; // 信号量

            // 交换链中的图像对应的 Image 对象和 ImageView 对象
        	std::vector<VkImage> swapchainImages;
        	std::vector<VkImageView> swapchainImageViews;

            // 命令缓冲和缓冲池
        	VkCommandPool commandPool;
        	std::vector<VkCommandBuffer> commandBuffers;

        	// For chapter5/6etc (compute shaders)

        	// Were we initialized with compute capabilities
        	bool useCompute = false;
            
            // 命令队列
        	// [may coincide with graphicsFamily]
        	uint32_t computeFamily;
        	VkQueue computeQueue;

            // 所有队列
        	// a list of all queues (for shared buffer allocation)
        	std::vector<uint32_t> deviceQueueIndices;
        	std::vector<VkQueue> deviceQueues;
            
            // 计算着色器专用的命令缓冲和缓冲池
        	VkCommandBuffer computeCommandBuffer;
        	VkCommandPool computeCommandPool;
        };

3. `VulkanContextCreator ctxCreator;` 类型定义如下，主要用来初始化 `vk` 和 `vkDev`：

        struct VulkanContextCreator
        {
        	VulkanContextCreator() = default;

        	VulkanContextCreator(VulkanInstance& vk, VulkanRenderDevice& dev, void* window, int screenWidth, int screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures());
        	~VulkanContextCreator();

        	VulkanInstance& instance;
        	VulkanRenderDevice& vkDev;
        };

        VulkanContextCreator::VulkanContextCreator(VulkanInstance& vk, VulkanRenderDevice& dev, void* window, int screenWidth, int screenHeight, const VulkanContextFeatures& ctxFeatures):
        	instance(vk),
        	vkDev(dev)
        {
        	createInstance(&vk.instance);

        	if (!setupDebugCallbacks(vk.instance, &vk.messenger, &vk.reportCallback))
        		exit(0);

        	if (glfwCreateWindowSurface(vk.instance, (GLFWwindow *)window, nullptr, &vk.surface))
        		exit(0);

        	if (!initVulkanRenderDevice3(vk, dev, screenWidth, screenHeight, ctxFeatures))
        		exit(0);
        }

    初始化主要分为以下几个部分：
    1. createInstance，创建 VulkanInstance 并传给 volk，创建信息主要包括 Vulkan 需要使用哪些扩展：

            void createInstance(VkInstance* instance)
            {
            	// https://vulkan.lunarg.com/doc/view/1.1.108.0/windows/validation_layers.html
            	const std::vector<const char*> ValidationLayers =
            	{
            		"VK_LAYER_KHRONOS_validation"
            	};

            	const std::vector<const char*> exts =
            	{
            		"VK_KHR_surface",
            #if defined (_WIN32)
            		"VK_KHR_win32_surface"
            #endif
            #if defined (__APPLE__)
            		"VK_MVK_macos_surface"
            #endif
            #if defined (__linux__)
            		"VK_KHR_xcb_surface"
            #endif
            		, VK_EXT_DEBUG_UTILS_EXTENSION_NAME
            		, VK_EXT_DEBUG_REPORT_EXTENSION_NAME
            		/* for indexed textures */
            		, VK_KHR_GET_PHYSICAL_DEVICE_PROPERTIES_2_EXTENSION_NAME
            	};

            	const VkApplicationInfo appinfo =
            	{
            		.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
            		.pNext = nullptr,
            		.pApplicationName = "Vulkan",
            		.applicationVersion = VK_MAKE_VERSION(1, 0, 0),
            		.pEngineName = "No Engine",
            		.engineVersion = VK_MAKE_VERSION(1, 0, 0),
            		.apiVersion = VK_API_VERSION_1_1
            	};

            	const VkInstanceCreateInfo createInfo =
            	{
            		.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
            		.pNext = nullptr,
            		.flags = 0,
            		.pApplicationInfo = &appinfo,
            		.enabledLayerCount = static_cast<uint32_t>(ValidationLayers.size()),
            		.ppEnabledLayerNames = ValidationLayers.data(),
            		.enabledExtensionCount = static_cast<uint32_t>(exts.size()),
            		.ppEnabledExtensionNames = exts.data()
            	};

            	VK_CHECK(vkCreateInstance(&createInfo, nullptr, instance));

            	volkLoadInstance(*instance);
            }
    2. setupDebugCallbacks，设置调试用回调函数：

            bool setupDebugCallbacks(VkInstance instance, VkDebugUtilsMessengerEXT* messenger, VkDebugReportCallbackEXT* reportCallback)
            {
            	{
            		const VkDebugUtilsMessengerCreateInfoEXT ci = {
            			.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT,
            			.messageSeverity =
            				VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT |
            				VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT,
            			.messageType =
            				VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT |
            				VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT |
            				VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT,
            			.pfnUserCallback = &VulkanDebugCallback,
            			.pUserData = nullptr
            		};

            		VK_CHECK(vkCreateDebugUtilsMessengerEXT(instance, &ci, nullptr, messenger));
            	}
            	{
            		const VkDebugReportCallbackCreateInfoEXT ci = {
            			.sType = VK_STRUCTURE_TYPE_DEBUG_REPORT_CALLBACK_CREATE_INFO_EXT,
            			.pNext = nullptr,
            			.flags =
            				VK_DEBUG_REPORT_WARNING_BIT_EXT |
            				VK_DEBUG_REPORT_PERFORMANCE_WARNING_BIT_EXT |
            				VK_DEBUG_REPORT_ERROR_BIT_EXT |
            				VK_DEBUG_REPORT_DEBUG_BIT_EXT,
            			.pfnCallback = &VulkanDebugReportCallback,
            			.pUserData = nullptr
            		};

            		VK_CHECK(vkCreateDebugReportCallbackEXT(instance, &ci, nullptr, reportCallback));
            	}

            	return true;
            }
    3. `glfwCreateWindowSurface` 创建 Surface 表面；
    4. `initVulkanRenderDevice3` 初始化 `vkDev`，主要是设置设备需要支持的特性，具体创建参照 `initVulkanRenderDevice2WithCompute`：

            struct VulkanContextFeatures
            {
            	bool supportScreenshots_ = false;

            	bool geometryShader_    = true;
            	bool tessellationShader_ = false;

            	bool vertexPipelineStoresAndAtomics_ = false;
            	bool fragmentStoresAndAtomics_ = false;
            };

            bool initVulkanRenderDevice3(VulkanInstance& vk, VulkanRenderDevice& vkDev, uint32_t width, uint32_t height, const VulkanContextFeatures& ctxFeatures)
            {
            	VkPhysicalDeviceDescriptorIndexingFeaturesEXT physicalDeviceDescriptorIndexingFeatures = {
            		.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DESCRIPTOR_INDEXING_FEATURES_EXT,
            		.shaderSampledImageArrayNonUniformIndexing = VK_TRUE,
            		.descriptorBindingVariableDescriptorCount = VK_TRUE,
            		.runtimeDescriptorArray = VK_TRUE,
            	};

            	VkPhysicalDeviceFeatures deviceFeatures = {
            		/* for wireframe outlines */
            		.geometryShader = (VkBool32)(ctxFeatures.geometryShader_ ? VK_TRUE : VK_FALSE),
            		/* for tesselation experiments */
            		.tessellationShader = (VkBool32)(ctxFeatures.tessellationShader_ ? VK_TRUE : VK_FALSE),
            		/* for indirect instanced rendering */
            		.multiDrawIndirect = VK_TRUE,
            		.drawIndirectFirstInstance = VK_TRUE,
            		/* for OIT and general atomic operations */
            		.vertexPipelineStoresAndAtomics = (VkBool32)(ctxFeatures.vertexPipelineStoresAndAtomics_ ? VK_TRUE : VK_FALSE),
            		.fragmentStoresAndAtomics = (VkBool32)(ctxFeatures.fragmentStoresAndAtomics_ ? VK_TRUE : VK_FALSE),
            		/* for arrays of textures */
            		.shaderSampledImageArrayDynamicIndexing = VK_TRUE,
            		/* for GL <-> VK material shader compatibility */
            		.shaderInt64 =  VK_TRUE,
            	};

            	VkPhysicalDeviceFeatures2 deviceFeatures2 = {
            		.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
            		.pNext = &physicalDeviceDescriptorIndexingFeatures,
            		.features = deviceFeatures  /*  */
            	};

            	return initVulkanRenderDevice2WithCompute(vk, vkDev, width, height, isDeviceSuitable, deviceFeatures2, ctxFeatures.supportScreenshots_);
            }
    5. `initVulkanRenderDevice2WithCompute` 创建 `VulkanRenderDevice` 变量：

            bool initVulkanRenderDevice2WithCompute(VulkanInstance& vk, VulkanRenderDevice& vkDev, uint32_t width, uint32_t height, std::function<bool(VkPhysicalDevice)> selector, VkPhysicalDeviceFeatures2 deviceFeatures2, bool supportScreenshots)
            {
            	// 1. 设置 framebuffer 宽高
                vkDev.framebufferWidth = width;
            	vkDev.framebufferHeight = height;

                // 2. 查找合适的 physicalDevice，并创建创建 device，以及查找图形队列和计算队列的索引
            	VK_CHECK(findSuitablePhysicalDevice(vk.instance, selector, &vkDev.physicalDevice));
            	vkDev.graphicsFamily = findQueueFamilies(vkDev.physicalDevice, VK_QUEUE_GRAPHICS_BIT);
            	vkDev.computeFamily = findQueueFamilies(vkDev.physicalDevice, VK_QUEUE_COMPUTE_BIT);
            	VK_CHECK(createDevice2WithCompute(vkDev.physicalDevice, deviceFeatures2, vkDev.graphicsFamily, vkDev.computeFamily, &vkDev.device));

                // 3. 创建图形队列和计算队列
            	vkGetDeviceQueue(vkDev.device, vkDev.graphicsFamily, 0, &vkDev.graphicsQueue);
            	if (vkDev.graphicsQueue == nullptr)
            		exit(EXIT_FAILURE);

            	vkGetDeviceQueue(vkDev.device, vkDev.computeFamily, 0, &vkDev.computeQueue);
            	if (vkDev.computeQueue == nullptr)
            		exit(EXIT_FAILURE);

                // 4. 查询是否支持该 surface
            	VkBool32 presentSupported = 0;
            	vkGetPhysicalDeviceSurfaceSupportKHR(vkDev.physicalDevice, vkDev.graphicsFamily, vk.surface, &presentSupported);
            	if (!presentSupported)
            		exit(EXIT_FAILURE);

                // 5. 创建 swapchain 交换链，以及交换链中的图像对象
            	VK_CHECK(createSwapchain(vkDev.device, vkDev.physicalDevice, vk.surface, vkDev.graphicsFamily, width, height, &vkDev.swapchain, supportScreenshots));
            	const size_t imageCount = createSwapchainImages(vkDev.device, vkDev.swapchain, vkDev.swapchainImages, vkDev.swapchainImageViews);

                // 6. 更新图像命令缓冲的数量，交换链中的每个图像对应一个命令缓冲
            	vkDev.commandBuffers.resize(imageCount);

                // 7. 创建信号量
            	VK_CHECK(createSemaphore(vkDev.device, &vkDev.semaphore));
            	VK_CHECK(createSemaphore(vkDev.device, &vkDev.renderSemaphore));

                // 8. 创建图像命令缓冲池，绑定图像提交队列，并创建命令缓冲
            	const VkCommandPoolCreateInfo cpi =
            	{
            		.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
            		.flags = 0,
            		.queueFamilyIndex = vkDev.graphicsFamily
            	};

            	VK_CHECK(vkCreateCommandPool(vkDev.device, &cpi, nullptr, &vkDev.commandPool));

            	const VkCommandBufferAllocateInfo ai =
            	{
            		.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
            		.pNext = nullptr,
            		.commandPool = vkDev.commandPool,
            		.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY,
            		.commandBufferCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
            	};

            	VK_CHECK(vkAllocateCommandBuffers(vkDev.device, &ai, &vkDev.commandBuffers[0]));
                
                // 9. 创建计算命令缓冲池，绑定计算提交队列，并创建命令缓冲
            	{
            		// Create compute command pool
            		const VkCommandPoolCreateInfo cpi1 =
            		{
            			.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
            			.pNext = nullptr,
            			.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT, /* Allow command from this pool buffers to be reset*/
            			.queueFamilyIndex = vkDev.computeFamily
            		};
            		VK_CHECK(vkCreateCommandPool(vkDev.device, &cpi1, nullptr, &vkDev.computeCommandPool));

            		const VkCommandBufferAllocateInfo ai1 =
            		{
            			.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
            			.pNext = nullptr,
            			.commandPool = vkDev.computeCommandPool,
            			.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY,
            			.commandBufferCount = 1,
            		};

            		VK_CHECK(vkAllocateCommandBuffers(vkDev.device, &ai1, &vkDev.computeCommandBuffer));
            	}

            	vkDev.useCompute = true;

            	return true;
            }

4. `VulkanResources resources;` 主要是用来构建各种 Vulkan 资源，对原生的 Vulkan API 进行封装，详细信息参照 [引擎架构之资源管理](engine/resource.md) 章节，以下是类型声明：

        struct VulkanResources
        {
        	VulkanResources(VulkanRenderDevice& vkDev): vkDev(vkDev) {}
        	~VulkanResources();

        	VulkanTexture loadTexture2D(const char* filename);

        	VulkanTexture loadCubeMap(const char* fileName, uint32_t mipLevels = 1);

        	VulkanTexture loadKTX(const char* fileName);

        	VulkanTexture createFontTexture(const char* fontFile);

        	VulkanTexture addColorTexture(int texWidth = 0, int texHeight = 0, VkFormat colorFormat = VK_FORMAT_B8G8R8A8_UNORM, VkFilter minFilter = VK_FILTER_LINEAR, VkFilter maxFilter = VK_FILTER_LINEAR, VkSamplerAddressMode addressMode = VK_SAMPLER_ADDRESS_MODE_REPEAT);

        	VulkanTexture addDepthTexture(int texWidth = 0, int texHeight = 0, VkImageLayout layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);

        	VulkanTexture addSolidRGBATexture(uint32_t color = 0xFFFFFFFF);

        	VulkanTexture addRGBATexture(int texWidth, int texHeight, void* data);

        	VulkanBuffer addBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, bool createMapping = false);

        	inline VulkanBuffer addUniformBuffer(VkDeviceSize bufferSize, bool createMapping = false) {
        		return addBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
        			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, createMapping);
        	}

        	inline VulkanBuffer addIndirectBuffer(VkDeviceSize bufferSize, bool createMapping = false) {
        		return addBuffer(bufferSize, VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT, // | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
        			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, createMapping); /* for debugging we make it host-visible */
        	}

        	inline VulkanBuffer addStorageBuffer(VkDeviceSize bufferSize, bool createMapping = false) {
        		return addBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
        			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, createMapping); /* for debugging we make it host-visible */
        	}

        	inline VulkanBuffer addComputedIndirectBuffer(VkDeviceSize bufferSize, bool createMapping = false) {
        		return addBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT,
        			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, createMapping); /* for debugging we make it host-visible */
        	}

        	/* Allocate and upload vertex & index buffer pair */
        	VulkanBuffer addVertexBuffer(uint32_t indexBufferSize, const void* indexData, uint32_t vertexBufferSize, const void* vertexData);

        	VkFramebuffer addFramebuffer(RenderPass renderPass, const std::vector<VulkanTexture>& images);

        	RenderPass addFullScreenPass(bool useDepth = true, const RenderPassCreateInfo& ci = RenderPassCreateInfo());

        	RenderPass addRenderPass(const std::vector<VulkanTexture>& outputs, const RenderPassCreateInfo& ci = {
        		.clearColor_ = true, .clearDepth_ = true,
        		.flags_ = eRenderPassBit_Offscreen | eRenderPassBit_First }, bool useDepth = true);

        	RenderPass addDepthRenderPass(const std::vector<VulkanTexture>& outputs, const RenderPassCreateInfo& ci = {
        		.clearColor_ = false, .clearDepth_ = true,
        		.flags_ = eRenderPassBit_Offscreen | eRenderPassBit_First });

        	VkPipelineLayout addPipelineLayout(VkDescriptorSetLayout dsLayout, uint32_t vtxConstSize = 0, uint32_t fragConstSize = 0);

        	VkPipeline addPipeline(VkRenderPass renderPass, VkPipelineLayout pipelineLayout,
        		const std::vector<const char*>& shaderFiles,
        		const PipelineInfo& pipelineParams = PipelineInfo { .width = 0, .height = 0, .topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST, .useDepth = true, .useBlending = false, .dynamicScissorState = false });

        	VkPipeline addComputePipeline(const char* shaderFile, VkPipelineLayout pipelineLayout);

        	/* Calculate the descriptor pool size from the list of buffers and textures */
        	VkDescriptorPool addDescriptorPool(const DescriptorSetInfo& dsInfo, uint32_t dSetCount = 1);

        	VkDescriptorSetLayout addDescriptorSetLayout(const DescriptorSetInfo& dsInfo);

        	VkDescriptorSet addDescriptorSet(VkDescriptorPool descriptorPool, VkDescriptorSetLayout dsLayout);

        	void updateDescriptorSet(VkDescriptorSet ds, const DescriptorSetInfo& dsInfo);

        	const std::vector<VulkanTexture>& getTextures() const { return allTextures; } 

        	std::vector<VkFramebuffer> addFramebuffers(VkRenderPass renderPass, VkImageView depthView = VK_NULL_HANDLE);

        	/**  Helper functions for small Chapter 8/9 demos */
        	std::pair<BufferAttachment, BufferAttachment> makeMeshBuffers(const std::vector<float>& vertices, const std::vector<unsigned int>& indices);

        	std::pair<BufferAttachment, BufferAttachment> loadMeshToBuffer(const char* filename, bool useTextureCoordinates, bool useNormals,
        		std::vector<float>& vertices,
        		std::vector<unsigned int>& indices);

        	std::pair<BufferAttachment, BufferAttachment> createPlaneBuffer_XZ(float sx, float sz);
        	std::pair<BufferAttachment, BufferAttachment> createPlaneBuffer_XY(float sx, float sy);

        private:
        	VulkanRenderDevice& vkDev;

        	std::vector<VulkanTexture> allTextures;
        	std::vector<VulkanBuffer> allBuffers;

        	std::vector<VkFramebuffer> allFramebuffers;
        	std::vector<VkRenderPass> allRenderPasses;

        	std::vector<VkPipelineLayout> allPipelineLayouts;
        	std::vector<VkPipeline> allPipelines;

        	std::vector<VkDescriptorSetLayout> allDSLayouts;
        	std::vector<VkDescriptorPool>      allDPools;

        	std::vector<ShaderModule> shaderModules;
        	std::map<std::string, int> shaderMap;

        	bool createGraphicsPipeline(
        		VulkanRenderDevice& vkDev,
        		VkRenderPass renderPass, VkPipelineLayout pipelineLayout,
        		const std::vector<const char*>& shaderFiles,
        		VkPipeline* pipeline,
        		VkPrimitiveTopology topology,
        		bool useDepth,
        		bool useBlending,
        		bool dynamicScissorState,
        		int32_t customWidth,
        		int32_t customHeight,
        		uint32_t numPatchControlPoints);
        };        
5. 除了上述最关键的四个初始属性外，其他属性用来保存 Vulkan 相关的渲染资源，其中关于 Renderer 的详细分析见 [引擎架构之 FrameworkRenderer 封装](engine/frameworkRenderer.md)：

        struct RenderItem {
        	Renderer& renderer_;
        	bool enabled_ = true;
        	bool useDepth_ = true;
        	explicit RenderItem(Renderer& r, bool useDepth = true)
        	: renderer_(r)
        	, useDepth_(useDepth)
        	{}
        };

        // 1. 主要用来保存 Renderer，后面会具体分析 Renderer，这里的 Renderer 由子类初始化时传入
        std::vector<RenderItem> onScreenRenderers_;

        // 2. 保存深度缓冲
    	VulkanTexture depthTexture;
    
        // 3. 用来屏幕绘制的 renderPass
    	// Framebuffers and renderpass for on-screen rendering
    	RenderPass screenRenderPass;
    	RenderPass screenRenderPass_NoDepth;

    	RenderPass clearRenderPass; // 清空颜色和深度缓存的 renderPass
    	RenderPass finalRenderPass; // 调整图像格式，用来进行呈现

        // 4. 保存不同 swapChain 中图像的 frameBuffer
    	std::vector<VkFramebuffer> swapchainFramebuffers;
    	std::vector<VkFramebuffer> swapchainFramebuffers_NoDepth;

分析完属性，我们看一下 `VulkanRenderContext` 中的函数：
1. 先看构造函数

        VulkanRenderContext(void* window, uint32_t screenWidth, uint32_t screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures()):
	    	ctxCreator(vk, vkDev, window, screenWidth, screenHeight, ctxFeatures),
	    	resources(vkDev),

	    	depthTexture(resources.addDepthTexture(vkDev.framebufferWidth, vkDev.framebufferHeight, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL)),

	    	screenRenderPass(resources.addFullScreenPass()),
	    	screenRenderPass_NoDepth(resources.addFullScreenPass(false)),

	    	finalRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ = false, .clearDepth_ = false, .flags_ = eRenderPassBit_Last  })),
	    	clearRenderPass(resources.addFullScreenPass(true, RenderPassCreateInfo { .clearColor_ =  true, .clearDepth_ =  true, .flags_ = eRenderPassBit_First })),

	    	swapchainFramebuffers(resources.addFramebuffers(screenRenderPass.handle, depthTexture.image.imageView)),
	    	swapchainFramebuffers_NoDepth(resources.addFramebuffers(screenRenderPass_NoDepth.handle))
	    {
	    }
    这里主要是用来初始化 vk、vkDev、resources，以及各种资源
2. 调用每个 renderer 的 `updateBuffers` 方法，更新数据：

        void VulkanRenderContext::updateBuffers(uint32_t imageIndex)
        {
        	for (auto& r : onScreenRenderers_)
        		if (r.enabled_)
        			r.renderer_.updateBuffers(imageIndex);
        }
3. 这里的 `composeFrame` 函数主要分为三部分，清空颜色与深度缓存、调用每个 renderer 的渲染指令、转换 swapChain 图像格式，以便于呈现

        void VulkanRenderContext::composeFrame(VkCommandBuffer commandBuffer, uint32_t imageIndex)
        {
        	const VkRect2D defaultScreenRect {
        		.offset = { 0, 0 },
        		.extent = {.width = vkDev.framebufferWidth, .height = vkDev.framebufferHeight }
        	};

        	static const VkClearValue defaultClearValues[2] =
        	{
        		VkClearValue { .color = { 1.0f, 1.0f, 1.0f, 1.0f } },
        		VkClearValue { .depthStencil = { 1.0f, 0 } }
        	};

        	beginRenderPass(commandBuffer, clearRenderPass.handle, imageIndex, defaultScreenRect, VK_NULL_HANDLE, 2u, defaultClearValues);
        	vkCmdEndRenderPass( commandBuffer );

        	for (auto& r : onScreenRenderers_)
        		if (r.enabled_)
        		{
        			RenderPass rp = r.useDepth_ ? screenRenderPass : screenRenderPass_NoDepth;
        			VkFramebuffer fb = (r.useDepth_ ? swapchainFramebuffers : swapchainFramebuffers_NoDepth)[imageIndex];

        			if (r.renderer_.renderPass_.handle != VK_NULL_HANDLE)
        				rp = r.renderer_.renderPass_;
        			if (r.renderer_.framebuffer_ != VK_NULL_HANDLE)
        				fb = r.renderer_.framebuffer_;

        			r.renderer_.fillCommandBuffer(commandBuffer, imageIndex, fb, rp.handle);
        		}

        	beginRenderPass(commandBuffer, finalRenderPass.handle, imageIndex, defaultScreenRect);
        	vkCmdEndRenderPass( commandBuffer );
        }  
4. 这里的 beginRenderPass，主要用于在 command 中开始一次 renderPass，这里主要需要设置的属性包括：renderPass、frameBuffer、渲染区域，以及用来清空缓存的值  

        void beginRenderPass(VkCommandBuffer cmdBuffer, VkRenderPass pass, size_t currentImage, const VkRect2D area,
        	VkFramebuffer fb = VK_NULL_HANDLE,
        	uint32_t clearValueCount = 0, const VkClearValue* clearValues = nullptr)
        {
        	const VkRenderPassBeginInfo renderPassInfo = {
        		.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        		.renderPass = pass,
        		.framebuffer = (fb != VK_NULL_HANDLE) ? fb : swapchainFramebuffers[currentImage],
        		.renderArea = area,
        		.clearValueCount = clearValueCount,
        		.pClearValues = clearValues
        	};

        	vkCmdBeginRenderPass( cmdBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE );
        }   

### mainLoop 
`mainLoop` 函数的主要作用是开始渲染循环

        void VulkanApp::mainLoop()
        {
        	double timeStamp = glfwGetTime();
        	float deltaSeconds = 0.0f;

        	do
        	{
        		// 1、 update 是虚函数，由子类覆盖实现，用来根据时间间隔，更新变量
                update(deltaSeconds);

        		const double newTimeStamp = glfwGetTime();
        		deltaSeconds = static_cast<float>(newTimeStamp - timeStamp);
        		timeStamp = newTimeStamp;

                // 2、 计算 FPS
        		fpsCounter_.tick(deltaSeconds);

                // 3、核心渲染函数，渲染一帧画面，下面详细分析
        		bool frameRendered = drawFrame(ctx_.vkDev,
        			[this](uint32_t img) { this->updateBuffers(img); },
        			[this](auto cmd, auto img) { ctx_.composeFrame(cmd, img); }
        		);

        		fpsCounter_.tick(deltaSeconds, frameRendered);

        		glfwPollEvents();

        	} while (!glfwWindowShouldClose(window_));
        }
其中，核心是 `drawFrame` 函数，下面详细分析：

        bool drawFrame(VulkanRenderDevice& vkDev, const std::function<void(uint32_t)>& updateBuffersFunc, const std::function<void(VkCommandBuffer, uint32_t)>& composeFrameFunc)
        {
        	// 1、渲染准备工作，获取 swapChain 交换链中可用的图像的索引，并重置图像提交命令缓冲池，注意这里的 `vkDev.semaphore` 表示获取到图像后再触发
            uint32_t imageIndex = 0;
        	VkResult result = vkAcquireNextImageKHR(vkDev.device, vkDev.swapchain, 0, vkDev.semaphore, VK_NULL_HANDLE, &imageIndex);
        	VK_CHECK(vkResetCommandPool(vkDev.device, vkDev.commandPool, 0));

        	if (result != VK_SUCCESS) return false;

            // 2、触发每帧数据缓冲的更新调用，下面有详细分析
        	updateBuffersFunc(imageIndex);

            // 3、获取对应的提交命令，准备开始录制命令
        	VkCommandBuffer commandBuffer = vkDev.commandBuffers[imageIndex];

        	const VkCommandBufferBeginInfo bi =
        	{
        		.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
        		.pNext = nullptr,
        		.flags = VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT,
        		.pInheritanceInfo = nullptr
        	};

        	VK_CHECK(vkBeginCommandBuffer(commandBuffer, &bi));
            
            // 4、此处调用的即是 `VulkanRenderContext::composeFrame`，用来依次调用 Renderer，录制绘制命令
        	composeFrameFunc(commandBuffer, imageIndex);

            // 5、结束命令缓冲录制
        	VK_CHECK(vkEndCommandBuffer(commandBuffer));

            // 6、提交命令，主要信息为三部分：要提交的命令，需要等待的获取图像的信号量 `vkDev.semaphore`，渲染完成后会触发的信号量 `vkDev.renderSemaphore`
        	const VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT }; // or even VERTEX_SHADER_STAGE

        	const VkSubmitInfo si =
        	{
        		.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
        		.pNext = nullptr,
        		.waitSemaphoreCount = 1,
        		.pWaitSemaphores = &vkDev.semaphore,
        		.pWaitDstStageMask = waitStages,
        		.commandBufferCount = 1,
        		.pCommandBuffers = &vkDev.commandBuffers[imageIndex],
        		.signalSemaphoreCount = 1,
        		.pSignalSemaphores = &vkDev.renderSemaphore
        	};

        	VK_CHECK(vkQueueSubmit(vkDev.graphicsQueue, 1, &si, nullptr));

            // 7、呈现渲染好的图像，主要信息包括：swapChain 交换链、本帧渲染的交换链中的图像的索引、等待的渲染完成的信号量 `vkDev.renderSemaphore`
        	const VkPresentInfoKHR pi =
        	{
        		.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
        		.pNext = nullptr,
        		.waitSemaphoreCount = 1,
        		.pWaitSemaphores = &vkDev.renderSemaphore,
        		.swapchainCount = 1,
        		.pSwapchains = &vkDev.swapchain,
        		.pImageIndices = &imageIndex
        	};

        	VK_CHECK(vkQueuePresentKHR(vkDev.graphicsQueue, &pi));

            // 8、简单起见，等待命令处理完成
        	VK_CHECK(vkDeviceWaitIdle(vkDev.device));

        	return true;
        }
`drawFrame` 函数的主要作用是用来从 swapChain 中获取该帧渲染对应的图像对象的索引，然后准备好命令缓冲之后，录制好命令，在提交到队列中渲染，渲染完成后呈现到 swapChain 中，其中有两个自定义的函数，是从外部传入进来的，下面会详细分析：
1. updateBuffersFunc

        void VulkanApp::updateBuffers(uint32_t imageIndex)
        {
        	ImGuiIO& io = ImGui::GetIO();
        	io.DisplaySize = ImVec2((float)ctx_.vkDev.framebufferWidth, (float)ctx_.vkDev.framebufferHeight);
        	ImGui::NewFrame();

        	drawUI();

        	ImGui::Render();

        	draw3D();

        	ctx_.updateBuffers(imageIndex);
        }
    该函数用来在每帧渲染时调用，用来触发以下渲染数据更新：
    1. `drawUI()` 虚函数，子类实现，更新 UI 的渲染数据；
    2. `draw3D()` 虚函数，子类实现，一般用来更新 Renderer 的 MVP 数据；
    3. `ctx_.updateBuffers(imageIndex)`，调用 `VulkanRenderContext::updateBuffers`，这里会触发每个 Renderer 的 `updateBuffers` 函数；
2. composeFrameFunc  
这里调用的是 `VulkanRenderContext::composeFrame` 函数，用来真正录制绘制命令，详细解释参考上文；

### 构造函数

        VulkanApp(int screenWidth, int screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures()):
            window_(initVulkanApp(screenWidth, screenHeight, &resolution_)),
	    	ctx_(window_, resolution_.width, resolution_.height, ctxFeatures),
	    	onScreenRenderers_(ctx_.onScreenRenderers_)
	    {
	    	glfwSetWindowUserPointer(window_, this);
	    	assignCallbacks();
	    }

        void VulkanApp::assignCallbacks()
        {
        	glfwSetCursorPosCallback(
        		window_,
        		[](GLFWwindow* window, double x, double y)
        		{
        			ImGui::GetIO().MousePos = ImVec2((float)x, (float)y);
        			int width, height;
        			glfwGetFramebufferSize(window, &width, &height);

        			void* ptr = glfwGetWindowUserPointer(window); 	
        			const float mx = static_cast<float>(x / width);
        			const float my = static_cast<float>(y / height);
        			reinterpret_cast<VulkanApp*>(ptr)->handleMouseMove(mx, my);
        		}
        	);

        	glfwSetMouseButtonCallback(
        		window_,
        		[](GLFWwindow* window, int button, int action, int mods)
        		{
        			auto& io = ImGui::GetIO();
        			const int idx = button == GLFW_MOUSE_BUTTON_LEFT ? 0 : button == GLFW_MOUSE_BUTTON_RIGHT ? 2 : 1;
        			io.MouseDown[idx] = action == GLFW_PRESS;

        			void* ptr = glfwGetWindowUserPointer(window); 	
        			reinterpret_cast<VulkanApp*>(ptr)->handleMouseClick(button, action == GLFW_PRESS);
        		}
        	);

        	glfwSetKeyCallback(
        		window_,
        		[](GLFWwindow* window, int key, int scancode, int action, int mods)
        		{
        			const bool pressed = action != GLFW_RELEASE;
        			if (key == GLFW_KEY_ESCAPE && pressed)
        				glfwSetWindowShouldClose(window, GLFW_TRUE);

        				void* ptr = glfwGetWindowUserPointer(window); 	
        				reinterpret_cast<VulkanApp*>(ptr)->handleKey(key, pressed);
        		}
        	);
        }
构造函数中主要是用来设置 GLFW 窗口的回调函数，监听鼠标和键盘的事件

## 源码分析 - CameraApp

		struct CameraApp: public VulkanApp
		{
			CameraApp(int screenWidth, int screenHeight, const VulkanContextFeatures& ctxFeatures = VulkanContextFeatures()):
				VulkanApp(screenWidth, screenHeight, ctxFeatures),

				positioner(glm::vec3(0.0f, 5.0f, 10.0f), vec3(0.0f, 0.0f, -1.0f), vec3(0.0f, -1.0f, 0.0f)),
				camera(positioner)
			{}

			virtual void update(float deltaSeconds) override
			{
				positioner.update(deltaSeconds, mouseState_.pos, shouldHandleMouse() ? mouseState_.pressedLeft : false);
			}

			glm::mat4 getDefaultProjection() const {
				const float ratio = ctx_.vkDev.framebufferWidth / (float)ctx_.vkDev.framebufferHeight;
				return glm::perspective(45.0f, ratio, 0.1f, 1000.0f);
			}

			virtual void handleKey(int key, bool pressed) override;

		protected:
			CameraPositioner_FirstPerson positioner;
			Camera camera;
		};

		struct VulkanContextFeatures
		{
			bool supportScreenshots_ = false;

			bool geometryShader_    = true;
			bool tessellationShader_ = false;

			bool vertexPipelineStoresAndAtomics_ = false;
			bool fragmentStoresAndAtomics_ = false;
		};

		void CameraApp::handleKey(int key, bool pressed)
		{
			if (key == GLFW_KEY_W)
				positioner.movement_.forward_ = pressed;
			if (key == GLFW_KEY_S)
				positioner.movement_.backward_ = pressed;
			if (key == GLFW_KEY_A)
				positioner.movement_.left_ = pressed;
			if (key == GLFW_KEY_D)
				positioner.movement_.right_ = pressed;
			if (key == GLFW_KEY_C)
				positioner.movement_.up_ = pressed;
			if (key == GLFW_KEY_E)
				positioner.movement_.down_ = pressed;
		}

这里的 CemaraApp 主要是增加了一个第一人称的摄像机组件，其中：
1. 实现了虚函数 `virtual void update(float deltaSeconds) override`，用来在每帧更新摄像机的位置；
2. 实现了虚函数 `virtual void handleKey(int key, bool pressed) override`，用来监听按键的变化；
3. 实现了函数 `glm::mat4 getDefaultProjection() const`，用来给子类提供 P 矩阵；

## 源码分析 - MyApp	

		struct MyApp: public CameraApp
		{
			MyApp()
			: CameraApp(-95, -95)
			, envMap(ctx_.resources.loadCubeMap("data/piazza_bologni_1k.hdr"))
			, irrMap(ctx_.resources.loadCubeMap("data/piazza_bologni_1k_irradiance.hdr"))
			, sceneData(ctx_, "data/meshes/test_graph.meshes", "data/meshes/test_graph.scene", "data/meshes/test_graph.materials", envMap, irrMap)
			, plane(ctx_)
			, multiRenderer(ctx_, sceneData)
			, imgui(ctx_)
			{
				onScreenRenderers_.emplace_back(plane, false);
				onScreenRenderers_.emplace_back(multiRenderer);
				onScreenRenderers_.emplace_back(imgui, false);

				sceneData.scene_.localTransform_[0] = glm::rotate(glm::mat4(1.f), (float)(M_PI / 2.f), glm::vec3(1.f, 0.f, 0.0f));
			}

			void drawUI() override {
				ImGui::Begin("Information", nullptr);
					ImGui::Text("FPS: %.2f", getFPS());
				ImGui::End();

				ImGui::Begin("Scene graph", nullptr);
					int node = renderSceneTree(sceneData.scene_, 0);
					if (node > -1)
						selectedNode = node;
				ImGui::End();

				editNode(selectedNode);
			}

			void draw3D() override {
				const mat4 p = getDefaultProjection();
				const mat4 view =camera.getViewMatrix();

				multiRenderer.setMatrices(p, view);
				plane.setMatrices(p, view, mat4(1.f));
			}

			void update(float deltaSeconds) override
			{
				CameraApp::update(deltaSeconds);

				// update/upload matrices for individual scene nodes
				sceneData.recalculateAllTransforms();
				sceneData.uploadGlobalTransforms();
			}

		private:
			VulkanTexture envMap;
			VulkanTexture irrMap;

			VKSceneData sceneData;

			InfinitePlaneRenderer plane;
			MultiRenderer multiRenderer;
			GuiRenderer imgui;

			int selectedNode = -1;

			void editNode(int node)
			{
				if (node < 0)
					return;

				mat4 cameraProjection = getDefaultProjection();
				mat4 cameraView = camera.getViewMatrix();

				ImGuizmo::SetOrthographic(false);
				ImGuizmo::BeginFrame();

				std::string name = getNodeName(sceneData.scene_, node);
				std::string label = name.empty() ? (std::string("Node") + std::to_string(node)) : name;
				label = "Node: " + label;

				ImGui::Begin("Editor");
					ImGui::Text("%s", label.c_str());

					glm::mat4 globalTransform = sceneData.scene_.globalTransform_[node]; // fetch global transform
					glm::mat4 srcTransform = globalTransform;
					glm::mat4 localTransform = sceneData.scene_.localTransform_[node];

					ImGui::Separator();
					ImGuizmo::SetID(1);

					editTransform(cameraView, cameraProjection, globalTransform);
					glm::mat4 deltaTransform = glm::inverse(srcTransform) * globalTransform;  // calculate delta for edited global transform
					sceneData.scene_.localTransform_[node] = localTransform * deltaTransform; // modify local transform
					markAsChanged(sceneData.scene_, node);

					ImGui::Separator();
					ImGui::Text("%s", "Material");

					editMaterial(node);
				ImGui::End();
			}

			void editTransform(const glm::mat4& view, const glm::mat4& projection, glm::mat4& matrix)
			{
				static ImGuizmo::OPERATION gizmoOperation(ImGuizmo::TRANSLATE);

				if (ImGui::RadioButton("Translate", gizmoOperation == ImGuizmo::TRANSLATE))
					gizmoOperation = ImGuizmo::TRANSLATE;
				ImGui::SameLine();
				if (ImGui::RadioButton("Rotate", gizmoOperation == ImGuizmo::ROTATE))
					gizmoOperation = ImGuizmo::ROTATE;

				ImGuiIO& io = ImGui::GetIO();
				ImGuizmo::SetRect(0, 0, io.DisplaySize.x, io.DisplaySize.y);
				ImGuizmo::Manipulate(glm::value_ptr(view), glm::value_ptr(projection), gizmoOperation, ImGuizmo::WORLD, glm::value_ptr(matrix));
			}

			void editMaterial(int node) {
				if (!sceneData.scene_.materialForNode_.contains(node))
					return;

				auto matIdx = sceneData.scene_.materialForNode_.at(node);
				MaterialDescription& material = sceneData.materials_[matIdx];

				float emissiveColor[4];
				memcpy(emissiveColor, &material.emissiveColor_, sizeof(gpuvec4));
				gpuvec4 oldColor = material.emissiveColor_;
				ImGui::ColorEdit3("Emissive color", emissiveColor);

				if (memcmp(emissiveColor, &oldColor, sizeof(gpuvec4))) {
					memcpy(&material.emissiveColor_, emissiveColor, sizeof(gpuvec4));
					sceneData.updateMaterial(matIdx);
				}
			}
		};
MyApp 的作用主要有三个：
1. 初始化 SceneData，场景数据；
2. 初始化 Renderer，并和 SceneData 关联；
3. 实现简单的节点编辑器功能；
下面我们依次来分析