# 引擎架构之 Vulkan Renderer 封装

## 基本思路
封装的基本思路是分层渲染，每一层保留自己的 uniform、descriptor set、pipeline、render pass、framebuffer，这通过继承 RendererBase 类实现，然后在渲染的循环里，在每一帧循环的最外层构建 command buffer，然后遍历不同层的 renderer，提交命令，代码如下：

    VK_CHECK(vkBeginCommandBuffer(commandBuffer, &bi));
	for (auto& r : renderers)
	    r->fillCommandBuffer(commandBuffer, imageIndex;
	VK_CHECK(vkEndCommandBuffer(commandBuffer));   

构建好命令后，提交到渲染队列，并等待呈现。

## 模块剖析
### RendererBase
RendererBase 作为所以 layer 的父类，负责保存 Vulkan 渲染需要的状态，以及根据这些状态拼接对应的命令，见代码：

    class RendererBase
    {
    public:
    	explicit RendererBase(const VulkanRenderDevice& vkDev, VulkanImage depthTexture)
    	: device_(vkDev.device)
    	, framebufferWidth_(vkDev.framebufferWidth)
    	, framebufferHeight_(vkDev.framebufferHeight)
    	, depthTexture_(depthTexture)
    	{}
    	virtual ~RendererBase();
    	virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) = 0;

    	inline VulkanImage getDepthTexture() const { return depthTexture_; }

    protected:
    	void beginRenderPass(VkCommandBuffer commandBuffer, size_t currentImage);
    	bool createUniformBuffers(VulkanRenderDevice& vkDev, size_t uniformDataSize);

    	VkDevice device_ = nullptr;

    	uint32_t framebufferWidth_ = 0;
    	uint32_t framebufferHeight_ = 0;

    	// Depth buffer
    	VulkanImage depthTexture_;

    	// Descriptor set (layout + pool + sets) -> uses uniform buffers, textures, framebuffers
    	VkDescriptorSetLayout descriptorSetLayout_ = nullptr;
    	VkDescriptorPool descriptorPool_ = nullptr;
    	std::vector<VkDescriptorSet> descriptorSets_;

    	// Framebuffers (one for each command buffer)
    	std::vector<VkFramebuffer> swapchainFramebuffers_;

    	// 4. Pipeline & render pass (using DescriptorSets & pipeline state options)
    	VkRenderPass renderPass_ = nullptr;
    	VkPipelineLayout pipelineLayout_ = nullptr;
    	VkPipeline graphicsPipeline_ = nullptr;

    	// 5. Uniform buffer
    	std::vector<VkBuffer> uniformBuffers_;
    	std::vector<VkDeviceMemory> uniformBuffersMemory_;
    };

先看方法：  
1. fillCommandBuffer，填充绘制命令；
2. getDepthTexture，获取深度缓冲，为了在各个 layer 中共用；
3. beginRenderPass，设置 vkCmdBeginRenderPass、vkCmdBindPipeline、vkCmdBindDescriptorSet；
4. createUniformBuffers，更新 uniform buffer；

再看属性：
1. device_，VkDevice；
2. framebufferWidth_、framebufferHeight_，framebuffer 大小；
3. depthTexture_，深度缓冲；
4. swapchainFramebuffers_，framebuffer；
5. descriptorSetLayout_、descriptorPool_、descriptorSets_，保存 uniform 变量；
6. uniformBuffers_、uniformBuffersMemory_，uniform 缓冲；
7. renderPass_、pipelineLayout_、graphicsPipeline_，render pass 和 pipeline；

方法实现：
1. createUniformBuffers

        bool RendererBase::createUniformBuffers(VulkanRenderDevice& vkDev, size_t uniformDataSize)
        {
        	uniformBuffers_.resize(vkDev.swapchainImages.size());
        	uniformBuffersMemory_.resize(vkDev.swapchainImages.size());
        	for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
        	{
        		if (!createUniformBuffer(vkDev, uniformBuffers_[i], uniformBuffersMemory_[i], uniformDataSize))
        		{
        			printf("Cannot create uniform buffer\n");
        			fflush(stdout);
        			return false;
        		}
        	}
        	return true;
        }

    根据 swapchainImages 大小，调用子类的 createUniformBuffer，填充 uniformBuffers_；
2. beginRenderPass

        void RendererBase::beginRenderPass(VkCommandBuffer commandBuffer, size_t currentImage)
        {
        	const VkRect2D screenRect = {
        		.offset = { 0, 0 },
        		.extent = {.width = framebufferWidth_, .height = framebufferHeight_ }
        	};

        	const VkRenderPassBeginInfo renderPassInfo = {
        		.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        		.pNext = nullptr,
        		.renderPass = renderPass_,
        		.framebuffer = swapchainFramebuffers_[currentImage],
        		.renderArea = screenRect
        	};

        	vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
        	vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline_);
        	vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout_, 0, 1, &descriptorSets_[currentImage], 0, nullptr);
        }
    
    开始 renderPass，绑定 pipeline 和 descriptorSets；

### VulkanClear
1. 构造函数

        VulkanClear::VulkanClear(VulkanRenderDevice& vkDev, VulkanImage depthTexture): RendererBase(vkDev, depthTexture)
        , shouldClearDepth(depthTexture.image != VK_NULL_HANDLE)
        {
        	if (!createColorAndDepthRenderPass(
        		vkDev, shouldClearDepth, &renderPass_, RenderPassCreateInfo{ .clearColor_ = true, .clearDepth_ = true, .flags_ = eRenderPassBit_First }))
        	{
        		printf("VulkanClear: failed to create render pass\n");
        		exit(EXIT_FAILURE);
        	}

        	createColorAndDepthFramebuffers(vkDev, renderPass_, depthTexture.imageView, swapchainFramebuffers_);
        }
    
    1. 创建 renderPass；
    2. 创建 swapchainFramebuffers_，注意：FrameBuffer 相当于 DescriptorSet，本身只是 SwapChainImage 的集合，不持有 image，所以可以每个 Layer 都创建一个，但是底层持有的 SwapChainImage 是同一份；
2. fillCommandBuffer

        void VulkanClear::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t swapFramebuffer)
        {
        	EASY_FUNCTION();

        	const VkClearValue clearValues[2] =
        	{
        		VkClearValue { .color = { 1.0f, 1.0f, 1.0f, 1.0f } },
        		VkClearValue { .depthStencil = { 1.0f, 0 } }
        	};

        	const VkRect2D screenRect = {
        		.offset = { 0, 0 },
        		.extent = {.width = framebufferWidth_, .height = framebufferHeight_ }
        	};

        	const VkRenderPassBeginInfo renderPassInfo = {
        		.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        		.renderPass = renderPass_,
        		.framebuffer = swapchainFramebuffers_[swapFramebuffer],
        		.renderArea = screenRect,
        		.clearValueCount = static_cast<uint32_t>(shouldClearDepth ? 2 : 1),
        		.pClearValues = &clearValues[0]
        	};

        	vkCmdBeginRenderPass( commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE );
        	vkCmdEndRenderPass( commandBuffer );
        }

    注意，这里只是清除 Framebuffer 的颜色和深度缓冲，不需要创建 Pipeline；

### VulkanFinish

    VulkanFinish::VulkanFinish(VulkanRenderDevice& vkDev, VulkanImage depthTexture): RendererBase(vkDev, depthTexture)
    {
    	if (!createColorAndDepthRenderPass(
    		vkDev, (depthTexture.image != VK_NULL_HANDLE), &renderPass_, RenderPassCreateInfo{ .clearColor_ = false, .clearDepth_ = false, .flags_ = eRenderPassBit_Last }))
    	{
    		printf("VulkanFinish: failed to create render pass\n");
    		exit(EXIT_FAILURE);
    	}

    	createColorAndDepthFramebuffers(vkDev, renderPass_, depthTexture.imageView, swapchainFramebuffers_);
    }

    void VulkanFinish::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
    {
    	EASY_FUNCTION();

    	const VkRect2D screenRect = {
    		.offset = { 0, 0 },
    		.extent = {.width = framebufferWidth_, .height = framebufferHeight_ }
    	};

    	const VkRenderPassBeginInfo renderPassInfo = {
    		.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
    		.renderPass = renderPass_,
    		.framebuffer = swapchainFramebuffers_[currentImage],
    		.renderArea = screenRect
    	};

    	vkCmdBeginRenderPass( commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE );
    	vkCmdEndRenderPass( commandBuffer );
    }

VulkanFinish 类似 VulkanClear，并不进行真正的渲染，只是将 FrameBuffer 中 Image 的 Layout 更新为是和呈现的布局，同时因为是最后的 Pass，所以不用清除缓存；

