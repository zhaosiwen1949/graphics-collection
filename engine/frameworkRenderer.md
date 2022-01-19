# 引擎架构之 FrameworkRenderer 封装

VulkanApp（[引擎架构之应用封装](engine/application.md)）主要封装了在 Vulkan 绘制中，每次绘制都一定会使用的部分，主要包括 FrameBuffer、SwapChain 及其 Image 对象、Device 和 Graphic\Compute Queue、Command Pool 和 Command Buffer，以及创建 Vulkan 资源的通用方法 VulkanResource（该对象介绍参见 [引擎架构之资源管理](engine/resource.md)），并且把 Vulkan 从准备上下文资源，一直到提交到呈现队列的过程都封装好了，甚至提供好了 Command 录制的上下文环境，留给 Renderer 处理的就剩 DescriptorSet、Pipeline、RenderPass 以及 DrawCall 的提交。

下面从 Renderer 开始，详细分析。

## Renderer

        struct Renderer
        {
        	Renderer(VulkanRenderContext& c)
        	: processingWidth(c.vkDev.framebufferWidth)
        	, processingHeight(c.vkDev.framebufferHeight)
        	, ctx_(c)
        	{}
        
        	virtual void fillCommandBuffer(VkCommandBuffer cmdBuffer, size_t currentImage, VkFramebuffer fb = VK_NULL_HANDLE, VkRenderPass rp = VK_NULL_HANDLE) = 0;
        	virtual void updateBuffers(size_t currentImage) {}
        
        	inline void updateUniformBuffer(uint32_t currentImage, const uint32_t offset, const uint32_t size, const void* data) {
        		uploadBufferData(ctx_.vkDev, uniforms_[currentImage].memory, offset, data, size);
        	}
        
        	void initPipeline(const std::vector<const char*>& shaders, const PipelineInfo& pInfo, uint32_t vtxConstSize = 0, uint32_t fragConstSize = 0)
        	{
        		pipelineLayout_ = ctx_.resources.addPipelineLayout(descriptorSetLayout_, vtxConstSize, fragConstSize);
        		graphicsPipeline_ = ctx_.resources.addPipeline(renderPass_.handle, pipelineLayout_, shaders, pInfo);
        	}
        
        	PipelineInfo initRenderPass(const PipelineInfo& pInfo, const std::vector<VulkanTexture>& outputs,
        		RenderPass renderPass = RenderPass(),
        		RenderPass fallbackPass = RenderPass())
        	{
        		PipelineInfo outInfo = pInfo;
        		if (!outputs.empty()) // offscreen rendering
        		{
        			printf("Creating framebuffer (outputs = %d). Output0: %dx%d; Output1: %dx%d\n", 
        				(int)outputs.size(), outputs[0].width, outputs[0].height,
        				(outputs.size() > 1 ? outputs[1].width : 0), (outputs.size() > 1 ? outputs[1].height : 0));
        			fflush(stdout);
        
        			processingWidth = outputs[0].width;
        			processingHeight = outputs[0].height;
        
        			outInfo.width = processingWidth;
        			outInfo.height = processingHeight;
        
        			renderPass_  = (renderPass.handle != VK_NULL_HANDLE) ? renderPass :
        					((isDepthFormat(outputs[0].format) && (outputs.size() == 1)) ? ctx_.resources.addDepthRenderPass(outputs) : ctx_.resources.addRenderPass(outputs, RenderPassCreateInfo(), true));
        			framebuffer_ = ctx_.resources.addFramebuffer(renderPass_, outputs);
        		} else
        		{
        			renderPass_ = (renderPass.handle != VK_NULL_HANDLE) ? renderPass : fallbackPass;
        		}
        		return outInfo;
        	}
        
        	void beginRenderPass(VkRenderPass rp, VkFramebuffer fb, VkCommandBuffer commandBuffer, size_t currentImage)
        	{
        		const VkClearValue clearValues[2] = {
        			VkClearValue { .color = { 1.0f, 1.0f, 1.0f, 1.0f } },
        			VkClearValue { .depthStencil = { 1.0f, 0 } }
        		};
        
        		const VkRect2D rect {
        			.offset = { 0, 0 },
        			.extent = { .width = processingWidth, .height = processingHeight }
        		};
        
        		ctx_.beginRenderPass(commandBuffer, rp, currentImage, rect,
        			fb,
        			(renderPass_.info.clearColor_ ? 1u : 0u) + (renderPass_.info.clearDepth_ ? 1u : 0u),
        			renderPass_.info.clearColor_ ? &clearValues[0] : (renderPass_.info.clearDepth_ ? &clearValues[1] : nullptr));
        
        		vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline_);
        		vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout_, 0, 1, &descriptorSets_[currentImage], 0, nullptr);
        	}
        
        	VkFramebuffer framebuffer_ = nullptr;
        	RenderPass renderPass_;
        
        	uint32_t processingWidth;
        	uint32_t processingHeight;
        
        	// Updating individual textures (9 is the binding in our Chapter7-Chapter9 IBL scene shaders)
        	void updateTexture(uint32_t textureIndex, VulkanTexture newTexture, uint32_t bindingIndex = 9)
        	{
        		for (auto ds: descriptorSets_)
        			updateTextureInDescriptorSetArray(ctx_.vkDev, ds, newTexture, textureIndex, bindingIndex);
        	}
        
        protected:
        	VulkanRenderContext& ctx_;
        
        	// Descriptor set (layout + pool + sets) -> uses uniform buffers, textures, framebuffers
        	VkDescriptorSetLayout descriptorSetLayout_ = nullptr;
        	VkDescriptorPool descriptorPool_ = nullptr;
        	std::vector<VkDescriptorSet> descriptorSets_;
        
        	// 4. Pipeline & render pass (using DescriptorSets & pipeline state options)
        	VkPipelineLayout pipelineLayout_ = nullptr;
        	VkPipeline graphicsPipeline_ = nullptr;
        
        	std::vector<VulkanBuffer> uniforms_;
        };
