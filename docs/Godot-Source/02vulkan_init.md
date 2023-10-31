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