### ModelRenderer
上述两个 Renderer 只是热身，这次来看渲染模型的 Renderer 如何组织。
1. 类声明

        class ModelRenderer: public RendererBase
        {
        public:
        	ModelRenderer(VulkanRenderDevice& vkDev, const char* modelFile, const char* textureFile, uint32_t uniformDataSize);
        	ModelRenderer(VulkanRenderDevice& vkDev, bool useDepth, VkBuffer storageBuffer, VkDeviceMemory storageBufferMemory, uint32_t vertexBufferSize, uint32_t indexBufferSize, VulkanImage texture, VkSampler textureSampler, const std::vector<const char*>& shaderFiles, uint32_t uniformDataSize, bool useGeneralTextureLayout = true, VulkanImage externalDepth = { .image = VK_NULL_HANDLE }, bool deleteMeshData = true);
        	virtual ~ModelRenderer();

        	virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) override;

        	void updateUniformBuffer(VulkanRenderDevice& vkDev, uint32_t currentImage, const void* data, const size_t dataSize);

        	// HACK to allow sharing textures between multiple ModelRenderers
        	void freeTextureSampler() { textureSampler_ = VK_NULL_HANDLE; }

        private:
        	bool useGeneralTextureLayout_ = false;
        	bool isExternalDepth_ = false;
        	bool deleteMeshData_ = true;

        	size_t vertexBufferSize_;
        	size_t indexBufferSize_;

        	// 6. Storage Buffer with index and vertex data
        	VkBuffer storageBuffer_;
        	VkDeviceMemory storageBufferMemory_;

        	VkSampler textureSampler_;
        	VulkanImage texture_;

        	bool createDescriptorSet(VulkanRenderDevice& vkDev, uint32_t uniformDataSize);
        };
    
    相对于父类，新增了以下方法和属性：
    1. updateUniformBuffer，更新 uniform 变量；
    2. vertexBufferSize_、indexBufferSize_、storageBuffer_、storageBufferMemory_，存储顶点；
    3. textureSampler_、texture_，模型纹理；
2. 构造方法

        ModelRenderer::ModelRenderer(VulkanRenderDevice& vkDev, const char* modelFile, const char* textureFile, uint32_t uniformDataSize): RendererBase(vkDev, VulkanImage())
        {
        	// Resource loading part
        	if (!createTexturedVertexBuffer(vkDev, modelFile, &storageBuffer_, &storageBufferMemory_, &vertexBufferSize_, &indexBufferSize_))
        	{
        		printf("ModelRenderer: createTexturedVertexBuffer() failed\n");
        		exit(EXIT_FAILURE);
        	}

        	createTextureImage(vkDev, textureFile, texture_.image, texture_.imageMemory);
        	createImageView(vkDev.device, texture_.image, VK_FORMAT_R8G8B8A8_UNORM, VK_IMAGE_ASPECT_COLOR_BIT, &texture_.imageView);
        	createTextureSampler(vkDev.device, &textureSampler_);

        	if (!createDepthResources(vkDev, vkDev.framebufferWidth, vkDev.framebufferHeight, depthTexture_) ||
        		!createColorAndDepthRenderPass(vkDev, true, &renderPass_, RenderPassCreateInfo()) ||
        		!createUniformBuffers(vkDev, uniformDataSize) ||
        		!createColorAndDepthFramebuffers(vkDev, renderPass_, depthTexture_.imageView, swapchainFramebuffers_) ||
        		!createDescriptorPool(vkDev, 1, 2, 1, &descriptorPool_) ||
        		!createDescriptorSet(vkDev, uniformDataSize) ||
        		!createPipelineLayout(vkDev.device, descriptorSetLayout_, &pipelineLayout_) ||
        		!createGraphicsPipeline(vkDev, renderPass_, pipelineLayout_, {"data/shaders/chapter03/VK02.vert", "data/shaders/chapter03/VK02.frag", "data/shaders/chapter03/VK02.geom" }, &graphicsPipeline_))
        	{
        		printf("ModelRenderer: failed to create pipeline\n");
        		exit(EXIT_FAILURE);
        	}
        }
    
    1. 加载顶点数组的 storage buffer；
    2. 加载纹理 texture；
    3. 创建 uniform buffer、descriptor set、descriptor set layout、pipeline、render pass、framebuffer、depth image；
3. createDescriptorSet

        bool ModelRenderer::createDescriptorSet(VulkanRenderDevice& vkDev, uint32_t uniformDataSize)
        {
        	const std::array<VkDescriptorSetLayoutBinding, 4> bindings = {
        		descriptorSetLayoutBinding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
        		descriptorSetLayoutBinding(1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
        		descriptorSetLayoutBinding(2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
        		descriptorSetLayoutBinding(3, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, VK_SHADER_STAGE_FRAGMENT_BIT)
        	};

        	const VkDescriptorSetLayoutCreateInfo layoutInfo = {
        		.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
        		.pNext = nullptr,
        		.flags = 0,
        		.bindingCount = static_cast<uint32_t>(bindings.size()),
        		.pBindings = bindings.data()
        	};

        	VK_CHECK(vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout_));

        	std::vector<VkDescriptorSetLayout> layouts(vkDev.swapchainImages.size(), descriptorSetLayout_);

        	const VkDescriptorSetAllocateInfo allocInfo = {
        		.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
        		.pNext = nullptr,
        		.descriptorPool = descriptorPool_,
        		.descriptorSetCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
        		.pSetLayouts = layouts.data()
        	};

        	descriptorSets_.resize(vkDev.swapchainImages.size());

        	VK_CHECK(vkAllocateDescriptorSets(vkDev.device, &allocInfo, descriptorSets_.data()));

        	for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
        	{
        		VkDescriptorSet ds = descriptorSets_[i];

        		const VkDescriptorBufferInfo bufferInfo  = { uniformBuffers_[i], 0, uniformDataSize };
        		const VkDescriptorBufferInfo bufferInfo2 = { storageBuffer_, 0, vertexBufferSize_ };
        		const VkDescriptorBufferInfo bufferInfo3 = { storageBuffer_, vertexBufferSize_, indexBufferSize_ };
        		const VkDescriptorImageInfo  imageInfo   = { textureSampler_, texture_.imageView, useGeneralTextureLayout_ ? VK_IMAGE_LAYOUT_GENERAL : VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL };

        		const std::array<VkWriteDescriptorSet, 4> descriptorWrites = {
        			bufferWriteDescriptorSet(ds, &bufferInfo,  0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER),
        			bufferWriteDescriptorSet(ds, &bufferInfo2, 1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
        			bufferWriteDescriptorSet(ds, &bufferInfo3, 2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
        			imageWriteDescriptorSet( ds, &imageInfo,   3)
        		};

        		vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
        	}

        	return true;
        }
    
    descriptorSet，对于每个 layer，是最不一样的部分，由各个 layer 自己负责单独实现，步骤如下：
    1. 确定 bindings，主要包括对应 buffer 的类型，被使用的 shader 类型；
    2. 根据 bindings，创建 descriptorSetLayout；
    3. 根据 descriptorSetLayout，创建 descriptorSet 数组；
    4. 通过 vkUpdateDescriptorSets，填充 buffer 到 descriptorSet 中；
3. updateUniformBuffer

        void ModelRenderer::updateUniformBuffer(VulkanRenderDevice& vkDev, uint32_t currentImage, const void* data, const size_t dataSize)
        {
        	uploadBufferData(vkDev, uniformBuffersMemory_[currentImage], 0, data, dataSize);
        }
    
    更新 uniform 变量；
4. fillCommandBuffer

        void ModelRenderer::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
        {
        	EASY_FUNCTION();

        	beginRenderPass(commandBuffer, currentImage);

        	vkCmdDraw(commandBuffer, static_cast<uint32_t>(indexBufferSize_ / (sizeof(unsigned int))), 1, 0, 0);
        	vkCmdEndRenderPass(commandBuffer);
        }

    发起 DrawCall;

### VulkanCanvas
对于调试来说，line 和 plane 是很常用的手段，但是并不能在 vulkan 中很方便的画出，本模块是实现即时模式下的画线和画平面操作；
1. 类定义

        class VulkanCanvas: public RendererBase
        {
        public:
        	explicit VulkanCanvas(VulkanRenderDevice& vkDev, VulkanImage depth);
        	virtual ~VulkanCanvas();

        	virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) override;

        	void clear();
        	void line(const vec3& p1, const vec3& p2, const vec4& c);
        	void plane3d(const vec3& orig, const vec3& v1, const vec3& v2, int n1, int n2, float s1, float s2, const vec4& color, const vec4& outlineColor);
        	void updateBuffer(VulkanRenderDevice& vkDev, size_t currentImage);
        	void updateUniformBuffer(VulkanRenderDevice& vkDev, const glm::mat4& modelViewProj , float time, uint32_t currentImage);

        private:
        	struct VertexData
        	{
        		vec3 position;
        		vec4 color;
        	};

        	struct UniformBuffer
        	{
        		glm::mat4 mvp;
        		float time;
        	};

        	bool createDescriptorSet(VulkanRenderDevice& vkDev);

        	std::vector<VertexData> lines_;

        	// 7. Storage Buffer with index and vertex data
        	std::vector<VkBuffer> storageBuffer_;
        	std::vector<VkDeviceMemory> storageBufferMemory_;

        	static constexpr unsigned kMaxLinesCount = 65536;
        	static constexpr unsigned kMaxLinesDataSize = kMaxLinesCount * sizeof(VulkanCanvas::VertexData) * 2;
        };
    
    这里主要的不同点在于新增了 line、plane3d 等，和几何绘制相关的方法，以及 lines_，保存顶点的属性；
2. 构造函数

		VulkanCanvas::VulkanCanvas(VulkanRenderDevice& vkDev, VulkanImage depth): RendererBase(vkDev, depth)
		{
			const size_t imgCount = vkDev.swapchainImages.size();

			storageBuffer_.resize(imgCount);
			storageBufferMemory_.resize(imgCount);

			for(size_t i = 0 ; i < imgCount ; i++)
			{
				if (!createBuffer(vkDev.device, vkDev.physicalDevice, kMaxLinesDataSize,
					VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
					VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
					storageBuffer_[i], storageBufferMemory_[i]))
				{
					printf("VaulkanCanvas: createBuffer() failed\n");
					exit(EXIT_FAILURE);
				}
			}

			if (!createColorAndDepthRenderPass(vkDev, (depth.image != VK_NULL_HANDLE), &renderPass_, RenderPassCreateInfo()) ||
				!createUniformBuffers(vkDev, sizeof(UniformBuffer)) ||
				!createColorAndDepthFramebuffers(vkDev, renderPass_, depth.imageView, swapchainFramebuffers_) ||
				!createDescriptorPool(vkDev, 1, 1, 0, &descriptorPool_) ||
				!createDescriptorSet(vkDev) ||
				!createPipelineLayout(vkDev.device, descriptorSetLayout_, &pipelineLayout_) ||
				!createGraphicsPipeline(vkDev, renderPass_, pipelineLayout_, { "data/shaders/chapter04/Lines.vert", "data/shaders/chapter04/Lines.frag" }, &graphicsPipeline_, VK_PRIMITIVE_TOPOLOGY_LINE_LIST, (depth.image != VK_NULL_HANDLE), true ))
			{
				printf("VulkanCanvas: failed to create pipeline\n");
				exit(EXIT_FAILURE);
			}
		}

	注意：
	1. 由于顶点数据是动态变化的，所以这里需要每一个 swapChainImage 都对应一个 storageBuffer，并且为了更新数据方便，而且数据量不大的情况下，直接采用 VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT 标志，创建 buffer；
	2. createGraphicsPipeline 方法中的 图元类型选择 VK_PRIMITIVE_TOPOLOGY_LINE_LIST；
3. createDescriptorSet

		bool VulkanCanvas::createDescriptorSet(VulkanRenderDevice& vkDev)
		{
			const std::array<VkDescriptorSetLayoutBinding, 2> bindings = {
				descriptorSetLayoutBinding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				descriptorSetLayoutBinding(1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT)
			};

			const VkDescriptorSetLayoutCreateInfo layoutInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.bindingCount = static_cast<uint32_t>(bindings.size()),
				.pBindings = bindings.data()
			};

			VK_CHECK(vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout_));

			std::vector<VkDescriptorSetLayout> layouts(vkDev.swapchainImages.size(), descriptorSetLayout_);

			const VkDescriptorSetAllocateInfo allocInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
				.pNext = nullptr,
				.descriptorPool = descriptorPool_,
				.descriptorSetCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
				.pSetLayouts = layouts.data()
			};

			descriptorSets_.resize(vkDev.swapchainImages.size());

			VK_CHECK(vkAllocateDescriptorSets(vkDev.device, &allocInfo, descriptorSets_.data()));

			for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
			{
				VkDescriptorSet ds = descriptorSets_[i];

				const VkDescriptorBufferInfo bufferInfo  = { uniformBuffers_[i], 0, sizeof(UniformBuffer) };
				const VkDescriptorBufferInfo bufferInfo2 = { storageBuffer_[i], 0, kMaxLinesDataSize };

				const std::array<VkWriteDescriptorSet, 2> descriptorWrites = {
					bufferWriteDescriptorSet(ds, &bufferInfo,	0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo2, 1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
				};

				vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
			}

			return true;
		}

	创建 DescriptorSet，除了注意每个 swapChainImage 有自己的 storageBuffer 之外，没有什么特别的点；