先来看属性，比对一个函数，可以主要分为入参、函数体、结果三部分：
1. public 权限的 framebuffer_ 和 renderPass_，通过 initRenderPass 设置，通过 beginRenderPass 使用，这里之所以是 public 权限，是为了在 VulkanApp 中访问，之后会作为 fillCommandBuffer 的参数传回来，这里的 framebuffer_ 和 renderPass_ 就相当于绘制的结果部分；
2. protected 权限的 descriptorSetLayout_、descriptorPool_、descriptorSets_，主要对应于入参部分，是输入数据的部分；
3. protected 权限的 pipelineLayout_ 和 graphicsPipeline_，主要是用来设置 pipeline 管线的各个部分的参数，以及绑定各个阶段的 shader，相当于函数的函数体；
4. `VulkanRenderContext& ctx_` 绘制上下文，里面有 vk、vkDev 和 vkResource；
5. `std::vector<VulkanBuffer> uniforms_` 保存 uniform 数据；

再看函数部分：
1. 虚函数 `fillCommandBuffer` 和 `updateBuffers`，由子类继承实现；
2. `updateUniformBuffer` 和 `updateTexture`，用来更新 uniform 缓存和 texture 纹理的绑定关系；
3. `initPipeline` 初始化渲染管线相关属性 pipelineLayout_、graphicsPipeline_；
4. `initRenderPass` 初始化 renderPass 和 framebuffer；
5. `beginRenderPass` 命令开始后绑定了 renderPass、pipeline 和 descriptorSet；

## MultiRenderer
MultiRenderer 是 Renderer 最重要的子类，负责根据场景数据绘制模型，这里的主要部分有三个：
1. 根据 VKSceneData，在构造函数中创建 descriptorSet；
2. `fillCommandBuffer` 命令绘制图像；
3. `updateBuffers` 命令更新缓存；
下面一一分析

