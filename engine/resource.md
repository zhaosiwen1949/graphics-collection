# 引擎架构之资源管理
资源管理类似 OpenGL 中的上下文，需要提供接口用来申请 Buffer、Image、RenderPass、FrameBuffer、Pipeline、DescriptorSet 等资源，并且在应用结束时，负责统一清理这些资源。

## DescriptorSet 管理
资源管理中，最重要的一项是对于 DescriptorSet 的管理，这里我们看下如何对 DescriptorSet 资源进行统一管理：
1. 抽象出 `DescriptorSetInfo` 结构作为对于 DescriptorSet 的核心描述；
2. DescriptorSetLayout、DescriptorPool 都会解析该结构，然后进行生成；
3. 对于每一帧的 DescriptorSet[] 数组，根据 DescriptorSetLayout、DescriptorPool 创建 DescriptorSet，并根据 DescriptorSetInfo 更新 DescriptorSet；

下面我们先来看下 DescriptorSetInfo 的基本结构与使用方式，再一一拆解其中使用到的函数：
### DescriptorSetInfo 的基本结构
```
// 1、保存 DescriptorSet 的信息
struct DescriptorSetInfo
{
	std::vector<BufferAttachment>       buffers;
	std::vector<TextureAttachment>      textures;
	std::vector<TextureArrayAttachment> textureArrays;
};

// 2、关联 buffer 对象和 DescriptorInfo 描述对象
struct BufferAttachment
{
	DescriptorInfo dInfo;

	VulkanBuffer   buffer;
	uint32_t       offset;
	uint32_t       size;
};

struct TextureAttachment
{
	DescriptorInfo dInfo;

	VulkanTexture  texture;
};

struct TextureArrayAttachment
{
	DescriptorInfo dInfo;

	std::vector<VulkanTexture> textures;
};

// 3、描述 buffer 对象的类型和被使用的阶段
struct DescriptorInfo
{
	VkDescriptorType type;
	VkShaderStageFlags shaderStageFlags;
};

// 4、描述不同的 buffer 对象，主要是 handle 和 memory，以及辅助信息
struct VulkanBuffer
{
	VkBuffer       buffer;
	VkDeviceSize   size;
	VkDeviceMemory memory;

	/* Permanent mapping to CPU address space (see VulkanResources::addBuffer) */
	void*          ptr;
};

struct VulkanImage final
{
	VkImage image = nullptr;
	VkDeviceMemory imageMemory = nullptr;
	VkImageView imageView = nullptr;
};

// Aggregate structure for passing around the texture data
struct VulkanTexture final
{
	uint32_t width;
	uint32_t height;
	uint32_t depth;
	VkFormat format;

	VulkanImage image;
	VkSampler sampler;

	// Offscreen buffers require VK_IMAGE_LAYOUT_GENERAL && static textures have VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
	VkImageLayout desiredLayout;
};

```
这里的核心是 `XXXAttachment`，用来将对于缓存对象的描述信息和真实的 buffer、image 对象联系起来。

### DescriptorSetInfo 的使用方法
`DescriptorSetInfo` 基本的使用规则有两条：
1. 首先根据 `DescriptorSetInfo` 初始化 `DescriptorSetLayout` 和 `DescriptorPool`，然后依次填充每个 `swapChainImage` 所需要的 `DescriptorSets`；
2. `DescriptorSetInfo` 中可以一开始不填写具体的 `VulkanBuffer/VulkanTexture`，只需要在 `addDescriptorSet()` 之前填充好就行；