4. fillCommandBuffer

		void VulkanCanvas::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
		{
			EASY_FUNCTION();

			if (lines_.empty())
				return;

			beginRenderPass(commandBuffer, currentImage);

			vkCmdDraw( commandBuffer, static_cast<uint32_t>(lines_.size()), 1, 0, 0 );
			vkCmdEndRenderPass( commandBuffer );
		}
		
	填充绘制命令
5. updateBuffer

		void VulkanCanvas::updateBuffer(VulkanRenderDevice& vkDev, size_t currentImage)
		{
			if (lines_.empty())
				return;

			const VkDeviceSize bufferSize = lines_.size() * sizeof(VertexData);

			uploadBufferData(vkDev, storageBufferMemory_[currentImage], 0, lines_.data(), bufferSize);
		}

	更新顶点缓冲；
6. 	updateUniformBuffer

		void VulkanCanvas::updateUniformBuffer(VulkanRenderDevice& vkDev, const glm::mat4& modelViewProj, float time, uint32_t currentImage)
		{
			const UniformBuffer ubo = {
				.mvp = modelViewProj,
				.time = time
			};

			uploadBufferData(vkDev, uniformBuffersMemory_[currentImage], 0, &ubo, sizeof(ubo));
		}
	
	更新 Uniform 缓冲；
7. 绘制线、面，以及清空

		void VulkanCanvas::clear()
		{
			lines_.clear();
		}

		void VulkanCanvas::line(const vec3& p1, const vec3& p2, const vec4& c)
		{
			lines_.push_back( { .position = p1, .color = c } );
			lines_.push_back( { .position = p2, .color = c } );
		}

		void VulkanCanvas::plane3d(const vec3& o, const vec3& v1, const vec3& v2, int n1, int n2, float s1, float s2, const vec4& color, const vec4& outlineColor)
		{
			line(o - s1 / 2.0f * v1 - s2 / 2.0f * v2, o - s1 / 2.0f * v1 + s2 / 2.0f * v2, outlineColor);
			line(o + s1 / 2.0f * v1 - s2 / 2.0f * v2, o + s1 / 2.0f * v1 + s2 / 2.0f * v2, outlineColor);

			line(o - s1 / 2.0f * v1 + s2 / 2.0f * v2, o + s1 / 2.0f * v1 + s2 / 2.0f * v2, outlineColor);
			line(o - s1 / 2.0f * v1 - s2 / 2.0f * v2, o + s1 / 2.0f * v1 - s2 / 2.0f * v2, outlineColor);

			for (int i = 1; i < n1; i++)
			{
				float t = ((float)i - (float)n1 / 2.0f) * s1 / (float)n1;
				const vec3 o1 = o + t * v1;
				line(o1 - s2 / 2.0f * v2, o1 + s2 / 2.0f * v2, color);
			}

			for (int i = 1; i < n2; i++)
			{
				const float t = ((float)i - (float)n2 / 2.0f) * s2 / (float)n2;
				const vec3 o2 = o + t * v2;
				line(o2 - s1 / 2.0f * v1, o2 + s1 / 2.0f * v1, color);
			}
		}

	这里分开说：
	1. clear 清空函数，清空数据，这里的实现逻辑并不支持在每一帧里动态修改、清除数据；
	2. line 划线函数，设置线的起点和终点；
	3. plane3d 绘制平面，o 是中心点，一般是 (0,y,0) 形式，表示沿着 y 轴的位置，然后 v1 和 v2 是扩展的两个方向，一般用 (1,0,0) 和 (0,0,1)，表示沿着 x 轴和 z 轴扩展，其实就是以 y 轴为中心，扩展一个和 y 轴垂直的平面，s 表示扩展的倍数，n 表示细分成多少网格；