### 场景数据 VKSceneData

		struct VKSceneData
		{
			VKSceneData(VulkanRenderContext& ctx,
				const char* meshFile,
				const char* sceneFile,
				const char* materialFile,
				VulkanTexture envMap,
				VulkanTexture irradianceMap,
				bool asyncLoad = false);

			// PBR 需要的环境贴图
			VulkanTexture envMapIrradiance_;
			VulkanTexture envMap_;
			VulkanTexture brdfLUT_;

			// storage buffer 传给 shader
			VulkanBuffer material_;
			VulkanBuffer transforms_;

			// 绘制上下文
			VulkanRenderContext& ctx;

			// 纹理贴图数组，用来绑定 uniform
			TextureArrayAttachment allMaterialTextures;

			// buffer 缓冲数组，用来绑定 uniform
			BufferAttachment indexBuffer_;
			BufferAttachment vertexBuffer_;

			// Mesh 几何数据
			MeshData meshData_;

			// 场景数据
			Scene scene_;

			// 材质数据
			std::vector<MaterialDescription> materials_;

			// transform 数组，保存 global trnasform 信息
			std::vector<glm::mat4> shapeTransforms_;
			
			// DrawData 信息
			std::vector<DrawData> shapes_;

			void loadScene(const char* sceneFile);
			void loadMeshes(const char* meshFile);

			void convertGlobalToShapeTransforms();
			void recalculateAllTransforms();
			void uploadGlobalTransforms();

			void updateMaterial(int matIdx);

			/* Chapter 9, async loading */
			struct LoadedImageData
			{
				int index_ = 0;
				int w_ = 0;
				int h_ = 0;
				const uint8_t* img_ = nullptr;
			};

			// 加载纹理图片文件
			std::vector<std::string> textureFiles_;
			std::vector<LoadedImageData> loadedFiles_;
			std::mutex loadedFilesMutex_;

		private:
			// 多线程相关
			tf::Taskflow taskflow_;
			tf::Executor executor_;
		};

		struct MeshData
		{
			std::vector<uint32_t> indexData_;
			std::vector<float> vertexData_;
			std::vector<Mesh> meshes_;
			std::vector<BoundingBox> boxes_;
		};

		struct Scene
		{
			// local transformations for each node and global transforms
			// + an array of 'dirty/changed' local transforms
			std::vector<mat4> localTransform_;
			std::vector<mat4> globalTransform_;

			// list of nodes whose global transform must be recalculated
			std::vector<int> changedAtThisFrame_[MAX_NODE_LEVEL];

			// Hierarchy component
			std::vector<Hierarchy> hierarchy_;

			// Mesh component: Which node corresponds to which node
			std::unordered_map<uint32_t, uint32_t> meshes_;

			// Material component: Which material belongs to which node
			std::unordered_map<uint32_t, uint32_t> materialForNode_;

			// Node name component: Which name is assigned to the node
			std::unordered_map<uint32_t, uint32_t> nameForNode_;

			// List of scene node names
			std::vector<std::string> names_;

			// Debug list of material names
			std::vector<std::string> materialNames_;
		};

		struct PACKED_STRUCT MaterialDescription final
		{
			gpuvec4 emissiveColor_ = { 0.0f, 0.0f, 0.0f, 0.0f};
			gpuvec4 albedoColor_   = { 1.0f, 1.0f, 1.0f, 1.0f };
			// UV anisotropic roughness (isotropic lighting models use only the first value). ZW values are ignored
			gpuvec4 roughness_     = { 1.0f, 1.0f, 0.0f, 0.0f };
			float transparencyFactor_ = 1.0f;
			float alphaTest_          = 0.0f;
			float metallicFactor_     = 0.0f;
			uint32_t flags_ = sMaterialFlags_CastShadow | sMaterialFlags_ReceiveShadow;
			// maps
			uint64_t ambientOcclusionMap_  = INVALID_TEXTURE;
			uint64_t emissiveMap_          = INVALID_TEXTURE;
			uint64_t albedoMap_            = INVALID_TEXTURE;
			/// Occlusion (R), Roughness (G), Metallic (B) https://github.com/KhronosGroup/glTF/issues/857
			uint64_t metallicRoughnessMap_ = INVALID_TEXTURE;
			uint64_t normalMap_            = INVALID_TEXTURE;
			uint64_t opacityMap_           = INVALID_TEXTURE;
		};