```
// 1. 配置 DescriptorSetInfo 描述信息
DescriptorSetInfo dsInfo = {
	.buffers = {
		// 1-1. uniform buffer 需要根据不同的 swapChainImage 来确定，故此时空缺
		uniformBufferAttachment(VulkanBuffer {}, 0, uniformBufferSize, VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT),
		sceneData_.vertexBuffer_,
		sceneData_.indexBuffer_,
		// 1-2. drawData buffer 同上
		storageBufferAttachment(VulkanBuffer {}, 0, shapesSize, VK_SHADER_STAGE_VERTEX_BIT),
		storageBufferAttachment(sceneData_.material_, 0, (uint32_t)sceneData_.material_.size, VK_SHADER_STAGE_FRAGMENT_BIT),
		storageBufferAttachment(sceneData_.transforms_, 0, (uint32_t)sceneData_.transforms_.size, VK_SHADER_STAGE_VERTEX_BIT),
	},
	.textures = textureAttachments,
	.textureArrays = { sceneData_.allMaterialTextures }
};

// 2. 创建 DescriptorSetLayout 和 DescriptorPool
descriptorSetLayout_ = ctx.resources.addDescriptorSetLayout(dsInfo);
descriptorPool_ = ctx.resources.addDescriptorPool(dsInfo, (uint32_t)imgCount);

// 3. 创建并更新 DescriptorSet
for (size_t i = 0; i != imgCount; i++)
{
	// 3-1. 创建 uniform 和 drawData buffer
	uniforms_[i] = ctx.resources.addUniformBuffer(uniformBufferSize);
	shape_[i] = ctx.resources.addStorageBuffer(shapesSize);
	uploadBufferData(ctx.vkDev, shape_[i].memory, 0, sceneData_.shapes_.data(), shapesSize);

	// 3-2. 关联 DescriptorSetInfo
	dsInfo.buffers[0].buffer = uniforms_[i];
	dsInfo.buffers[3].buffer = shape_[i];

	// 3-3. 创建并更新 DescriptorSet
	descriptorSets_[i] = ctx.resources.addDescriptorSet(descriptorPool_, descriptorSetLayout_);
	ctx.resources.updateDescriptorSet(descriptorSets_[i], dsInfo);
}

```

### DescriptorSetXXX 相关的函数
和 DescriptorSet 创建相关的函数总共有四个，下面一一分析：
1. addDescriptorSetLayout，根据 DescriptorSetInfo，创建描述符布局信息；

		VkDescriptorSetLayout VulkanResources::addDescriptorSetLayout(const DescriptorSetInfo& dsInfo)
		{
			VkDescriptorSetLayout descriptorSetLayout;

			uint32_t bindingIdx = 0;

			std::vector<VkDescriptorSetLayoutBinding> bindings;
			
			// 1. 创建 bindings，bindingIdx 从 0 开始递增
			for (const auto& b: dsInfo.buffers) {
				bindings.push_back(descriptorSetLayoutBinding(bindingIdx++, b.dInfo.type, b.dInfo.shaderStageFlags));
			}

			for (const auto& i: dsInfo.textures) {
				bindings.push_back(descriptorSetLayoutBinding(bindingIdx++, i.dInfo.type /*VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER*/, i.dInfo.shaderStageFlags));
			}

			for (const auto& t: dsInfo.textureArrays) {
				bindings.push_back(descriptorSetLayoutBinding(bindingIdx++, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, t.dInfo.shaderStageFlags, static_cast<uint32_t>(t.textures.size())));
			}

			// 2. 根据 bindings 创建 DescriptorSetLayout
			const VkDescriptorSetLayoutCreateInfo layoutInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.bindingCount = static_cast<uint32_t>(bindings.size()),
				.pBindings = bindings.size() > 0 ? bindings.data() : nullptr
			};

			if (vkCreateDescriptorSetLayout(vkDev.device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS)
			{
				printf("Failed to create descriptor set layout\n");
				exit(EXIT_FAILURE);
			}

			allDSLayouts.push_back(descriptorSetLayout);
			return descriptorSetLayout;
		}

		inline VkDescriptorSetLayoutBinding descriptorSetLayoutBinding(uint32_t binding, VkDescriptorType descriptorType, VkShaderStageFlags stageFlags, uint32_t descriptorCount = 1)
		{
			return VkDescriptorSetLayoutBinding{
				.binding = binding,
				.descriptorType = descriptorType,
				.descriptorCount = descriptorCount,
				.stageFlags = stageFlags,
				.pImmutableSamplers = nullptr
			};
		}