### VulkanImGui
1. 类定义

		class ImGuiRenderer: public RendererBase
		{
		public:
			explicit ImGuiRenderer(VulkanRenderDevice& vkDev);
			explicit ImGuiRenderer(VulkanRenderDevice& vkDev, const std::vector<VulkanTexture>& textures);
			virtual ~ImGuiRenderer();

			virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) override;
			void updateBuffers(VulkanRenderDevice& vkDev, uint32_t currentImage, const ImDrawData* imguiDrawData);

		private:
			const ImDrawData* drawData = nullptr;

			bool createDescriptorSet(VulkanRenderDevice& vkDev);

			/* Descriptor set with multiple textures (for offscreen buffer display etc.) */
			bool createMultiDescriptorSet(VulkanRenderDevice& vkDev);

			std::vector<VulkanTexture> extTextures_;

			// storage buffer with index and vertex data
			VkDeviceSize bufferSize_;
			std::vector<VkBuffer> storageBuffer_;
			std::vector<VkDeviceMemory> storageBufferMemory_;

			VkSampler fontSampler_;
			VulkanImage font_;
		};

	注意 updateBuffers 函数的 ImDrawData 参数，里面保留了绘制 ImGUI 所需要的全部数据，并将其保存在 ```const ImDrawData* drawData = nullptr;``` 属性中；
2. createDescriptorSet

		constexpr uint32_t ImGuiVtxBufferSize = 512 * 1024 * sizeof(ImDrawVert);
		constexpr uint32_t ImGuiIdxBufferSize = 512 * 1024 * sizeof(uint32_t);

		bool ImGuiRenderer::createDescriptorSet(VulkanRenderDevice& vkDev)
		{
			const std::array<VkDescriptorSetLayoutBinding, 4> bindings = {
				descriptorSetLayoutBinding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				descriptorSetLayoutBinding(1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				descriptorSetLayoutBinding(2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				descriptorSetLayoutBinding(3, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, VK_SHADER_STAGE_FRAGMENT_BIT)
			};

			const VkDescriptorSetLayoutCreateInfo layoutInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.bindingCount = static_cast<uint32_t>(bindings.size()),
				.pBindings = bindings.data()
			};

			VK_CHECK(vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout_));

			std::vector<VkDescriptorSetLayout> layouts(vkDev.swapchainImages.size(), descriptorSetLayout_);

			const VkDescriptorSetAllocateInfo allocInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
				.pNext = nullptr,
				.descriptorPool = descriptorPool_,
				.descriptorSetCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
				.pSetLayouts = layouts.data()
			};

			descriptorSets_.resize(vkDev.swapchainImages.size());

			VK_CHECK(vkAllocateDescriptorSets(vkDev.device, &allocInfo, descriptorSets_.data()));

			for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
			{
				VkDescriptorSet ds = descriptorSets_[i];
				const VkDescriptorBufferInfo bufferInfo  = { uniformBuffers_[i], 0, sizeof(mat4) };
				const VkDescriptorBufferInfo bufferInfo2 = { storageBuffer_[i], 0, ImGuiVtxBufferSize };
				const VkDescriptorBufferInfo bufferInfo3 = { storageBuffer_[i], ImGuiVtxBufferSize, ImGuiIdxBufferSize };
				const VkDescriptorImageInfo  imageInfo   = { fontSampler_, font_.imageView, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL };

				const std::array<VkWriteDescriptorSet, 4> descriptorWrites = {
					bufferWriteDescriptorSet(ds, &bufferInfo,  0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo2, 1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo3, 2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
					imageWriteDescriptorSet( ds, &imageInfo,   3)
				};

				vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
			}

			return true;
		}

	和之前的函数类似，需要注意的是，这里一开始设定了 ImGuiVtxBufferSize 和 ImGuiIdxBufferSize，预先分配了顶点缓冲和索引缓冲的大小；
3. 构造函数

		ImGuiRenderer::ImGuiRenderer(VulkanRenderDevice& vkDev): RendererBase(vkDev, VulkanImage())
		{
			// Resource loading
			ImGuiIO& io = ImGui::GetIO();
			createFontTexture(io, "data/OpenSans-Light.ttf", vkDev, font_.image, font_.imageMemory);

			createImageView(vkDev.device, font_.image, VK_FORMAT_R8G8B8A8_UNORM, VK_IMAGE_ASPECT_COLOR_BIT, &font_.imageView);
			createTextureSampler(vkDev.device, &fontSampler_);

			// Buffer allocation
			const size_t imgCount = vkDev.swapchainImages.size();

			storageBuffer_.resize(imgCount);
			storageBufferMemory_.resize(imgCount);

			bufferSize_ = ImGuiVtxBufferSize + ImGuiIdxBufferSize;

			for(size_t i = 0 ; i < imgCount ; i++)
			{
				if (!createBuffer(vkDev.device, vkDev.physicalDevice, bufferSize_,
					VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
					VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
					storageBuffer_[i], storageBufferMemory_[i]))
				{
					printf("ImGuiRenderer: createBuffer() failed\n");
					exit(EXIT_FAILURE);
				}
			}

			// Pipeline creation
			if (!createColorAndDepthRenderPass(vkDev, false, &renderPass_, RenderPassCreateInfo()) ||
				!createColorAndDepthFramebuffers(vkDev, renderPass_, VK_NULL_HANDLE, swapchainFramebuffers_) ||
				!createUniformBuffers(vkDev, sizeof(mat4)) ||
				!createDescriptorPool(vkDev, 1, 2, 1, &descriptorPool_) ||
				!createDescriptorSet(vkDev) ||
				!createPipelineLayout(vkDev.device, descriptorSetLayout_, &pipelineLayout_) ||
				!createGraphicsPipeline(vkDev, renderPass_, pipelineLayout_,
					{ "data/shaders/chapter04/imgui.vert", "data/shaders/chapter04/imgui.frag" }, &graphicsPipeline_, VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST,
					true, true, true))
			{
				printf("ImGuiRenderer: pipeline creation failed\n");
				exit(EXIT_FAILURE);
			}
		}
	
	主要分为三部分：
	1. font_，加载字体纹理；
	2. storageBuffer_，分配几何数据缓存；
	3. graphicsPipeline_，创建绘图管线；
4. createFontTexture

		bool createFontTexture(ImGuiIO& io, const char* fontFile, VulkanRenderDevice& vkDev, VkImage& textureImage, VkDeviceMemory& textureImageMemory)
		{
			// Build texture atlas
			ImFontConfig cfg = ImFontConfig();
			cfg.FontDataOwnedByAtlas = false;
			cfg.RasterizerMultiply = 1.5f;
			cfg.SizePixels = 768.0f / 32.0f;
			cfg.PixelSnapH = true;
			cfg.OversampleH = 4;
			cfg.OversampleV = 4;
			ImFont* Font = io.Fonts->AddFontFromFileTTF(fontFile, cfg.SizePixels, &cfg);

			unsigned char* pixels = nullptr;
			int texWidth = 1, texHeight = 1;
			io.Fonts->GetTexDataAsRGBA32(&pixels, &texWidth, &texHeight);

			if (!pixels || !createTextureImageFromData(vkDev, textureImage, textureImageMemory, pixels, texWidth, texHeight, VK_FORMAT_R8G8B8A8_UNORM))
			{
				printf("Failed to load texture\n"); fflush(stdout);
				return false;
			}

			io.Fonts->TexID = (ImTextureID)0;
			io.FontDefault = Font;
			io.DisplayFramebufferScale = ImVec2(1, 1);

			return true;
		}
	
	添加字体时，使用到了 createFontTexture 函数，这个函数主要是给 io 的 Fonts->TexID 和 FontDefault 赋值，Fonts->TexID 赋值默认 0 即可，FontDefault 赋值需要通过 io.Fonts->AddFontFromFileTTF 加载 ttf 字体文件，同时我们通过 io.Fonts->GetTexDataAsRGBA32 获取 BitMap，并构造纹理对象；
5. fillCommandBuffer

		void ImGuiRenderer::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
		{
			EASY_FUNCTION();

			beginRenderPass(commandBuffer, currentImage);

			ImVec2 clipOff = drawData->DisplayPos;         // (0,0) unless using multi-viewports
			ImVec2 clipScale = drawData->FramebufferScale; // (1,1) unless using retina display which are often (2,2)

			int vtxOffset = 0;
			int idxOffset = 0;

			for (int n = 0; n < drawData->CmdListsCount; n++)
			{
				const ImDrawList* cmdList = drawData->CmdLists[n];

				for (int cmd = 0; cmd < cmdList->CmdBuffer.Size ; cmd++)
				{
					const ImDrawCmd* pcmd = &cmdList->CmdBuffer[cmd];

					addImGuiItem(framebufferWidth_, framebufferHeight_, commandBuffer, pcmd, clipOff, clipScale, idxOffset, vtxOffset, extTextures_, pipelineLayout_);
				}
				idxOffset += cmdList->IdxBuffer.Size;
				vtxOffset += cmdList->VtxBuffer.Size;
			}

			vkCmdEndRenderPass(commandBuffer);
		}
	
	这里通过 addImGuiItem 函数解析每一条 ImCMD