VKSceneData 主要是将场景数据加载进来，便于 MultiRenderer 渲染，其中主要包括几何数据、材质数据、变换数据这三部分，具体分析可见 [引擎架构之场景数据](engine/sceneData.md)
下面详细看一下 VKSceneData 的构造函数，是如何将这些数据加载进来的

		VKSceneData::VKSceneData(VulkanRenderContext& ctx,
			const char* meshFile,
			const char* sceneFile,
			const char* materialFile,
			VulkanTexture envMap,
			VulkanTexture irradianceMap,
			bool asyncLoad)
		: ctx(ctx)
		, envMapIrradiance_(irradianceMap)
		, envMap_(envMap)
		{
			brdfLUT_ = ctx.resources.loadKTX("data/brdfLUT.ktx");

			// 1. 加载材质数据
			loadMaterials(materialFile, materials_, textureFiles_);

			std::vector<VulkanTexture> textures;
			for (const auto& f: textureFiles_) {
				auto t = asyncLoad ? ctx.resources.addSolidRGBATexture() : ctx.resources.loadTexture2D(f.c_str());
				textures.push_back(t);
		#if 0
				if (t.image.image != nullptr)
					setVkImageName(ctx.vkDev, t.image.image, f.c_str());
		#endif
			}

			if (asyncLoad)
			{
				loadedFiles_.reserve(textureFiles_.size());

				taskflow_.for_each_index(0u, (uint32_t)textureFiles_.size(), 1u, [this](int idx)
					{
						int w, h;
						const uint8_t* img = stbi_load(this->textureFiles_[idx].c_str(), &w, &h, nullptr, STBI_rgb_alpha);
						if (!img)
							img = genDefaultCheckerboardImage(&w, &h);
						std::lock_guard lock(loadedFilesMutex_);
						loadedFiles_.emplace_back(LoadedImageData { idx, w, h, img });
					}
				);

				executor_.run(taskflow_);
			}

			allMaterialTextures = fsTextureArrayAttachment(textures);

			const uint32_t materialsSize = static_cast<uint32_t>(sizeof(MaterialDescription) * materials_.size());
			material_ = ctx.resources.addStorageBuffer(materialsSize);
			uploadBufferData(ctx.vkDev, material_.memory, 0, materials_.data(), materialsSize);

			// 2. 加载几何数据
			loadMeshes(meshFile);

			// 3. 加载场景数据
			loadScene(sceneFile);
		}
