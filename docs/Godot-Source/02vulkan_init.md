---
comments: true
---

# Godot渲染流程
### 1.2 vulkan 的实例化
在 DisplayServerWindows 的构造函数中 可以看到 VulkanContextWindows 被创建，然后调用 initialize 函数进行初始化。
``` cpp
#if defined(VULKAN_ENABLED)
	if (rendering_driver == "vulkan") {
		context_vulkan = memnew(VulkanContextWindows);
		if (context_vulkan->initialize() != OK) {
			memdelete(context_vulkan);
			context_vulkan = nullptr;
			r_error = ERR_UNAVAILABLE;
			return;
		}
	}
#endif
```
一步一步来，先看 VulkanContextWindows 是怎么定义的：
``` cpp
class VulkanContextWindows : public VulkanContext {
	virtual const char *_get_platform_surface_extension() const;

public:
	Error window_create(DisplayServer::WindowID p_window_id, DisplayServer::VSyncMode p_vsync_mode, HWND p_window, HINSTANCE p_instance, int p_width, int p_height);

	VulkanContextWindows();
	~VulkanContextWindows();
};
```
VulkanContextWindows 是 VulkanContext 的子类，通过上面的代码也可以看出大部分的功能是有父类实现。直接阅读VulkanContext 的构造函数代码、
```cpp
VulkanContext::VulkanContext() {
	command_buffer_queue.resize(1); // First one is always the setup command.
	command_buffer_queue.write[0] = nullptr;
}
```
VulkanContext 的构造函数 将 命令缓冲队列大小设置为1，第一个命令总是设置命令。VulkanContext 看成员变量的定义，它其实是整合了 VkInstance、VKPhyscalDevice、VkDevice、VkQueue等，基本上完成了大部分vulkan设备的初始化。里面细节感兴趣可以自行阅读代码。程序下一步执行调用了 initialize 函数。
``` cpp
Error VulkanContext::initialize() {
#ifdef USE_VOLK
	if (volkInitialize() != VK_SUCCESS) {
		return FAILED;
	}
#endif

	Error err = _create_instance();
	if (err != OK) {
		return err;
	}

	return OK;
}
```
使用 volk 第三方库 获取 Vulkan 的函数， 再通过 _create_instance 函数创建 VkInstance, 函数前面加下划线估计是为了区分是否是用于底层的内部API，跟 Ogre3D 类似。
``` cpp
Error VulkanContext::_create_instance() {
	// Obtain Vulkan version.
	_obtain_vulkan_version(); // 获取 vulkan 版本 默认版本为 1.0

	// Initialize extensions.
	{
		Error err = _initialize_instance_extensions(); // 这个会 vulkan 的很好理解，就是查询 instance 的扩展是否都被满足
		if (err != OK) {
			return err;
		}
	}
	// 获取可支持的扩展
	int enabled_extension_count = 0;
	const char *enabled_extension_names[MAX_EXTENSIONS];
	ERR_FAIL_COND_V(enabled_instance_extension_names.size() > MAX_EXTENSIONS, ERR_CANT_CREATE);
	for (const CharString &extension_name : enabled_instance_extension_names) {
		enabled_extension_names[enabled_extension_count++] = extension_name.ptr();
	}

	// We'll set application version to the Vulkan version we're developing against, even if our instance is based on
	// an older Vulkan version, devices can still support newer versions of Vulkan.
	// The exception is when we're on Vulkan 1.0, we should not set this to anything but 1.0.
	// Note that this value is only used by validation layers to warn us about version issues.
	uint32_t application_api_version = instance_api_version == VK_API_VERSION_1_0 ? VK_API_VERSION_1_0 : VK_API_VERSION_1_2;

	CharString cs = GLOBAL_GET("application/config/name").operator String().utf8();
	const VkApplicationInfo app = {
		/*sType*/ VK_STRUCTURE_TYPE_APPLICATION_INFO,
		/*pNext*/ nullptr,
		/*pApplicationName*/ cs.get_data(),
		/*applicationVersion*/ 0, // It would be really nice if we store a version number in project settings, say "application/config/version"
		/*pEngineName*/ VERSION_NAME,
		/*engineVersion*/ VK_MAKE_VERSION(VERSION_MAJOR, VERSION_MINOR, VERSION_PATCH),
		/*apiVersion*/ application_api_version
	};
	VkInstanceCreateInfo inst_info{};
	inst_info.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
	inst_info.pApplicationInfo = &app;
	inst_info.enabledExtensionCount = enabled_extension_count;
	inst_info.ppEnabledExtensionNames = (const char *const *)enabled_extension_names;
	if (_use_validation_layers()) { // 获取 validation layers
		_get_preferred_validation_layers(&inst_info.enabledLayerCount, &inst_info.ppEnabledLayerNames);
	}

	/*
	 * This is info for a temp callback to use during CreateInstance.
	 * After the instance is created, we use the instance-based
	 * function to register the final callback.
	 */
	VkDebugUtilsMessengerCreateInfoEXT dbg_messenger_create_info = {};
	VkDebugReportCallbackCreateInfoEXT dbg_report_callback_create_info = {};
	if (is_instance_extension_enabled(VK_EXT_DEBUG_UTILS_EXTENSION_NAME)) {
		// VK_EXT_debug_utils style.
		dbg_messenger_create_info.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
		dbg_messenger_create_info.pNext = nullptr;
		dbg_messenger_create_info.flags = 0;
		dbg_messenger_create_info.messageSeverity =
				VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
		dbg_messenger_create_info.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT |
				VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT |
				VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
		dbg_messenger_create_info.pfnUserCallback = _debug_messenger_callback;
		dbg_messenger_create_info.pUserData = this;
		inst_info.pNext = &dbg_messenger_create_info;
	} else if (is_instance_extension_enabled(VK_EXT_DEBUG_REPORT_EXTENSION_NAME)) {
		dbg_report_callback_create_info.sType = VK_STRUCTURE_TYPE_DEBUG_REPORT_CALLBACK_CREATE_INFO_EXT;
		dbg_report_callback_create_info.flags = VK_DEBUG_REPORT_INFORMATION_BIT_EXT |
				VK_DEBUG_REPORT_WARNING_BIT_EXT |
				VK_DEBUG_REPORT_PERFORMANCE_WARNING_BIT_EXT |
				VK_DEBUG_REPORT_ERROR_BIT_EXT |
				VK_DEBUG_REPORT_DEBUG_BIT_EXT;
		dbg_report_callback_create_info.pfnCallback = _debug_report_callback;
		dbg_report_callback_create_info.pUserData = this;
		inst_info.pNext = &dbg_report_callback_create_info;
	}

	VkResult err;

	if (vulkan_hooks) { // 这里定义了一个 获取 vulkan 实例的钩子 
		if (!vulkan_hooks->create_vulkan_instance(&inst_info, &inst)) {
			return ERR_CANT_CREATE;
		}
	} else {
		err = vkCreateInstance(&inst_info, nullptr, &inst);
		ERR_FAIL_COND_V_MSG(err == VK_ERROR_INCOMPATIBLE_DRIVER, ERR_CANT_CREATE,
				"Cannot find a compatible Vulkan installable client driver (ICD).\n\n"
				"vkCreateInstance Failure");
		ERR_FAIL_COND_V_MSG(err == VK_ERROR_EXTENSION_NOT_PRESENT, ERR_CANT_CREATE,
				"Cannot find a specified extension library.\n"
				"Make sure your layers path is set appropriately.\n"
				"vkCreateInstance Failure");
		ERR_FAIL_COND_V_MSG(err, ERR_CANT_CREATE,
				"vkCreateInstance failed.\n\n"
				"Do you have a compatible Vulkan installable client driver (ICD) installed?\n"
				"Please look at the Getting Started guide for additional information.\n"
				"vkCreateInstance Failure");
	}

	inst_initialized = true;  // 设置 vkinstance 实例化成功的标志

#ifdef USE_VOLK
	volkLoadInstance(inst); // 加载 volk
#endif

	if (is_instance_extension_enabled(VK_EXT_DEBUG_UTILS_EXTENSION_NAME)) { // 获取调试函数的函数指针
		// Setup VK_EXT_debug_utils function pointers always (we use them for debug labels and names).
		CreateDebugUtilsMessengerEXT =
				(PFN_vkCreateDebugUtilsMessengerEXT)vkGetInstanceProcAddr(inst, "vkCreateDebugUtilsMessengerEXT");
		DestroyDebugUtilsMessengerEXT =
				(PFN_vkDestroyDebugUtilsMessengerEXT)vkGetInstanceProcAddr(inst, "vkDestroyDebugUtilsMessengerEXT");
		SubmitDebugUtilsMessageEXT =
				(PFN_vkSubmitDebugUtilsMessageEXT)vkGetInstanceProcAddr(inst, "vkSubmitDebugUtilsMessageEXT");
		CmdBeginDebugUtilsLabelEXT =
				(PFN_vkCmdBeginDebugUtilsLabelEXT)vkGetInstanceProcAddr(inst, "vkCmdBeginDebugUtilsLabelEXT");
		CmdEndDebugUtilsLabelEXT =
				(PFN_vkCmdEndDebugUtilsLabelEXT)vkGetInstanceProcAddr(inst, "vkCmdEndDebugUtilsLabelEXT");
		CmdInsertDebugUtilsLabelEXT =
				(PFN_vkCmdInsertDebugUtilsLabelEXT)vkGetInstanceProcAddr(inst, "vkCmdInsertDebugUtilsLabelEXT");
		SetDebugUtilsObjectNameEXT =
				(PFN_vkSetDebugUtilsObjectNameEXT)vkGetInstanceProcAddr(inst, "vkSetDebugUtilsObjectNameEXT");
		if (nullptr == CreateDebugUtilsMessengerEXT || nullptr == DestroyDebugUtilsMessengerEXT ||
				nullptr == SubmitDebugUtilsMessageEXT || nullptr == CmdBeginDebugUtilsLabelEXT ||
				nullptr == CmdEndDebugUtilsLabelEXT || nullptr == CmdInsertDebugUtilsLabelEXT ||
				nullptr == SetDebugUtilsObjectNameEXT) {
			ERR_FAIL_V_MSG(ERR_CANT_CREATE,
					"GetProcAddr: Failed to init VK_EXT_debug_utils\n"
					"GetProcAddr: Failure");
		}

		err = CreateDebugUtilsMessengerEXT(inst, &dbg_messenger_create_info, nullptr, &dbg_messenger);
		switch (err) {
			case VK_SUCCESS:
				break;
			case VK_ERROR_OUT_OF_HOST_MEMORY:
				ERR_FAIL_V_MSG(ERR_CANT_CREATE,
						"CreateDebugUtilsMessengerEXT: out of host memory\n"
						"CreateDebugUtilsMessengerEXT Failure");
				break;
			default:
				ERR_FAIL_V_MSG(ERR_CANT_CREATE,
						"CreateDebugUtilsMessengerEXT: unknown failure\n"
						"CreateDebugUtilsMessengerEXT Failure");
				ERR_FAIL_V(ERR_CANT_CREATE);
				break;
		}
	} else if (is_instance_extension_enabled(VK_EXT_DEBUG_REPORT_EXTENSION_NAME)) {
		CreateDebugReportCallbackEXT = (PFN_vkCreateDebugReportCallbackEXT)vkGetInstanceProcAddr(inst, "vkCreateDebugReportCallbackEXT");
		DebugReportMessageEXT = (PFN_vkDebugReportMessageEXT)vkGetInstanceProcAddr(inst, "vkDebugReportMessageEXT");
		DestroyDebugReportCallbackEXT = (PFN_vkDestroyDebugReportCallbackEXT)vkGetInstanceProcAddr(inst, "vkDestroyDebugReportCallbackEXT");

		if (nullptr == CreateDebugReportCallbackEXT || nullptr == DebugReportMessageEXT || nullptr == DestroyDebugReportCallbackEXT) {
			ERR_FAIL_V_MSG(ERR_CANT_CREATE,
					"GetProcAddr: Failed to init VK_EXT_debug_report\n"
					"GetProcAddr: Failure");
		}

		err = CreateDebugReportCallbackEXT(inst, &dbg_report_callback_create_info, nullptr, &dbg_debug_report);
		switch (err) {
			case VK_SUCCESS:
				break;
			case VK_ERROR_OUT_OF_HOST_MEMORY:
				ERR_FAIL_V_MSG(ERR_CANT_CREATE,
						"CreateDebugReportCallbackEXT: out of host memory\n"
						"CreateDebugReportCallbackEXT Failure");
				break;
			default:
				ERR_FAIL_V_MSG(ERR_CANT_CREATE,
						"CreateDebugReportCallbackEXT: unknown failure\n"
						"CreateDebugReportCallbackEXT Failure");
				ERR_FAIL_V(ERR_CANT_CREATE);
				break;
		}
	}

	return OK;
}
```
如果有 vulkan 基础， create_instance 函数应该都能看懂，全是配置 vulkan，相信在写 vulkan demo 的时候也写了不少了。上面代码执行完了以后又回到了 DisplayServerWindows 的构造函数中继续执行。
执行 DisplayServerWindows::_create_window 函数，在这个函数中有一段 vulkan 代码。
``` cpp
#ifdef VULKAN_ENABLED
		if (context_vulkan) {
			if (context_vulkan->window_create(id, p_vsync_mode, wd.hWnd, hInstance, WindowRect.right - WindowRect.left, WindowRect.bottom - WindowRect.top) != OK) {
				memdelete(context_vulkan);
				context_vulkan = nullptr;
				windows.erase(id);
				ERR_FAIL_V_MSG(INVALID_WINDOW_ID, "Failed to create Vulkan Window.");
			}
			wd.context_created = true;
		}
#endif

Error VulkanContextWindows::window_create(DisplayServer::WindowID p_window_id, DisplayServer::VSyncMode p_vsync_mode, HWND p_window, HINSTANCE p_instance, int p_width, int p_height) {
	VkWin32SurfaceCreateInfoKHR createInfo;
	createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
	createInfo.pNext = nullptr;
	createInfo.flags = 0;
	createInfo.hinstance = p_instance;
	createInfo.hwnd = p_window;
	VkSurfaceKHR surface;
	VkResult err = vkCreateWin32SurfaceKHR(get_instance(), &createInfo, nullptr, &surface);
	ERR_FAIL_COND_V(err, ERR_CANT_CREATE);
	return _window_create(p_window_id, p_vsync_mode, surface, p_width, p_height);
}
```
通过 context_vulkan 对象调用 window_create 函数，实参有窗体的 id，垂直同步模式，windows 窗体相关的句柄，窗口的宽高，如果没创建成功就删除 id 和 context vulkan，比较常见的失败处理方式。