6. addImGuiItem

		void addImGuiItem(uint32_t width, uint32_t height, VkCommandBuffer commandBuffer, const ImDrawCmd* pcmd, ImVec2 clipOff, ImVec2 clipScale, int idxOffset, int vtxOffset, const std::vector<VulkanTexture>& textures, VkPipelineLayout pipelineLayout)
		{
			if (pcmd->UserCallback)
				return;

			// Project scissor/clipping rectangles into framebuffer space
			ImVec4 clipRect;
			clipRect.x = (pcmd->ClipRect.x - clipOff.x) * clipScale.x;
			clipRect.y = (pcmd->ClipRect.y - clipOff.y) * clipScale.y;
			clipRect.z = (pcmd->ClipRect.z - clipOff.x) * clipScale.x;
			clipRect.w = (pcmd->ClipRect.w - clipOff.y) * clipScale.y;

			if (clipRect.x < width && clipRect.y < height && clipRect.z >= 0.0f && clipRect.w >= 0.0f)
			{
				if (clipRect.x < 0.0f) clipRect.x = 0.0f;
				if (clipRect.y < 0.0f) clipRect.y = 0.0f;
				// Apply scissor/clipping rectangle
				const VkRect2D scissor = {
					.offset = { .x = (int32_t)(clipRect.x), .y = (int32_t)(clipRect.y) },
					.extent = { .width = (uint32_t)(clipRect.z - clipRect.x), .height = (uint32_t)(clipRect.w - clipRect.y) }
				};
				vkCmdSetScissor(commandBuffer, 0, 1, &scissor);

				// this is added in the Chapter 6: Using descriptor indexing in Vulkan to render an ImGui UI
				if (textures.size() > 0)
				{
					uint32_t texture = (uint32_t)(intptr_t)pcmd->TextureId;
					vkCmdPushConstants(commandBuffer, pipelineLayout, VK_SHADER_STAGE_FRAGMENT_BIT, 0, sizeof(uint32_t), (const void*)&texture);
				}

				vkCmdDraw(commandBuffer,
					pcmd->ElemCount,
					1,
					pcmd->IdxOffset + idxOffset,
					pcmd->VtxOffset + vtxOffset);
			}
		}
	
	这里注意两点：
	1. 使用 vkCmdSetScissor 设定裁剪窗口，这里在调用 createGraphicsPipeline 函数时，设置了 dynamicScissorState 参数为 true；
	2. 注意 vkCmdDraw 最后两个参数分别表示 firstVerticle 和 firstInstance，乍一看非常迷惑，其实理解了 ImGUI 的每条命令的数据结构后就不难理解了，附上 imgui.vert 的代码：

			#version 460

			layout(location = 0) out vec2 uv;
			layout(location = 1) out vec4 color;

			struct ImDrawVert{ float x, y, u, v; uint color; };

			layout(binding = 0) uniform  UniformBuffer { mat4   inMtx; } ubo;
			layout(binding = 1) readonly buffer SBO    { ImDrawVert data[]; } 			sbo;
			layout(binding = 2) readonly buffer IBO    { uint   data[]; } ibo;

			void main()
			{
				uint idx = ibo.data[gl_VertexIndex] + gl_BaseInstance;

				ImDrawVert v = sbo.data[idx];
				uv     = vec2(v.u, v.v);
				color  = unpackUnorm4x8(v.color);
				gl_Position = ubo.inMtx * vec4(v.x, v.y, 0.0, 1.0);
			}
		
		1. pcmd->ElemCount，设定了 gl_VertexIndex 的结束范围是  pcmd->ElemCount;
		2. pcmd->IdxOffset + idxOffset，设定了 gl_VertexIndex 的开始范围是  pcmd->IdxOffset + idxOffset，所以目前 gl_VertexIndex 的范围是[pcmd->IdxOffset + idxOffset, pcmd->ElemCount];
		3. pcmd->VtxOffset + vtxOffset，设定了 gl_BaseInstance 的值；
7. updateBuffers

		void ImGuiRenderer::updateBuffers(VulkanRenderDevice& vkDev, uint32_t currentImage, const ImDrawData* imguiDrawData)
		{
			drawData = imguiDrawData;

			const float L = drawData->DisplayPos.x;
			const float R = drawData->DisplayPos.x + drawData->DisplaySize.x;
			const float T = drawData->DisplayPos.y;
			const float B = drawData->DisplayPos.y + drawData->DisplaySize.y;

			const mat4 inMtx = glm::ortho(L, R, T, B);

			uploadBufferData(vkDev, uniformBuffersMemory_[currentImage], 0, glm::value_ptr(inMtx), sizeof(mat4));

			void* data = nullptr;
			vkMapMemory(vkDev.device, storageBufferMemory_[currentImage], 0, bufferSize_, 0, &data);

			ImDrawVert* vtx = (ImDrawVert*)data;
			for (int n = 0; n < drawData->CmdListsCount; n++)
			{
				const ImDrawList* cmdList = drawData->CmdLists[n];
				memcpy(vtx, cmdList->VtxBuffer.Data, cmdList->VtxBuffer.Size * sizeof(ImDrawVert));
				vtx += cmdList->VtxBuffer.Size;
			}

			uint32_t* idx = (uint32_t*)((uint8_t*)data + ImGuiVtxBufferSize);
			for (int n = 0; n < drawData->CmdListsCount; n++)
			{
				const ImDrawList* cmdList = drawData->CmdLists[n];
				const uint16_t* src = (const uint16_t*)cmdList->IdxBuffer.Data;

				for (int j = 0; j < cmdList->IdxBuffer.Size; j++)
					*idx++ = (uint32_t)*src++;
			}

			vkUnmapMemory(vkDev.device, storageBufferMemory_[currentImage]);
		}
	
	注意：
	1. imguiDrawData 的数据是在外面设定好了，然后传进来，该函数只是起到一个更新几何数据的作用；
	2. 遍历 drawData，填充几何数据缓冲；

### VulkanCube
通过 Cube，绘制场景背景
1. 类声明

		class CubeRenderer: public RendererBase
		{
		public:
			CubeRenderer(VulkanRenderDevice& vkDev, VulkanImage inDepthTexture, const char* textureFile);
			virtual ~CubeRenderer();

			virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) override;

			void updateUniformBuffer(VulkanRenderDevice& vkDev, uint32_t currentImage, const mat4& m);

		private:
			VkSampler textureSampler;
			VulkanImage texture;

			bool createDescriptorSet(VulkanRenderDevice& vkDev);
		};

	主要新增一个 texture，用来保存 cubemap；

2. 构造函数

		CubeRenderer::CubeRenderer(VulkanRenderDevice& vkDev, VulkanImage inDepthTexture, const char* textureFile): RendererBase(vkDev, inDepthTexture)
		{
			// Resource loading
			createCubeTextureImage(vkDev, textureFile, texture.image, texture.imageMemory);

			createImageView(vkDev.device, texture.image, VK_FORMAT_R32G32B32A32_SFLOAT, VK_IMAGE_ASPECT_COLOR_BIT, &texture.imageView, VK_IMAGE_VIEW_TYPE_CUBE, 6);
			createTextureSampler(vkDev.device, &textureSampler);

			// Pipeline initialization
			if (!createColorAndDepthRenderPass(vkDev, true, &renderPass_, RenderPassCreateInfo()) ||
				!createUniformBuffers(vkDev, sizeof(mat4)) ||
				!createColorAndDepthFramebuffers(vkDev, renderPass_, depthTexture_.imageView, swapchainFramebuffers_) ||
				!createDescriptorPool(vkDev, 1, 0, 1, &descriptorPool_) ||
				!createDescriptorSet(vkDev) ||
				!createPipelineLayout(vkDev.device, descriptorSetLayout_, &pipelineLayout_) ||
				!createGraphicsPipeline(vkDev, renderPass_, pipelineLayout_, { "data/shaders/chapter04/VKCube.vert", "data/shaders/chapter04/VKCube.frag" }, &graphicsPipeline_))
			{
				printf("CubeRenderer: failed to create pipeline\n");
				exit(EXIT_FAILURE);
			}
		}
	
	通过 createCubeTextureImage 函数创建一个 cubemap 的纹理对象；