下面分别看下材质、几何、场景数据的加载流程：
1. 加载材质数据

		// 1. 加载材质数据
		loadMaterials(materialFile, materials_, textureFiles_);
		std::vector<VulkanTexture> textures;
		for (const auto& f: textureFiles_) {
			auto t = asyncLoad ? ctx.resources.addSolidRGBATexture() : ctx.resources.loadTexture2D(f.c_str());
			textures.push_back(t);
		#if 0
			if (t.image.image != nullptr)
				setVkImageName(ctx.vkDev, t.image.image, f.c_str());
		#endif
		}
		if (asyncLoad)
		{
			loadedFiles_.reserve(textureFiles_.size());
			taskflow_.for_each_index(0u, (uint32_t)textureFiles_.size(), 1u, [this](int idx)
				{
					int w, h;
					const uint8_t* img = stbi_load(this->textureFiles_[idx].c_str(), &w, &h, nullptr, STBI_rgb_alpha);
					if (!img)
						img = genDefaultCheckerboardImage(&w, &h);
					std::lock_guard lock(loadedFilesMutex_);
					loadedFiles_.emplace_back(LoadedImageData { idx, w, h, img });
				}
			);
			executor_.run(taskflow_);
		}
		allMaterialTextures = fsTextureArrayAttachment(textures);
		const uint32_t materialsSize = static_cast<uint32_t>(sizeof(MaterialDescription) * materials_.size());
		material_ = ctx.resources.addStorageBuffer(materialsSize);
		uploadBufferData(ctx.vkDev, material_.memory, 0, materials_.data(), materialsSize);
	其中，`loadMaterials(materialFile, materials_, textureFiles_);` 加载 MaterialDescription 和 纹理文件路径，代码如下：

		void loadMaterials(const char* fileName, std::vector<MaterialDescription>& materials, std::vector<std::string>& files)
		{
			FILE* f = fopen(fileName, "rb");
			if (!f) {
				printf("Cannot load file %s\nPlease run SceneConverter tool from Chapter7\n", fileName);
				exit(255);
			}

			uint32_t sz;
			fread(&sz, 1, sizeof(uint32_t), f);
			materials.resize(sz);
			fread(materials.data(), sizeof(MaterialDescription), materials.size(), f);
			loadStringList(f, files);
			fclose(f);
		}
	加载完后，对于 Material 的处理分为两部分，一部分是加载纹理图片数据，一部分是将 MaterialDescription 的数据上传到 storage buffer 中：

		std::vector<VulkanTexture> textures;
		for (const auto& f: textureFiles_) {
			auto t = asyncLoad ? ctx.resources.addSolidRGBATexture() : ctx.resources.loadTexture2D(f.c_str());
			textures.push_back(t);
		#if 0
			if (t.image.image != nullptr)
				setVkImageName(ctx.vkDev, t.image.image, f.c_str());
		#endif
		}

		if (asyncLoad)
		{
			loadedFiles_.reserve(textureFiles_.size());

			taskflow_.for_each_index(0u, (uint32_t)textureFiles_.size(), 1u, [this](int idx)
				{
					int w, h;
					const uint8_t* img = stbi_load(this->textureFiles_[idx].c_str(), &w, &h, nullptr, STBI_rgb_alpha);
					if (!img)
						img = genDefaultCheckerboardImage(&w, &h);
					std::lock_guard lock(loadedFilesMutex_);
					loadedFiles_.emplace_back(LoadedImageData { idx, w, h, img });
				}
			);

			executor_.run(taskflow_);
		}

		allMaterialTextures = fsTextureArrayAttachment(textures);
	加载纹理可以选择同步或者异步的方式，同步的情况下，直接就在主线程加载 texture，异步的话会先构建一个占位的 texture，然后异步加载数据，之后再更新 texture，代码如下：	

		bool MultiRenderer::checkLoadedTextures()
		{
			VKSceneData::LoadedImageData data;

			{
				std::lock_guard lock(sceneData_.loadedFilesMutex_);

				if (sceneData_.loadedFiles_.empty())
					return false;

				data = sceneData_.loadedFiles_.back();

				sceneData_.loadedFiles_.pop_back();
			}

			this->updateTexture(data.index_, ctx_.resources.addRGBATexture(data.w_, data.h_, const_cast<uint8_t*>(data.img_)));

			stbi_image_free((void*)data.img_);

			return true;
		}
	除了纹理外，另一部分需要处理的是材质的描述信息：

		allMaterialTextures = fsTextureArrayAttachment(textures);

		const uint32_t materialsSize = static_cast<uint32_t>(sizeof(MaterialDescription) * materials_.size());
		material_ = ctx.resources.addStorageBuffer(materialsSize);
		uploadBufferData(ctx.vkDev, material_.memory, 0, materials_.data(), materialsSize);
	其中，关于 `fsTextureArrayAttachment(textures)`，是将 texture 包装成统一的 DescriptorSetAttachment 的形式，具体参见 [引擎架构之资源管理](engine/resource.md)；
