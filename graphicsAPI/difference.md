# OpenGL 与 Vulkan 的区别

1. Buffer 更新
    * OpenGL 3.x/2.x

            float vertices[] = {
                -0.5f, -0.5f, 0.0f,
                 0.5f, -0.5f, 0.0f,
                 0.0f,  0.5f, 0.0f
            };

            unsigned int VBO;
            glGenBuffers(1, &VBO);
            glBindBuffer(GL_ARRAY_BUFFER, VBO);
            glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

        对于经典版本的 OpenGL，只有顶点数组和索引数组是通过 Buffer 进行更新， Uniform 变量通过 glUniformXX 系列函数更新，其中对于 Buffer 的更新需要首先绑定 GL_ARRAY_BUFFER 目标，然后再对于 GL_ARRAY_BUFFER 目标进行更新；
    * OpenGL 4.6

            # 创建和 Shader 中 Uniform 变量一致的类型
            struct PerFrameData
            {
            	mat4 mvp;
            	int isWireframe;
            };
            const GLsizeiptr kBufferSize = sizeof(PerFrameData);

            # 创建 NamedBuffer，并将其某一段数据绑定到 Uniform 变量
            unsigned int perFrameDataBuffer;
	        glCreateBuffers(1, &perFrameDataBuffer);
	        glNamedBufferStorage(perFrameDataBuffer, kBufferSize, nullptr, GL_DYNAMIC_STORAGE_BIT);
	        glBindBufferRange(GL_UNIFORM_BUFFER, 0, perFrameDataBuffer, 0, kBufferSize);

            # 更新 NamedBuffer
            const mat4 m = glm::rotate(glm::translate(mat4(1.0f), vec3(0.0f, 0.0f, -3.5f)), (float)glfwGetTime(), vec3(1.0f, 1.0f, 1.0f));
		    const mat4 p = glm::perspective(45.0f, ratio, 0.1f, 1000.0f);

		    PerFrameData perFrameData = { .mvp = p * m, .isWireframe = false };

		    glNamedBufferSubData(perFrameDataBuffer, 0, kBufferSize, &perFrameData);

        其中，Shader 代码如下：

            #version 460 core
            layout(std140, binding = 0) uniform PerFrameData
            {
            	uniform mat4 MVP;
            	uniform int isWireframe;
            };
            
        对于 OpenGL 4.6，通过 NamedBuffer 的方式，直接将 Buffer 和 Shader 中的 Uniform 常量对应起来，再通过直接更新 NamedBuffer，避免了 BindAndSub 模式，直接采用 Data State Access 的方式，降低了开销；
    * Vulkan

            VkBufferCreateInfo bufferInfo{};
            bufferInfo.sType =  VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
            bufferInfo.size = sizeof(vertices[0]) *     vertices.size();
            bufferInfo.usage =  VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
            bufferInfo.sharingMode =    VK_SHARING_MODE_EXCLUSIVE;

            if (vkCreateBuffer(device, &bufferInfo,nullptr, &vertexBuffer) != VK_SUCCESS) {
                throw std::runtime_error("failed to create vertex buffer!");
            }

            VkMemoryRequirements memRequirements;
            vkGetBufferMemoryRequirements(device,   vertexBuffer, &memRequirements);

            VkMemoryAllocateInfo allocInfo{};
            allocInfo.sType =   VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
            allocInfo.allocationSize = memRequirements. size;
            allocInfo.memoryTypeIndex = findMemoryType  (memRequirements.memoryTypeBits,  VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);

            if (vkAllocateMemory(device, &allocInfo,    nullptr, &vertexBufferMemory) != VK_SUCCESS) {
                throw std::runtime_error("failed to allocate vertex buffer memory!");
            }

            vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);

            void* data;
            vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
                memcpy(data, vertices.data(), (size_t)bufferInfo.size);
            vkUnmapMemory(device, vertexBufferMemory);

        Vulkan 中 Buffer 的使用方式，是把最底层的 memory 暴露给开发者，有开发者决定如何使用内存，主要步骤为：
        1. 创建 buffer，设定好信息
        2. 根据 buffer 选择 memory
        3. 根据 memory 的特性，复制数据  
    
    总结：可以看出，新一代的 API 的思想是脱离状态机的模式，直接控制内存，使用状态思维，直接描述这时候应该是什么状态，而不是状态机下的命令思维，一步一步修改状态。

2. 多线程支持
3. 内存资源布局的方式，Vulkan 具有更多更明确的控制权限，比如可以将 CPU 用不到的纹理，直接分配 device 的内存， OpenGL 只能在创建时使用提示，并不能明确指定；