3. shared/UtilsVulkan.cpp

		bool createCubeTextureImage(VulkanRenderDevice& vkDev, const char* filename, VkImage& textureImage, VkDeviceMemory& textureImageMemory, uint32_t* width, uint32_t* height)
		{
			int w, h, comp;
			const float* img = stbi_loadf(filename, &w, &h, &comp, 3);
			std::vector<float> img32(w * h * 4);

			float24to32(w, h, img, img32.data());

			if (!img) {
				printf("Failed to load [%s] texture\n", filename); fflush(stdout);
				return false;
			}

			stbi_image_free((void*)img);

			Bitmap in(w, h, 4, eBitmapFormat_Float, img32.data());
			Bitmap out = convertEquirectangularMapToVerticalCross(in);

			Bitmap cube = convertVerticalCrossToCubeMapFaces(out);

			if (width && height)
			{
				*width = w;
				*height = h;
			}

			return createTextureImageFromData(vkDev, textureImage, textureImageMemory,
				cube.data_.data(), cube.w_, cube.h_,
				VK_FORMAT_R32G32B32A32_SFLOAT,
				6, VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT);
		}

		bool createImageView(VkDevice device, VkImage image, VkFormat format, VkImageAspectFlags aspectFlags, VkImageView* imageView, VkImageViewType viewType, uint32_t layerCount, uint32_t mipLevels)
		{
			const VkImageViewCreateInfo viewInfo =
			{
				.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.image = image,
				.viewType = viewType,
				.format = format,
				.subresourceRange =
				{
					.aspectMask = aspectFlags,
					.baseMipLevel = 0,
					.levelCount = mipLevels,
					.baseArrayLayer = 0,
					.layerCount = layerCount
				}
			};

			return (vkCreateImageView(device, &viewInfo, nullptr, imageView) == VK_SUCCESS);
		}

	注意，这里生成 cubemap texture 的过程和普通的 texture 的过程没什么区别，主要是通过 layerCount 设置为 6，在生成 image 和 imageView 是都需要设置 6 个面（+x、-x、+y、-y、+z、-z）；

4. 其他方法

		bool CubeRenderer::createDescriptorSet(VulkanRenderDevice& vkDev)
		{
			const std::array<VkDescriptorSetLayoutBinding, 2> bindings = {
				descriptorSetLayoutBinding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				descriptorSetLayoutBinding(1, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, VK_SHADER_STAGE_FRAGMENT_BIT)
			};

			const VkDescriptorSetLayoutCreateInfo layoutInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.bindingCount = static_cast<uint32_t>(bindings.size()),
				.pBindings = bindings.data()
			};

			VK_CHECK(vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout_));

			std::vector<VkDescriptorSetLayout> layouts(vkDev.swapchainImages.size(), descriptorSetLayout_);

			const VkDescriptorSetAllocateInfo allocInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
				.pNext = nullptr,
				.descriptorPool = descriptorPool_,
				.descriptorSetCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
				.pSetLayouts = layouts.data()
			};

			descriptorSets_.resize(vkDev.swapchainImages.size());

			VK_CHECK(vkAllocateDescriptorSets(vkDev.device, &allocInfo, descriptorSets_.data()));

			for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
			{
				VkDescriptorSet ds = descriptorSets_[i];

				const VkDescriptorBufferInfo bufferInfo  = { uniformBuffers_[i], 0, sizeof(mat4) };
				const VkDescriptorImageInfo  imageInfo   = { textureSampler, texture.imageView, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL };

				const std::array<VkWriteDescriptorSet, 2> descriptorWrites = {
					bufferWriteDescriptorSet(ds, &bufferInfo,  0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER),
					imageWriteDescriptorSet( ds, &imageInfo,   1)
				};

				vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
			}

			return true;
		}

		void CubeRenderer::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
		{
			EASY_FUNCTION();

			beginRenderPass(commandBuffer, currentImage);

			vkCmdDraw(commandBuffer, 36, 1, 0, 0);

			vkCmdEndRenderPass(commandBuffer);
		}

		void CubeRenderer::updateUniformBuffer(VulkanRenderDevice& vkDev, uint32_t currentImage, const mat4& m)
		{
			uploadBufferData(vkDev, uniformBuffersMemory_[currentImage], 0, glm::value_ptr(m), sizeof(mat4));
		}

	其余部分和其他 Renderer 基本一致，注意我们会将 mvp 矩阵，传递给 cubemap，以便用来采样；

5. shader

		// VKCude.vert
		#version 460 core

		layout (location=0) out vec3 dir;

		layout(binding = 0) uniform UniformBuffer
		{
			mat4 mvp;
		} ubo;

		const vec3 pos[8] = vec3[8](
			vec3(-1.0,-1.0, 1.0),
			vec3( 1.0,-1.0, 1.0),
			vec3( 1.0, 1.0, 1.0),
			vec3(-1.0, 1.0, 1.0),

			vec3(-1.0,-1.0,-1.0),
			vec3( 1.0,-1.0,-1.0),
			vec3( 1.0, 1.0,-1.0),
			vec3(-1.0, 1.0,-1.0)
		);

		const int indices[36] = int[36](
			// front
			0, 1, 2, 2, 3, 0,
			// right
			1, 5, 6, 6, 2, 1,
			// back
			7, 6, 5, 5, 4, 7,
			// left
			4, 0, 3, 3, 7, 4,
			// bottom
			4, 5, 1, 1, 0, 4,
			// top
			3, 2, 6, 6, 7, 3
		);

		void main()
		{
			float cubeSize = 10.0;
			int idx = indices[gl_VertexIndex];
			gl_Position = ubo.mvp * vec4(cubeSize * pos[idx], 1.0);
			dir = pos[idx].xyz;
		}

		//VKCube.frag
		#version 460 core

		layout (location=0) in vec3 dir;

		layout (location=0) out vec4 outColor;

		layout (binding=1) uniform samplerCube texture1;

		void main()
		{
			outColor = texture(texture1, dir);
		};
	
	通过 Shader 可以看出来，其实就是将摄像机放到一个大立方体里面，然后从里往外看；

### VulkanMultiMeshRenderer
Vulkan 间接绘制的 Renderer，对于多个 Mesh，可以调用一次 DrawCall，从而降低性能瓶颈

1. 类声明

		class MultiMeshRenderer: public RendererBase
		{
		public:
			virtual void fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage) override;

			MultiMeshRenderer(
				VulkanRenderDevice& vkDev,
				const char* meshFile,
				const char* drawDataFile,
				const char* materialFile,
				const char* vtxShaderFile,
				const char* fragShaderFile);

			void updateIndirectBuffers(VulkanRenderDevice& vkDev, size_t currentImage, bool* visibility = nullptr);

			void updateGeometryBuffers(VulkanRenderDevice& vkDev, uint32_t vertexCount, uint32_t indexCount, const void* vertices, const void* indices);
			void updateMaterialBuffer(VulkanRenderDevice& vkDev, uint32_t materialSize, const void* materialData);

			void updateUniformBuffer(VulkanRenderDevice& vkDev, size_t currentImage, const mat4& m);
			void updateDrawDataBuffer(VulkanRenderDevice& vkDev, size_t currentImage, uint32_t drawDataSize, const void* drawData);
			void updateCountBuffer(VulkanRenderDevice& vkDev, size_t currentImage, uint32_t itemCount);

			virtual ~MultiMeshRenderer();

			uint32_t vertexBufferSize_;
			uint32_t indexBufferSize_;

		private:
			VulkanRenderDevice& vkDev;

			uint32_t maxVertexBufferSize_;
			uint32_t maxIndexBufferSize_;

			uint32_t maxShapes_;

			uint32_t maxDrawDataSize_;
			uint32_t maxMaterialSize_;

			// 6. Storage Buffer with index and vertex data
			VkBuffer storageBuffer_;
			VkDeviceMemory storageBufferMemory_;

			VkBuffer materialBuffer_;
			VkDeviceMemory materialBufferMemory_;

			std::vector<VkBuffer> indirectBuffers_;
			std::vector<VkDeviceMemory> indirectBuffersMemory_;

			std::vector<VkBuffer> drawDataBuffers_;
			std::vector<VkDeviceMemory> drawDataBuffersMemory_;

			// Buffer for draw count
			std::vector<VkBuffer> countBuffers_;
			std::vector<VkDeviceMemory> countBuffersMemory_;

			/* DrawData loaded from file. Converted to indirectBuffers[] and uploaded to drawDataBuffers[] */
			std::vector<DrawData> shapes;
			MeshData meshData_;

			bool createDescriptorSet(VulkanRenderDevice& vkDev);

			void loadDrawData(const char* drawDataFile);
		};

	首先，看下私有属性部分：
	1. maxXXX_ 等，一些固定的最大限制值；
	2. storageBuffer_、storageBufferMemory_，用来存储 indexData 和 vertexData；
	3. materialBuffer_、materialBufferMemory_，用来存储材质数据；
	4. indirectBuffers_、indirectBuffersMemory_，每帧的间接绘制命令；
	5. drawDataBuffers_、drawDataBuffersMemory_，每帧变更的 mesh 数据；
	6. countBuffers_、countBuffersMemory_，每帧的 DrawCount；
	7. std::vector<DrawData> shapes，从 drawData 文件读取的，每个 mesh 对应的信息；
	8. MeshData meshData_，从 mesh 文件读取的，合并后的几何数据；