2. 加载几何数据

		void VKSceneData::loadMeshes(const char* meshFile)
		{
			// 1. 加载 Mesh 文件信息到 meshData_
			MeshFileHeader header = loadMeshData(meshFile, meshData_);

			// 2. 获取索引、顶点缓存的大小，并做对齐处理
			const uint32_t indexBufferSize = header.indexDataSize;
			uint32_t vertexBufferSize = header.vertexDataSize;

			const uint32_t offsetAlignment = getVulkanBufferAlignment(ctx.vkDev);
			if ((vertexBufferSize & (offsetAlignment - 1)) != 0)
			{
				const size_t numFloats = (offsetAlignment - (vertexBufferSize & (offsetAlignment - 1))) / sizeof(float);
				for (size_t i = 0; i != numFloats; i++)
					meshData_.vertexData_.push_back(0);
				vertexBufferSize = (vertexBufferSize + offsetAlignment) & ~(offsetAlignment - 1);
			}

			// 3. 上传到 storage buffer
			VulkanBuffer storage = ctx.resources.addStorageBuffer(vertexBufferSize + indexBufferSize);
			uploadBufferData(ctx.vkDev, storage.memory, 0, meshData_.vertexData_.data(), vertexBufferSize);
			uploadBufferData(ctx.vkDev, storage.memory, vertexBufferSize, meshData_.indexData_.data(), indexBufferSize);

			vertexBuffer_ = BufferAttachment { .dInfo = { .type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, .shaderStageFlags = VK_SHADER_STAGE_VERTEX_BIT }, .buffer = storage, .offset = 0, .size = vertexBufferSize };
			indexBuffer_  = BufferAttachment { .dInfo = { .type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, .shaderStageFlags = VK_SHADER_STAGE_VERTEX_BIT }, .buffer = storage, .offset = vertexBufferSize, .size = indexBufferSize };
		}
	这里主要是将 index 和 vertex 缓冲上传上去，Mesh 信息，在 loadScene 里面给 DrawData 用。

3. 加载场景数据

		void VKSceneData::loadScene(const char* sceneFile)
		{
			// 1. 加载场景数据
			::loadScene(sceneFile, scene_);

			// 2. 通过 DrawData 组织每个 IndirectDraw 所需要的数据
			// prepare draw data buffer
			for (const auto& c : scene_.meshes_) // 这里的 scene_.meshes_ 是 k(nodeIndex) : v(meshIndex) 的字典
			{
				auto material = scene_.materialForNode_.find(c.first); // 这里的 scene_.materialForNode_ 是 k(nodeIndex) : v(materialIndex) 的字典
				if (material == scene_.materialForNode_.end())
					continue;

				shapes_.push_back(
					DrawData{
						.meshIndex = c.second,
						.materialIndex = material->second,
						.LOD = 0,
						.indexOffset = meshData_.meshes_[c.second].indexOffset,
						.vertexOffset = meshData_.meshes_[c.second].vertexOffset,
						.transformIndex = c.first
					});
			}

			// 3. 计算 global transform，并且按照 mesh 的顺序重新组织数组上传
			shapeTransforms_.resize(shapes_.size());
			transforms_ = ctx.resources.addStorageBuffer(shapes_.size() * sizeof(glm::mat4));

			recalculateAllTransforms();
			uploadGlobalTransforms();
		}

		void VKSceneData::recalculateAllTransforms()
		{
			// force recalculation of global transformations
			markAsChanged(scene_, 0);
			recalculateGlobalTransforms(scene_);
		}

		void VKSceneData::uploadGlobalTransforms()
		{
			convertGlobalToShapeTransforms();
		 	uploadBufferData(ctx.vkDev, transforms_.memory, 0, shapeTransforms_.data(), transforms_.size);
		}

		void VKSceneData::convertGlobalToShapeTransforms()
		{
			// fill the shapeTransforms_ array from globalTransforms_
			size_t i = 0;
			for (const auto& c : shapes_)
				shapeTransforms_[i++] = scene_.globalTransform_[c.transformIndex];
		}
	场景数据主要是处理 DrawData 和 transform 两大部分。