2. addDescriptorPool，根据 DescriptorSetInfo 和 swapChainImageCount 来创建描述符集分配池，其中最关键的属性是 DescriptorSet 的数量和不同类型的 Descriptor 的数量；

		VkDescriptorPool VulkanResources::addDescriptorPool(const DescriptorSetInfo& dsInfo, uint32_t dSetCount)
		{
			// 1. 统计 uniform、storage、sampler 的数量
			uint32_t uniformBufferCount = 0;
			uint32_t storageBufferCount = 0;
			uint32_t samplerCount = static_cast<uint32_t>(dsInfo.textures.size());

			for(const auto& ta : dsInfo.textureArrays)
				samplerCount += static_cast<uint32_t>(ta.textures.size());

			for(const auto& b: dsInfo.buffers)
			{
				if (b.dInfo.type == VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER)
					uniformBufferCount++;
				if (b.dInfo.type == VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
					storageBufferCount++;
			}

			std::vector<VkDescriptorPoolSize> poolSizes;

			/* printf("Allocating pool[%d | %d | %d]\n", (int)uniformBufferCount, (int)storageBufferCount, (int)samplerCount); */

			// 2. 配置 DescriptorPool 所需的数量信息
			if (uniformBufferCount)
				poolSizes.push_back(VkDescriptorPoolSize{ .type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, .descriptorCount = dSetCount * uniformBufferCount });

			if (storageBufferCount)
				poolSizes.push_back(VkDescriptorPoolSize{ .type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, .descriptorCount = dSetCount * storageBufferCount });

			if (samplerCount)
				poolSizes.push_back(VkDescriptorPoolSize{ .type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, .descriptorCount = dSetCount * samplerCount });

			// 3. 创建 DescriptorPool
			const VkDescriptorPoolCreateInfo poolInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO,
				.pNext = nullptr,
				.flags = 0,
				.maxSets = static_cast<uint32_t>(dSetCount),
				.poolSizeCount = static_cast<uint32_t>(poolSizes.size()),
				.pPoolSizes = poolSizes.empty() ? nullptr : poolSizes.data()
			};

			VkDescriptorPool descriptorPool = VK_NULL_HANDLE;

			if (vkCreateDescriptorPool(vkDev.device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS)
			{
				printf("Cannot allocate descriptor pool\n");
				exit(EXIT_FAILURE);
			}

			allDPools.push_back(descriptorPool);
			return descriptorPool;
		}

3. addDescriptorSet，根据 descriptorPool_ 描述符集分配池和 descriptorSetLayout_ 描述符布局信息，创建 DescriptorSet 描述符集；

		VkDescriptorSet VulkanResources::addDescriptorSet(VkDescriptorPool descriptorPool, VkDescriptorSetLayout dsLayout)
		{
			VkDescriptorSet descriptorSet;

			const VkDescriptorSetAllocateInfo allocInfo = {
				.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
				.pNext = nullptr,
				.descriptorPool = descriptorPool,
				.descriptorSetCount = 1,
				.pSetLayouts = &dsLayout
			};

			if (vkAllocateDescriptorSets(vkDev.device, &allocInfo, &descriptorSet) != VK_SUCCESS)
			{
				printf("Cannot allocate descriptor set\n");
				exit(EXIT_FAILURE);
			}

			return descriptorSet;
		}
		
4. updateDescriptorSet，根据 DescriptorSetInfo，更新 DescriptorSet 描述符集；

		void VulkanResources::updateDescriptorSet(VkDescriptorSet ds, const DescriptorSetInfo& dsInfo)
		{
			uint32_t bindingIdx = 0;
			std::vector<VkWriteDescriptorSet> descriptorWrites;

			// 1. 分类型存储 Descriptor 描述符信息
			std::vector<VkDescriptorBufferInfo> bufferDescriptors(dsInfo.buffers.size());
			std::vector<VkDescriptorImageInfo>  imageDescriptors(dsInfo.textures.size());
			std::vector<VkDescriptorImageInfo>  imageArrayDescriptors;

			// 2. 保存 buffer 信息
			for (size_t i = 0 ; i < dsInfo.buffers.size() ; i++)
			{
				BufferAttachment b = dsInfo.buffers[i];

				bufferDescriptors[i] = VkDescriptorBufferInfo {
					.buffer = b.buffer.buffer,
					.offset = b.offset,
					.range  = (b.size > 0) ? b.size : VK_WHOLE_SIZE
				};

				descriptorWrites.push_back(bufferWriteDescriptorSet(ds, &bufferDescriptors[i], bindingIdx++, b.dInfo.type));
			}

			// 3. 保存 texture 信息
			for(size_t i = 0 ; i < dsInfo.textures.size() ; i++)
			{
				VulkanTexture t = dsInfo.textures[i].texture;

				imageDescriptors[i] = VkDescriptorImageInfo {
					.sampler = t.sampler,
					.imageView = t.image.imageView,
					.imageLayout = /* t.texture.layout */ VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
				};

				descriptorWrites.push_back(imageWriteDescriptorSet(ds, &imageDescriptors[i], bindingIdx++));
			}

			// 4. 保存 textureArray 信息，注意，这里的 taOffsets 是为了后面和 imageArrayDescriptors.data() 指针配合，用来偏移起始位置
			uint32_t taOffset = 0;
			std::vector<uint32_t> taOffsets(dsInfo.textureArrays.size());
			for (size_t ta = 0 ; ta < dsInfo.textureArrays.size() ; ta++)
			{
				taOffsets[ta] = taOffset;

				for (size_t j = 0 ; j < dsInfo.textureArrays[ta].textures.size() ; j++)
				{
					VulkanTexture t = dsInfo.textureArrays[ta].textures[j];

					VkDescriptorImageInfo imageInfo = {
						.sampler = t.sampler,
						.imageView = t.image.imageView,
						.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
					};

					imageArrayDescriptors.push_back(imageInfo); // item 'taOffsets[ta] + j'
				}

				taOffset += static_cast<uint32_t>(dsInfo.textureArrays[ta].textures.size());
			}

			for (size_t ta = 0 ; ta < dsInfo.textureArrays.size() ; ta++)
			{
				VkWriteDescriptorSet writeSet = {
					.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
					.dstSet = ds,
					.dstBinding = bindingIdx++,
					.dstArrayElement = 0,
					.descriptorCount = static_cast<uint32_t>(dsInfo.textureArrays[ta].textures.size()),
					.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
					.pImageInfo = imageArrayDescriptors.data() + taOffsets[ta]
				};

				descriptorWrites.push_back(writeSet);
			}
			
			// 4. 更新绑定
			vkUpdateDescriptorSets(vkDev.device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
		}

		inline VkWriteDescriptorSet bufferWriteDescriptorSet(VkDescriptorSet ds, const VkDescriptorBufferInfo* bi, uint32_t bindIdx, VkDescriptorType dType)
		{
			return VkWriteDescriptorSet { VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET, nullptr,
				ds, bindIdx, 0, 1, dType, nullptr, bi, nullptr
			};
		}

		inline VkWriteDescriptorSet imageWriteDescriptorSet(VkDescriptorSet ds, const VkDescriptorImageInfo* ii, uint32_t bindIdx) 
		{
			return VkWriteDescriptorSet { VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET, nullptr,
				ds, bindIdx, 0, 1, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
				ii, nullptr, nullptr
			};
		}

		typedef struct VkWriteDescriptorSet {
		    VkStructureType                  sType;
		    const void*                      pNext;
		    VkDescriptorSet                  dstSet;
		    uint32_t                         dstBinding;
		    uint32_t                         dstArrayElement;
		    uint32_t                         descriptorCount;
		    VkDescriptorType                 descriptorType;
		    const VkDescriptorImageInfo*     pImageInfo;
		    const VkDescriptorBufferInfo*    pBufferInfo;
		    const VkBufferView*              pTexelBufferView;
		} VkWriteDescriptorSet;