2. createDescriptorSet

		bool MultiMeshRenderer::createDescriptorSet(VulkanRenderDevice& vkDev)
		{
			const std::array<VkDescriptorSetLayoutBinding, 5> bindings = {
				descriptorSetLayoutBinding(0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				/* vertices [part of this.storageBuffer] */
				descriptorSetLayoutBinding(1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				/* indices [part of this.storageBuffer] */
				descriptorSetLayoutBinding(2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				/* draw data [this.drawDataBuffer] */
				descriptorSetLayoutBinding(3, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_VERTEX_BIT),
				/* material data [this.materialBuffer] */
				descriptorSetLayoutBinding(4, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_FRAGMENT_BIT)
			};

			const VkDescriptorSetLayoutCreateInfo layoutInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.bindingCount = static_cast<uint32_t>(bindings.size()),
				.pBindings = bindings.data()
			};

			VK_CHECK(vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout_));

			std::vector<VkDescriptorSetLayout> layouts(vkDev.swapchainImages.size(), descriptorSetLayout_);

			const VkDescriptorSetAllocateInfo allocInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
				.pNext = nullptr,
				.descriptorPool = descriptorPool_,
				.descriptorSetCount = static_cast<uint32_t>(vkDev.swapchainImages.size()),
				.pSetLayouts = layouts.data()
			};

			descriptorSets_.resize(vkDev.swapchainImages.size());

			VK_CHECK(vkAllocateDescriptorSets(vkDev.device, &allocInfo, descriptorSets_.data()));

			for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
			{
				VkDescriptorSet ds = descriptorSets_[i];

				const VkDescriptorBufferInfo bufferInfo  = { uniformBuffers_[i], 0, sizeof(mat4) };
				const VkDescriptorBufferInfo bufferInfo2 = { storageBuffer_, 0, maxVertexBufferSize_ };
				const VkDescriptorBufferInfo bufferInfo3 = { storageBuffer_, maxVertexBufferSize_, maxIndexBufferSize_ };
				const VkDescriptorBufferInfo bufferInfo4 = { drawDataBuffers_[i], 0, maxDrawDataSize_ };
				const VkDescriptorBufferInfo bufferInfo5 = { materialBuffer_, 0, maxMaterialSize_ };

				const std::array<VkWriteDescriptorSet, 5> descriptorWrites = {
					bufferWriteDescriptorSet(ds, &bufferInfo,  0, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo2, 1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo3, 2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo4, 3, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER),
					bufferWriteDescriptorSet(ds, &bufferInfo5, 4, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
				};

				vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
			}

			return true;
		}

		//VK01.vert
		#version 460

		layout(location = 0) out vec3 uvw;

		struct ImDrawVert   { float x, y, z; float u, v; float nx, ny, nz; };
		struct DrawData {
			uint mesh;
			uint material;
			uint lod;
			uint indexOffset;
			uint vertexOffset;
			uint transformIndex;
		};
		struct MaterialData { uint tex2D; };

		layout(binding = 0) uniform  UniformBuffer { mat4   inMtx; } ubo;
		layout(binding = 1) readonly buffer SBO    { ImDrawVert data[]; } sbo;
		layout(binding = 2) readonly buffer IBO    { uint   data[]; } ibo;
		layout(binding = 3) readonly buffer DrawBO { DrawData data[]; } drawDataBuffer;

		void main()
		{
			DrawData dd = drawDataBuffer.data[gl_BaseInstance];

			uint refIdx = dd.indexOffset + gl_VertexIndex;
			ImDrawVert v = sbo.data[ibo.data[refIdx] + dd.vertexOffset];

			uvw = normalize(vec3(v.x, v.y, v.z));

		//	mat4 xfrm(1.0); // = transpose(drawDataBuffer.data[gl_BaseInstance].xfrm);

			gl_Position = ubo.inMtx /* xfrm*/ * vec4(v.x, v.y, v.z, 1.0);
		}
	
	这里和之前的创建 descriptorSet 的过程类似，可以看出，主要将 uniform、顶点、索引和偏移信息传递进来；

3. updateXXXBuffer

		void MultiMeshRenderer::updateUniformBuffer(VulkanRenderDevice& vkDev, size_t currentImage, const mat4& m)
		{
			uploadBufferData(vkDev, uniformBuffersMemory_[currentImage], 0, glm::value_ptr(m), sizeof(mat4));
		}

		void MultiMeshRenderer::updateGeometryBuffers(VulkanRenderDevice& vkDev, uint32_t vertexCount, uint32_t indexCount, const void* vertices, const void* indices)
		{
			uploadBufferData(vkDev, storageBufferMemory_, 0, vertices, vertexCount);
			uploadBufferData(vkDev, storageBufferMemory_, maxVertexBufferSize_, indices, indexCount);
		}

		void MultiMeshRenderer::updateDrawDataBuffer(VulkanRenderDevice& vkDev, size_t currentImage, uint32_t drawDataSize, const void* drawData)
		{
			uploadBufferData(vkDev, drawDataBuffersMemory_[currentImage], 0, drawData, drawDataSize);
		}

		void MultiMeshRenderer::updateMaterialBuffer(VulkanRenderDevice& vkDev, uint32_t materialSize, const void* materialData)
		{
		}

		void MultiMeshRenderer::updateCountBuffer(VulkanRenderDevice& vkDev, size_t currentImage, uint32_t itemCount)
		{
			uploadBufferData(vkDev, countBuffersMemory_[currentImage], 0, &itemCount, sizeof(uint32_t));
		}

	上述 buffer，都是 HOST 可见的，直接通过 upload 方法更新；

4. updateIndirectBuffers

		void MultiMeshRenderer::updateIndirectBuffers(VulkanRenderDevice& vkDev, size_t currentImage, bool* visibility)
		{
			VkDrawIndirectCommand* data = nullptr;
			vkMapMemory(vkDev.device, indirectBuffersMemory_[currentImage], 0, 2 * sizeof(VkDrawIndirectCommand), 0, (void **)&data);

			for (uint32_t i = 0 ; i < maxShapes_ ; i++)
			{
				const uint32_t j = shapes[i].meshIndex;
				const uint32_t lod = shapes[i].LOD;
				data[i] = {
					.vertexCount = meshData_.meshes_[j].getLODIndicesCount(lod),
					.instanceCount = visibility ? (visibility[i] ? 1u : 0u) : 1u,
					.firstVertex = 0,
					.firstInstance = i
				};
			}
			vkUnmapMemory(vkDev.device, indirectBuffersMemory_[currentImage]);
		}
	
	更新间接绘制命令参数的 indirectBuffersMemory_；

5. fillCommandBuffer

		void MultiMeshRenderer::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage)
		{
			beginRenderPass(commandBuffer, currentImage);
			/* For CountKHR (Vulkan 1.1) we may use indirect rendering with GPU-based object counter */
			/// vkCmdDrawIndirectCountKHR(commandBuffer, indirectBuffers_[currentImage], 0, countBuffers_[currentImage], 0, maxShapes_, sizeof(VkDrawIndirectCommand));
			/* For Vulkan 1.0 vkCmdDrawIndirect is enough */
			vkCmdDrawIndirect(commandBuffer, indirectBuffers_[currentImage], 0, maxShapes_, sizeof(VkDrawIndirectCommand));

			vkCmdEndRenderPass(commandBuffer);
		}

	调用 vkCmdDrawIndirect，触发间接绘制；