### 构造函数

		MultiRenderer::MultiRenderer(
			VulkanRenderContext& ctx,
			VKSceneData& sceneData,
			const char* vertShaderFile,
			const char* fragShaderFile,
			const std::vector<VulkanTexture>& outputs,
			RenderPass screenRenderPass,
			const std::vector<BufferAttachment>& auxBuffers,
			const std::vector<TextureAttachment>& auxTextures)
		: Renderer(ctx)
		, sceneData_(sceneData)
		{
			// 1. 初始化 RenderPass
			const PipelineInfo pInfo = initRenderPass(PipelineInfo {}, outputs, screenRenderPass, ctx.screenRenderPass);

			// 2. 计算 indirectBuffer 大小
			const uint32_t indirectDataSize = (uint32_t)sceneData_.shapes_.size() * sizeof(VkDrawIndirectCommand);

			// 3. 每个 swapChain 中的 Image，对应一套缓存数据
			const size_t imgCount = ctx.vkDev.swapchainImages.size();
			uniforms_.resize(imgCount);
			shape_.resize(imgCount);
			indirect_.resize(imgCount);

			descriptorSets_.resize(imgCount);

			// 4. DrawData 对应缓存大小
			const uint32_t shapesSize = (uint32_t)sceneData_.shapes_.size() * sizeof(DrawData);
			// 5. MVP 数据对应缓存大小
			const uint32_t uniformBufferSize = sizeof(ubo_);

			// 6. 收集 envMap_、envMapIrradiance_ 和 brdfLUT_，如果有额外贴图 auxTextures，也可以传入
			std::vector<TextureAttachment> textureAttachments;
			if (sceneData_.envMap_.width)
				textureAttachments.push_back(fsTextureAttachment(sceneData_.envMap_));
			if (sceneData_.envMapIrradiance_.width)
				textureAttachments.push_back(fsTextureAttachment(sceneData_.envMapIrradiance_));
			if (sceneData_.brdfLUT_.width)
				textureAttachments.push_back(fsTextureAttachment(sceneData_.brdfLUT_));

			for (const auto& t: auxTextures)
				textureAttachments.push_back(t);

			// 7. 保存 DescriptorSet 中所用资源的信息，其中 buffers 保存 storage buffer，textures 保存纹理对象，textureArrays 保存纹理数组
			DescriptorSetInfo dsInfo = {
				.buffers = {
					uniformBufferAttachment(VulkanBuffer {},         0, uniformBufferSize, VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT),
					sceneData_.vertexBuffer_,
					sceneData_.indexBuffer_,
					storageBufferAttachment(VulkanBuffer {},         0, shapesSize, VK_SHADER_STAGE_VERTEX_BIT),
					storageBufferAttachment(sceneData_.material_,    0, (uint32_t)sceneData_.material_.size, VK_SHADER_STAGE_FRAGMENT_BIT),
					storageBufferAttachment(sceneData_.transforms_,  0, (uint32_t)sceneData_.transforms_.size, VK_SHADER_STAGE_VERTEX_BIT),
				},
				.textures = textureAttachments,
				.textureArrays = { sceneData_.allMaterialTextures }
			};

			// 8. 如果有额外的 buffer，也保存起来
			for (const auto& b: auxBuffers)
				dsInfo.buffers.push_back(b);

			// 9. 接下来根据 dsInfo，创建 descriptorSets_
			descriptorSetLayout_ = ctx.resources.addDescriptorSetLayout(dsInfo);
			descriptorPool_ = ctx.resources.addDescriptorPool(dsInfo, (uint32_t)imgCount);

			for (size_t i = 0; i != imgCount; i++)
			{
				uniforms_[i] = ctx.resources.addUniformBuffer(uniformBufferSize);
				indirect_[i] = ctx.resources.addIndirectBuffer(indirectDataSize);
				updateIndirectBuffers(i);

				shape_[i] = ctx.resources.addStorageBuffer(shapesSize);
				uploadBufferData(ctx.vkDev, shape_[i].memory, 0, sceneData_.shapes_.data(), shapesSize);

				dsInfo.buffers[0].buffer = uniforms_[i];
				dsInfo.buffers[3].buffer = shape_[i];

				descriptorSets_[i] = ctx.resources.addDescriptorSet(descriptorPool_, descriptorSetLayout_);
				ctx.resources.updateDescriptorSet(descriptorSets_[i], dsInfo);
			}

			// 10. 初始化 pipeline
			initPipeline({ vertShaderFile, fragShaderFile }, pInfo);
		}