context_vulkan 是 VulkanContextWindows 的实例对象，调用的是自身的 window_create 函数。 在函数中 创建了 vulkan 必要的 win32surface，这个是创建交换链的必要数据。随后在 返回语句中调用了 父类的 _window_create 内部函数。
``` cpp
Error VulkanContext::_window_create(DisplayServer::WindowID p_window_id, DisplayServer::VSyncMode p_vsync_mode, VkSurfaceKHR p_surface, int p_width, int p_height) {
	ERR_FAIL_COND_V(windows.has(p_window_id), ERR_INVALID_PARAMETER);

	if (!device_initialized) {
		Error err = _create_physical_device(p_surface); // 创建 物理设备，这个跟 GPU 是一一对应的关系，有多少张显卡，就可以创建相应数量的物理设备
		ERR_FAIL_COND_V(err != OK, ERR_CANT_CREATE);
	}

	if (!queues_initialized) {
		// We use a single GPU, but we need a surface to initialize the
		// queues, so this process must be deferred until a surface
		// is created.
		Error err = _initialize_queues(p_surface); // 创建逻辑设备，获取队列，如果有个队列及支持 graphics 又支持 present，那么它将优先被选中 
		ERR_FAIL_COND_V(err != OK, ERR_CANT_CREATE);
	}

	Window window;
	window.surface = p_surface;
	window.width = p_width;
	window.height = p_height;
	window.vsync_mode = p_vsync_mode;
	Error err = _update_swap_chain(&window); // 更新交换链
	ERR_FAIL_COND_V(err != OK, ERR_CANT_CREATE);

	windows[p_window_id] = window;
	return OK;
}
```
其中 _update_swap_chain 创建了 swapchain, renderPass。再通过 renderPass 和 imageview 创建 frameBuffer，还是 vulkan 的默认流程，只不过这里用了一个临时的 rende pass。
``` cpp
#if defined(VULKAN_ENABLED)

	if (rendering_driver == "vulkan") {
		rendering_device_vulkan = memnew(RenderingDeviceVulkan);
		rendering_device_vulkan->initialize(context_vulkan);

		RendererCompositorRD::make_current();
	}
#endif
```
我觉得这部分主要是在初始化 vulkan,但是真正的渲染操作却还没执行。