---
title: Vulkan Render 的创建
comments: true
---

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

	{ // 创建 vma 内存分配器

		VmaAllocatorCreateInfo allocatorInfo;
		memset(&allocatorInfo, 0, sizeof(VmaAllocatorCreateInfo));
		allocatorInfo.physicalDevice = p_context->get_physical_device();
		allocatorInfo.device = device;
		allocatorInfo.instance = p_context->get_instance();
		vmaCreateAllocator(&allocatorInfo, &allocator);
	}
	// 帧缓冲的的数量比 交换链的图片数量多1
	frames.resize(frame_count);
	frame = 0;
	// 为 frame 创建 setup command buffer、 draw command buffer、query pool
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
			// 创建设置的命令缓冲区
			VkResult err = vkAllocateCommandBuffers(device, &cmdbuf, &frames[i].setup_command_buffer);
			ERR_CONTINUE_MSG(err, "vkAllocateCommandBuffers failed with error " + itos(err) + ".");
			// 创建绘制的命令缓冲区
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
	// 设置暂存缓冲区的大小
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
	// 最大描述符的数量
	max_descriptors_per_pool = GLOBAL_GET("rendering/rendering_device/vulkan/max_descriptors_per_pool");

	// Check to make sure DescriptorPoolKey is good.
	static_assert(sizeof(uint64_t) * 3 >= UNIFORM_TYPE_MAX * sizeof(uint16_t));

	draw_list = nullptr;
	draw_list_count = 0;
	draw_list_split = false;

	compute_list = nullptr;

	// 设置渲染管线的缓存路径 user://vulkan/pipelines.xxxxx.cache
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

### 1.4 RenderServer
```cpp
rendering_server = memnew(RenderingServerDefault(OS::get_singleton()->get_render_thread_mode() == OS::RENDER_SEPARATE_THREAD));

rendering_server->init();
//rendering_server->call_set_use_vsync(OS::get_singleton()->_use_vsync);
rendering_server->set_render_loop_enabled(!disable_render_loop);

if (profile_gpu || (!editor && bool(GLOBAL_GET("debug/settings/stdout/print_gpu_profile")))) {
	rendering_server->set_print_gpu_profile(true);
}
```
看代码中有个渲染线程模式的参数，RenderServer中渲染线程类型由这个参数决定。翻到之前的代码可以看到程序启动时，可以执行渲染的线程模式：
```cpp
} else if (I->get() == "--render-thread") { // render thread mode

	if (I->next()) {
		if (I->next()->get() == "safe") {
			rtm = OS::RENDER_THREAD_SAFE;
		} else if (I->next()->get() == "unsafe") {
			rtm = OS::RENDER_THREAD_UNSAFE;
		} else if (I->next()->get() == "separate") {
			rtm = OS::RENDER_SEPARATE_THREAD;
		} else {
			OS::get_singleton()->print("Unknown render thread mode, aborting.\nValid options are 'unsafe', 'safe' and 'separate'.\n");
			goto error;
		}

		N = I->next()->next();
	} else {
		OS::get_singleton()->print("Missing render thread mode argument, aborting.\n");
		goto error;
	}
```
Godot 中渲染线程有三种类型：
- 线程安全
- 非线程安全
- 分离线程

如果没有在启动初期设置，那么在后面的代码中会自动配置
```cpp
	if (rtm == -1) {
		rtm = GLOBAL_DEF("rendering/driver/threads/thread_model", OS::RENDER_THREAD_SAFE);
	}

	if (rtm >= 0 && rtm < 3) {
		if (editor || project_manager) {
			// Editor and project manager cannot run with rendering in a separate thread (they will crash on startup).
			rtm = OS::RENDER_THREAD_SAFE;
		}
		OS::get_singleton()->_render_thread_mode = OS::RenderThreadMode(rtm);
	}
```
默认就是线程安全的模式。编辑器和项目管理器也是也以线程安全的模式运行。  

RenderingServerDefault 类集成 RenderingServer，RenderingServer是一个抽象类，定义了实现渲染服务的所有接口，我比较在意的是 RenderingServerDefault 的成员变量 command_queue
```cpp
RenderingServerDefault::RenderingServerDefault(bool p_create_thread) :
		command_queue(p_create_thread) {
	RenderingServer::init();

	create_thread = p_create_thread;

	if (!p_create_thread) {
		server_thread = Thread::get_caller_id();
	} else {
		server_thread = 0;
	}

	RSG::threaded = p_create_thread;
	RSG::canvas = memnew(RendererCanvasCull);
	RSG::viewport = memnew(RendererViewport);
	RendererSceneCull *sr = memnew(RendererSceneCull);
	RSG::camera_attributes = memnew(RendererCameraAttributes);
	RSG::scene = sr;
	RSG::rasterizer = RendererCompositor::create();
	RSG::utilities = RSG::rasterizer->get_utilities();
	RSG::light_storage = RSG::rasterizer->get_light_storage();
	RSG::material_storage = RSG::rasterizer->get_material_storage();
	RSG::mesh_storage = RSG::rasterizer->get_mesh_storage();
	RSG::particles_storage = RSG::rasterizer->get_particles_storage();
	RSG::texture_storage = RSG::rasterizer->get_texture_storage();
	RSG::gi = RSG::rasterizer->get_gi();
	RSG::fog = RSG::rasterizer->get_fog();
	RSG::canvas_render = RSG::rasterizer->get_canvas();
	sr->set_scene_render(RSG::rasterizer->get_scene());

	frame_profile_frame = 0;
}
```
通过构造函数中的代码也可以看出，command_queue 应该是储存绘制命令的队列，至于它的类型解析我放到了（[CommandQueueMT解析](05CommandQueueMT.md)）中。init函数则是渲染器定义了渲染中的一些参数，后面创建了画布裁剪、场景裁剪等等，光照和材质等信息是作为 storage 进行存储。
至于 RSG 其实是一个全局的 RenderServer别名。
```cpp

class RenderingServerGlobals {
public:
	static bool threaded;

	static RendererUtilities *utilities;
	static RendererLightStorage *light_storage;
	static RendererMaterialStorage *material_storage;
	static RendererMeshStorage *mesh_storage;
	static RendererParticlesStorage *particles_storage;
	static RendererTextureStorage *texture_storage;
	static RendererGI *gi;
	static RendererFog *fog;
	static RendererCameraAttributes *camera_attributes;
	static RendererCanvasRender *canvas_render;
	static RendererCompositor *rasterizer;

	static RendererCanvasCull *canvas;
	static RendererViewport *viewport;
	static RenderingMethod *scene;
};

#define RSG RenderingServerGlobals
```
也就是在 RenderServerDefault 中创建的一列类渲染相关的对象 都被赋给了 RSG 中的静态成员指针，所以 RSG 也是单例模式的一种运用。

创建完 RenderServerDefault 调用它的 init 函数。
```cpp
void RenderingServerDefault::init() {
	if (create_thread) {
		print_verbose("RenderingServerWrapMT: Creating render thread");
		DisplayServer::get_singleton()->release_rendering_thread();
		if (create_thread) {
			thread.start(_thread_callback, this);
			print_verbose("RenderingServerWrapMT: Starting render thread");
		}
		while (!draw_thread_up.is_set()) {
			OS::get_singleton()->delay_usec(1000);
		}
		print_verbose("RenderingServerWrapMT: Finished render thread");
	} else {
		_init();
	}
}
void RenderingServerDefault::_init() {
	RSG::rasterizer->initialize();
}
```