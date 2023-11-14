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
