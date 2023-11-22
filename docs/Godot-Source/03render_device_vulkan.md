# Godot渲染流程
### 1.3 Vulkan Render Device 的创建

RenderingDeviceVulkan 继承于 RenderingDevice，而 RenderDevcice 则抽象了渲染了大部分的功能。我在翻译 The future of Render in Godot 视频的时候, 看到负责人有介绍下一代的 RD 渲染器将会有更高级别的抽象，Metal 和 DX12 都会基于RD 渲染器。但是粗略的查看代码之后，RenderingDevice 这个类其实还是没有抽象得很干净。

RenderDevice 负责的内容很多，它具有底层渲染的版本信息，是CPU，还是GPU。是 Vulkan 还是 OpenGL. Texture、FrameBuffer、Buffer 和 DrawList等的构建。我觉得还是有必要详细的解释 RenderDevice的代码

#### 1.3.1 RenderDevice
我把它单独写成一个文件，方便后面的引用，详细内容可以点击链接进入
[RenderDevice类介绍](04render_device.md)
#### 1.3.2 Vulkan Render Device
```cpp
rendering_device_vulkan->initialize(context_vulkan);
RendererCompositorRD::make_current();
```
先看下 initialize 函数的实现：
```cpp
void RenderingDeviceVulkan::initialize(VulkanContext *p_context, bool p_local_device) {
	// Get our device capabilities.
	{
		device_capabilities.version_major = p_context->get_vulkan_major();
		device_capabilities.version_minor = p_context->get_vulkan_minor();
	}

	context = p_context;
	device = p_context->get_device();
	if (p_local_device) {
		frame_count = 1;
		local_device = p_context->local_device_create();
		device = p_context->local_device_get_vk_device(local_device);
	} else {
		frame_count = p_context->get_swapchain_image_count() + 1; // Always need one extra to ensure it's unused at any time, without having to use a fence for this.
	}
	limits = p_context->get_device_limits();
	max_timestamp_query_elements = 256;

	{ // Initialize allocator.

		VmaAllocatorCreateInfo allocatorInfo;
		memset(&allocatorInfo, 0, sizeof(VmaAllocatorCreateInfo));
		allocatorInfo.physicalDevice = p_context->get_physical_device();
		allocatorInfo.device = device;
		allocatorInfo.instance = p_context->get_instance();
		vmaCreateAllocator(&allocatorInfo, &allocator);
	}

	frames.resize(frame_count);
	frame = 0;
	// Create setup and frame buffers.
	for (int i = 0; i < frame_count; i++) {
		frames[i].index = 0;

		{ // Create command pool, one per frame is recommended.
			VkCommandPoolCreateInfo cmd_pool_info;
			cmd_pool_info.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
			cmd_pool_info.pNext = nullptr;
			cmd_pool_info.queueFamilyIndex = p_context->get_graphics_queue_family_index();
			cmd_pool_info.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;

			VkResult res = vkCreateCommandPool(device, &cmd_pool_info, nullptr, &frames[i].command_pool);
			ERR_FAIL_COND_MSG(res, "vkCreateCommandPool failed with error " + itos(res) + ".");
		}

		{ // Create command buffers.

			VkCommandBufferAllocateInfo cmdbuf;
			// No command buffer exists, create it.
			cmdbuf.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
			cmdbuf.pNext = nullptr;
			cmdbuf.commandPool = frames[i].command_pool;
			cmdbuf.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
			cmdbuf.commandBufferCount = 1;

			VkResult err = vkAllocateCommandBuffers(device, &cmdbuf, &frames[i].setup_command_buffer);
			ERR_CONTINUE_MSG(err, "vkAllocateCommandBuffers failed with error " + itos(err) + ".");

			err = vkAllocateCommandBuffers(device, &cmdbuf, &frames[i].draw_command_buffer);
			ERR_CONTINUE_MSG(err, "vkAllocateCommandBuffers failed with error " + itos(err) + ".");
		}

		{
			// Create query pool.
			VkQueryPoolCreateInfo query_pool_create_info;
			query_pool_create_info.sType = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO;
			query_pool_create_info.flags = 0;
			query_pool_create_info.pNext = nullptr;
			query_pool_create_info.queryType = VK_QUERY_TYPE_TIMESTAMP;
			query_pool_create_info.queryCount = max_timestamp_query_elements;
			query_pool_create_info.pipelineStatistics = 0;

			vkCreateQueryPool(device, &query_pool_create_info, nullptr, &frames[i].timestamp_pool);

			frames[i].timestamp_names.resize(max_timestamp_query_elements);
			frames[i].timestamp_cpu_values.resize(max_timestamp_query_elements);
			frames[i].timestamp_count = 0;
			frames[i].timestamp_result_names.resize(max_timestamp_query_elements);
			frames[i].timestamp_cpu_result_values.resize(max_timestamp_query_elements);
			frames[i].timestamp_result_values.resize(max_timestamp_query_elements);
			frames[i].timestamp_result_count = 0;
		}
	}

	{
		// Begin the first command buffer for the first frame, so
		// setting up things can be done in the meantime until swap_buffers(), which is called before advance.
		VkCommandBufferBeginInfo cmdbuf_begin;
		cmdbuf_begin.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
		cmdbuf_begin.pNext = nullptr;
		cmdbuf_begin.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
		cmdbuf_begin.pInheritanceInfo = nullptr;

		VkResult err = vkBeginCommandBuffer(frames[0].setup_command_buffer, &cmdbuf_begin);
		ERR_FAIL_COND_MSG(err, "vkBeginCommandBuffer failed with error " + itos(err) + ".");

		err = vkBeginCommandBuffer(frames[0].draw_command_buffer, &cmdbuf_begin);
		ERR_FAIL_COND_MSG(err, "vkBeginCommandBuffer failed with error " + itos(err) + ".");
		if (local_device.is_null()) {
			context->set_setup_buffer(frames[0].setup_command_buffer); // Append now so it's added before everything else.
			context->append_command_buffer(frames[0].draw_command_buffer);
		}
	}

	for (int i = 0; i < frame_count; i++) {
		//Reset all queries in a query pool before doing any operations with them.
		vkCmdResetQueryPool(frames[0].setup_command_buffer, frames[i].timestamp_pool, 0, max_timestamp_query_elements);
	}

	staging_buffer_block_size = GLOBAL_GET("rendering/rendering_device/staging_buffer/block_size_kb");
	staging_buffer_block_size = MAX(4u, staging_buffer_block_size);
	staging_buffer_block_size *= 1024; // Kb -> bytes.
	staging_buffer_max_size = GLOBAL_GET("rendering/rendering_device/staging_buffer/max_size_mb");
	staging_buffer_max_size = MAX(1u, staging_buffer_max_size);
	staging_buffer_max_size *= 1024 * 1024;

	if (staging_buffer_max_size < staging_buffer_block_size * 4) {
		// Validate enough blocks.
		staging_buffer_max_size = staging_buffer_block_size * 4;
	}
	texture_upload_region_size_px = GLOBAL_GET("rendering/rendering_device/staging_buffer/texture_upload_region_size_px");
	texture_upload_region_size_px = nearest_power_of_2_templated(texture_upload_region_size_px);

	frames_drawn = frame_count; // Start from frame count, so everything else is immediately old.

	// Ensure current staging block is valid and at least one per frame exists.
	staging_buffer_current = 0;
	staging_buffer_used = false;

	for (int i = 0; i < frame_count; i++) {
		// Staging was never used, create a block.
		Error err = _insert_staging_block();
		ERR_CONTINUE(err != OK);
	}

	max_descriptors_per_pool = GLOBAL_GET("rendering/rendering_device/vulkan/max_descriptors_per_pool");

	// Check to make sure DescriptorPoolKey is good.
	static_assert(sizeof(uint64_t) * 3 >= UNIFORM_TYPE_MAX * sizeof(uint16_t));

	draw_list = nullptr;
	draw_list_count = 0;
	draw_list_split = false;

	compute_list = nullptr;

	pipelines_cache.file_path = "user://vulkan/pipelines";
	pipelines_cache.file_path += "." + context->get_device_name().validate_filename().replace(" ", "_").to_lower();
	if (Engine::get_singleton()->is_editor_hint()) {
		pipelines_cache.file_path += ".editor";
	}
	pipelines_cache.file_path += ".cache";

	// Prepare most fields now.
	VkPhysicalDeviceProperties props;
	vkGetPhysicalDeviceProperties(context->get_physical_device(), &props);
	pipelines_cache.header.magic = 868 + VK_PIPELINE_CACHE_HEADER_VERSION_ONE;
	pipelines_cache.header.device_id = props.deviceID;
	pipelines_cache.header.vendor_id = props.vendorID;
	pipelines_cache.header.driver_version = props.driverVersion;
	memcpy(pipelines_cache.header.uuid, props.pipelineCacheUUID, VK_UUID_SIZE);
	pipelines_cache.header.driver_abi = sizeof(void *);

	_load_pipeline_cache();
	print_verbose(vformat("Startup PSO cache (%.1f MiB)", pipelines_cache.buffer.size() / (1024.0f * 1024.0f)));
	VkPipelineCacheCreateInfo cache_info = {};
	cache_info.sType = VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO;
	cache_info.pNext = nullptr;
	if (context->get_pipeline_cache_control_support()) {
		cache_info.flags = VK_PIPELINE_CACHE_CREATE_EXTERNALLY_SYNCHRONIZED_BIT;
	}
	cache_info.initialDataSize = pipelines_cache.buffer.size();
	cache_info.pInitialData = pipelines_cache.buffer.ptr();
	VkResult err = vkCreatePipelineCache(device, &cache_info, nullptr, &pipelines_cache.cache_object);

	if (err != VK_SUCCESS) {
		WARN_PRINT("vkCreatePipelinecache failed with error " + itos(err) + ".");
	}
}

```