6. 构造函数

		MultiMeshRenderer::MultiMeshRenderer(
			VulkanRenderDevice& vkDev,
			const char* meshFile,
			const char* drawDataFile,
			const char* materialFile,
			const char* vertShaderFile,
			const char* fragShaderFile) :
			vkDev(vkDev),
			RendererBase(vkDev, VulkanImage())
		{
			if (!createColorAndDepthRenderPass(vkDev, false, &renderPass_, RenderPassCreateInfo()))
			{
				printf("Failed to create render pass\n");
				exit(EXIT_FAILURE);
			}

			framebufferWidth_ = vkDev.framebufferWidth;
			framebufferHeight_ = vkDev.framebufferHeight;

			createDepthResources(vkDev, framebufferWidth_, framebufferHeight_, depthTexture_);

			loadDrawData(drawDataFile);

			MeshFileHeader header = loadMeshData(meshFile, meshData_);

			const uint32_t indirectDataSize = maxShapes_ * sizeof(VkDrawIndirectCommand);
			maxDrawDataSize_ = maxShapes_ * sizeof(DrawData);
			maxMaterialSize_ = 1024;

			countBuffers_.resize(vkDev.swapchainImages.size());
			countBuffersMemory_.resize(vkDev.swapchainImages.size());

			drawDataBuffers_.resize(vkDev.swapchainImages.size());
			drawDataBuffersMemory_.resize(vkDev.swapchainImages.size());

			indirectBuffers_.resize(vkDev.swapchainImages.size());
			indirectBuffersMemory_.resize(vkDev.swapchainImages.size());

			if (!createBuffer(vkDev.device, vkDev.physicalDevice, maxMaterialSize_,
				VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
				VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
				materialBuffer_, materialBufferMemory_))
			{
				printf("Cannot create material buffer\n");
				fflush(stdout);
				exit(EXIT_FAILURE);
			}

			maxVertexBufferSize_ = header.vertexDataSize;
			maxIndexBufferSize_ = header.indexDataSize;

			VkPhysicalDeviceProperties devProps;
		        vkGetPhysicalDeviceProperties(vkDev.physicalDevice, &devProps);
			const uint32_t offsetAlignment = static_cast<uint32_t>(devProps.limits.minStorageBufferOffsetAlignment);
			if ((maxVertexBufferSize_ & (offsetAlignment - 1)) != 0)
			{
				int floats = (offsetAlignment - (maxVertexBufferSize_ & (offsetAlignment - 1))) / sizeof(float);
				for (int ii = 0; ii < floats; ii++)
					meshData_.vertexData_.push_back(0);
				maxVertexBufferSize_ = (maxVertexBufferSize_ + offsetAlignment) & ~(offsetAlignment - 1);
			}

			if (!createBuffer(vkDev.device, vkDev.physicalDevice, maxVertexBufferSize_ + maxIndexBufferSize_,
				VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
				VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
				storageBuffer_, storageBufferMemory_))
			{
				printf("Cannot create vertex/index buffer\n"); fflush(stdout);
				exit(EXIT_FAILURE);
			}

			updateGeometryBuffers(vkDev, header.vertexDataSize, header.indexDataSize, meshData_.vertexData_.data(), meshData_.indexData_.data());

			for (size_t i = 0; i < vkDev.swapchainImages.size(); i++)
			{
				if (!createBuffer(vkDev.device, vkDev.physicalDevice, indirectDataSize,
					VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT, // | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
					VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, /* for debugging we make it host-visible */
					indirectBuffers_[i], indirectBuffersMemory_[i]))
				{
					printf("Cannot create indirect buffer\n"); fflush(stdout);
					exit(EXIT_FAILURE);
				}

				updateIndirectBuffers(vkDev, i);

				if (!createBuffer(vkDev.device, vkDev.physicalDevice, maxDrawDataSize_,
					VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
					VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, /* for debugging we make it host-visible */
					drawDataBuffers_[i], drawDataBuffersMemory_[i]))
				{
					printf("Cannot create draw data buffer\n"); fflush(stdout);
					exit(EXIT_FAILURE);
				}

				updateDrawDataBuffer(vkDev, i, maxDrawDataSize_, shapes.data());

				if (!createBuffer(vkDev.device, vkDev.physicalDevice, sizeof(uint32_t),
					VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT,
					VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, /* for debugging we make it host-visible */
					countBuffers_[i], countBuffersMemory_[i]))
				{
					printf("Cannot create count buffer\n");
					fflush(stdout);
					exit(EXIT_FAILURE);
				}

				updateCountBuffer(vkDev, i, maxShapes_);
			}

			if (!createUniformBuffers(vkDev, sizeof(mat4)) ||
				!createColorAndDepthFramebuffers(vkDev, renderPass_, VK_NULL_HANDLE, swapchainFramebuffers_) ||
				!createDescriptorPool(vkDev, 1, 4, 0, &descriptorPool_) ||
				!createDescriptorSet(vkDev) ||
				!createPipelineLayout(vkDev.device, descriptorSetLayout_, &pipelineLayout_) ||
				!createGraphicsPipeline(vkDev, renderPass_, pipelineLayout_, { vertShaderFile, fragShaderFile }, &graphicsPipeline_))
			{
				printf("Failed to create pipeline\n"); fflush(stdout);
				exit(EXIT_FAILURE);
			}
		}

	一步步分析，构造器做了什么：
	1. createColorAndDepthRenderPass，构造renderPass；
	2. createDepthResources，创建深度缓冲；
	3. loadDrawData(drawDataFile)，加载 drawData 文件到 shapes 变量：

			void MultiMeshRenderer::loadDrawData(const char* drawDataFile)
			{
				FILE* f = fopen(drawDataFile, "rb");

				if (!f) {
					printf("Unable to open draw data file. Run MeshConvert first\n");
					exit(255);
				}

				fseek(f, 0, SEEK_END);
				size_t fsize = ftell(f);
				fseek(f, 0, SEEK_SET);
				maxShapes_ = static_cast<uint32_t>(fsize / sizeof(DrawData));

				printf("Reading draw data items: %d\n", (int)maxShapes_); fflush(stdout);

				shapes.resize(maxShapes_);

				if (fread(shapes.data(), sizeof(DrawData), maxShapes_, f) != maxShapes_) {
					printf("Unable to read draw data\n");
					exit(255);
				}

				fclose(f);
			}
	4. MeshFileHeader header = loadMeshData(meshFile, meshData_)，加载 mesh 文件到 meshData_ 变量；

			MeshFileHeader loadMeshData(const char* meshFile, MeshData& out)
			{
				MeshFileHeader header;

				FILE* f = fopen(meshFile, "rb");

				assert(f); // Did you forget to run "Ch5_Tool05_MeshConvert"?

				if (!f)
				{
					printf("Cannot open %s. Did you forget to run \"Ch5_Tool05_MeshConvert\"?\n", meshFile);
					exit(EXIT_FAILURE);
				}

				if (fread(&header, 1, sizeof(header), f) != sizeof(header))
				{
					printf("Unable to read mesh file header\n");
					exit(EXIT_FAILURE);
				}

				out.meshes_.resize(header.meshCount);
				if (fread(out.meshes_.data(), sizeof(Mesh), header.meshCount, f) != header.meshCount)
				{
					printf("Could not read mesh descriptors\n");
					exit(EXIT_FAILURE);
				}
				out.boxes_.resize(header.meshCount);
				if (fread(out.boxes_.data(), sizeof(BoundingBox), header.meshCount, f) != header.meshCount)
				{
					printf("Could not read bounding boxes\n");
					exit(255);
				}

				out.indexData_.resize(header.indexDataSize / sizeof(uint32_t));
				out.vertexData_.resize(header.vertexDataSize / sizeof(float));

				if ((fread(out.indexData_.data(), 1, header.indexDataSize, f) != header.indexDataSize) ||
					(fread(out.vertexData_.data(), 1, header.vertexDataSize, f) != header.vertexDataSize))
				{
					printf("Unable to read index/vertex data\n");
					exit(255);
				}

				fclose(f);

				return header;
			}
	5. 创建 storageBuffer，存储顶点和索引数据，注意这里要对使用的内存进行对齐，下面解释一下为什么要对齐，以及如何对齐：
		1. 为什么要对齐，简而言之，是为了让 CPU/GPU 的读取更有效率，参考[C/C++内存对齐详解](http://www.codebaoku.com/tech/tech-memory-align.html);
		2. 如何进行对齐，首先我们要想对齐在数学上的本质是什么？其实就是模运算，比如以 4 字节对齐，那么
				
				1 -> 4
				2 -> 4
				3 -> 4
				4 -> 4
				5 -> 8
				6 -> 8
				7 -> 8
				8 -> 8
			对齐，其实就是把模运算里面空缺的部分补齐，我们再来看下具体代码：

				const uint32_t offsetAlignment = static_cast<uint32_t>(devProps.limits.minStorageBufferOffsetAlignment);
				if ((maxVertexBufferSize_ & (offsetAlignment - 1)) != 0)
				{
					int floats = (offsetAlignment - (maxVertexBufferSize_ & (offsetAlignment - 1))) / sizeof(float);
					for (int ii = 0; ii < floats; ii++)
						meshData_.vertexData_.push_back(0);
					maxVertexBufferSize_ = (maxVertexBufferSize_ + offsetAlignment) & ~(offsetAlignment - 1);
				}
			`maxVertexBufferSize_ & (offsetAlignment - 1)`，本质上等于以下模运算：`maxVertexBufferSize_ % offsetAlignment`，也就是获取按照 offsetAlignment 求模后的余数，
			对于`(maxVertexBufferSize_ + offsetAlignment) & ~(offsetAlignment - 1)`，其实是先把当前的数字扩大一个 offsetAlignment 的大小，然后再把求模后多余的部分截断，这里的原理可以参考这篇文章：[C++内存对齐-位运算公式](https://blog.csdn.net/weixin_50523841/article/details/121871878);
	6. 创建并更新 drawData、count 和 indirect Buffer；
	7. 其余方法和一般 renderer 类似；  
	
	注意，这里为了使用方便，把 mesh、drawData、count、indirect 为了方便，把 buffer 声明成 HOST 可见的，其实对于 CPU 之后不会再读写更改的数据，可以通过 stagingBuffer 直接复制到 GPU 中，可以提升性能，其中和 compute shader 结合后，尤其可以在 GPU 端自己完成 Cull 和 LOD 的计算等，可以减轻 CPU 端的性能负担；