# Vulkan API 梳理

> 参考文章：[Understanding Vulkan® Objects](https://gpuopen.com/learn/understanding-vulkan-objects/)

## 整体概览
![整体架构](/resources/images/Vulkan-Diagram.webp)

1. 创建 Instance，这一步需要配置支持的 Layer 和 Extension 扩展功能；
2. 创建 device 逻辑设备，这一步主要需要：物理设备 physical device、队列族 queue family、逻辑设备需要支持的扩展特性；
3. 创建 swapChain 交换链，注意 surface 需要通过窗口库创建，然后根据 surface 和 physical device 查询 swapChain 支持的 图像格式 和 呈现模式，选择合适的呈现模式；
4. swapChain 创建主要就是把 surface 和 graphic queue 连接起来，并且设置好 image 的格式、大小、数量、用途等，以及呈现模式、旋转模式、裁剪模式等 image 处理的内容；
5. 从 swapChain 中获取对应的 VkImage 对象，并创建x相应的 ImageView；
6. 创建 commandPool 和 queueFamily 关联；
7. 创建 commandBuffer，从 commandPool 分配；
8. 创建两个 信号量 semaphore，一个用来呈现完成后，通知图像 available，一个渲染完成后，通知可以用于呈现；
9. Buffer 和 Image：  
    1. createBuffer 创建 Buffer，并分配 memory 内存；
    2. createImage、createImageView 创建 Image 和 ImageView 对象，用于 color/depth/stencil attachment 和 texture 纹理对象；
    3. createSampler 创建纹理采样器；
    4. copyBuffer、copyImageToBuffer、copyBufferToImage，通过命令方式传输数据，其中对于 Image 对象的数据传输，需要先通过 transitionImageLayout，来转换图像布局 layout；
10. 创建深度缓冲 createDepthResources；
11. 创建纹理对象 createTextureImage；
12. 创建顶点缓存 createTexturedVertexBuffer，注意这里是用的 storage buffer 实现的；
13. 创建 Uniform 缓存 createUniformBuffers；
14. 创建描述符分配池 DescriptorPool，其中主要分配 DescriptorSet 的数量，以及 Uniform Buffer、Storage Buffer 和 Image Sampler 的数量，注意这里每一个 swapChain 上的 framebuffer 都都一个单独的DescriptorSet，并且会影响其他资源乘以对应的数量；
15. 创建 DescriptorSetLayout，描述资源布局；
16. 根据 DescriptorSetLayout，从 DescriptorPool 中，创建 DescriptorSet，最后通过 vkUpdateDescriptorSets，将真正的 buffer 数据更新到 DescriptorSet 中；
![描述符集](/resources/images/descriptor.png)
17. 创建 GraphicsPipeline 之前，先创建 PipelineLayout，PipelineLayout 主要描述 DescriptorSetLayout 和 PushContantRange，类似描述了函数的入参类型；
18. 创建 RenderPass，RenderPass 主要描述三部分内容，重点是如何输出，类似函数的返回值类型：
    1. color、depth、stencil attachment description，描述在 RenderPass 整个前后的 layout 分别是什么，注意这里并未和具体 framebuffer 绑定；
    2. subpass，这里 subpass 会引用上述 attachment，并且在这个 subpass 中使用的 layout 是什么；
    3. SubpassDependency，用来描述 subpass 之间异步操作的依赖，同时对于 memory layout 的变换也很重要；
19. 开始创建 GraphicsPipeline，设定以下部分：
    1. Vertext 顶点；
    2. Viewport 视角；
    3. Scissor 裁剪；
    4. Rasterizer 光栅化；
    5. MultiSampling 多重采样；
    6. Blend 混合；
    7. DepthAndStencil 深度测试、模板测试；
    8. DynamicState 动态状态；
    9. Tessellation 曲面细分；
    10. Shader；
    11. DescriptorSetLayout -> PipelineLayout；
    12. RenderPass；
20. 创建 FrameBuffer，主要是将 RenderPass 和 SwapChain 中的 ImageView，以及深度缓冲的 DepthImageView 关联起来，注意：FrameBuffer 类似 DescriptorSets，只是用来组合 image 的，RenderPass 类似 DescriptorSetLayout 只是描述使用的 framebuffer 的格式；

到此，资源的创建完成，下面是如何将上述资源串联起来，完成一次绘制：
1. vkAcquireNextImageKHR，获取下一帧可用资源；
2. vkResetCommandPool 重置命令缓冲池；
3. 更新 uniform 变量；
4. 开始录制 command buffer：
    1. vkBeginCommandBuffer，开始录制 command，设置 command buffer 的使用频率；
    2. vkCmdBeginRenderPass，开始 renderPass，设置 renderPass、framebuffer、绘图区域、颜色\深度的清除值；
    3. vkCmdBindPipeline，绑定 pipeline，设定采用 graphic 管线；
    4. vkCmdBindDescriptorSets，绑定 DescriptorSet，这里需要传入 PipelineLayout，并且支持绑定多个 DescriptorSets；
    5. vkCmdDraw，发起 DrawCall；
    6. vkCmdEndRenderPass；
    7. vkEndCommandBuffer；
5. vkQueueSubmit，提交命令，这里主要配置需要提交的 command、需要等待的信号量、绘制完成后命令发出的信号量；
6. vkQueuePresentKHR，呈现绘制的图像，主要配置 swapChain 以及对应的 imageIndex，以及需要等待完成的绘制信号量；