构造函数的主要功能分为三部分：  
1. 初始化 DescriptorSet，其中详细分析参照 [引擎架构之资源管理](engine/resource.md)；
2. 初始化 RenderPass；
3. 初始化 Pipeline；

其中，创建 DescriptorSet 过程中涉及到了 IndirectCmdBuffer 的创建，下面会分析一下：

		void MultiRenderer::updateIndirectBuffers(size_t currentImage, bool* visibility)
		{
			VkDrawIndirectCommand* data = nullptr;
			vkMapMemory(ctx_.vkDev.device, indirect_[currentImage].memory, 0, sizeof(VkDrawIndirectCommand), 0, (void**)&data);

			const uint32_t size = (uint32_t)sceneData_.shapes_.size();

			for (uint32_t i = 0; i != size; i++)
			{
				const uint32_t j = sceneData_.shapes_[i].meshIndex;

				const uint32_t lod = sceneData_.shapes_[i].LOD;
				data[i] = {
					.vertexCount = sceneData_.meshData_.meshes_[j].getLODIndicesCount(lod),
					.instanceCount = visibility ? (visibility[i] ? 1u : 0u) : 1u,
					.firstVertex = 0,
					.firstInstance = i
				};
			}
			vkUnmapMemory(ctx_.vkDev.device, indirect_[currentImage].memory);
		}
主要是根据 DrawData 填充 IndirectDrawCmd 数据，其中映射 Map 的时候，注意要使用指针的指针，这样才能正确填充数据。

### fillCommandBuffer、updateBuffers

		void MultiRenderer::fillCommandBuffer(VkCommandBuffer commandBuffer, size_t currentImage, VkFramebuffer fb, VkRenderPass rp)
		{
			beginRenderPass((rp != VK_NULL_HANDLE) ? rp : renderPass_.handle, (fb != VK_NULL_HANDLE) ? fb : framebuffer_, commandBuffer, currentImage);

			/* For CountKHR (Vulkan 1.1) we may use indirect rendering with GPU-based object counter */
			/// vkCmdDrawIndirectCountKHR(commandBuffer, indirectBuffers_[currentImage], 0, countBuffers_[currentImage], 0, shapes.size(), sizeof(VkDrawIndirectCommand));
			/* For Vulkan 1.0 vkCmdDrawIndirect is enough */
			vkCmdDrawIndirect(commandBuffer, indirect_[currentImage].buffer, 0, (uint32_t)sceneData_.shapes_.size(), sizeof(VkDrawIndirectCommand));

			vkCmdEndRenderPass(commandBuffer);
		}

		void MultiRenderer::updateBuffers(size_t imageIndex)
		{
			updateUniformBuffer((uint32_t)imageIndex, 0, sizeof(ubo_), &ubo_);
		}
		