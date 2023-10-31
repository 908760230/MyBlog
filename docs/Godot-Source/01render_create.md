---
comments: true
---

# Godot渲染流程

## 1. 渲染系统的构建
这次分析的源码是基于Godot 4.2 beta版本。 CommitId是： 09946f79bd8215b2c6332de8821737580909a91c

这篇文章的内容主要是断点调试跟进Godot渲染系统是怎么创建图形渲染设备，调用图形API接口，绘制图像数据。分析的主要是与Vulkan相关的内容，opengl3的类似就不多赘述了，而且与图形不相关的内容我会直接跳过。

### 1.1 渲染系统的配置 
通过断点进入到 Main::setup 函数中，中间有很长的一段参数解析内容，底层的图形接口可以通过命令行指定。下面是指定各个平台默认的图形驱动代码，通过 VULKAN_ENABLED 宏来指定图形底层为vulkan，而 VULKAN_ENABLED 宏的指定是定义在detect.py中
``` cpp
	{
		String driver_hints = "";
#ifdef VULKAN_ENABLED
		driver_hints = "vulkan";
#endif

		String default_driver = driver_hints.get_slice(",", 0);

		// For now everything defaults to vulkan when available. This can change in future updates.
		GLOBAL_DEF_RST("rendering/rendering_device/driver", default_driver);
		GLOBAL_DEF_RST(PropertyInfo(Variant::STRING, "rendering/rendering_device/driver.windows", PROPERTY_HINT_ENUM, driver_hints), default_driver);
		GLOBAL_DEF_RST(PropertyInfo(Variant::STRING, "rendering/rendering_device/driver.linuxbsd", PROPERTY_HINT_ENUM, driver_hints), default_driver);
		GLOBAL_DEF_RST(PropertyInfo(Variant::STRING, "rendering/rendering_device/driver.android", PROPERTY_HINT_ENUM, driver_hints), default_driver);
		GLOBAL_DEF_RST(PropertyInfo(Variant::STRING, "rendering/rendering_device/driver.ios", PROPERTY_HINT_ENUM, driver_hints), default_driver);
		GLOBAL_DEF_RST(PropertyInfo(Variant::STRING, "rendering/rendering_device/driver.macos", PROPERTY_HINT_ENUM, driver_hints), default_driver);
	}
```

下面代码指定了主机、手机等默认的渲染方式，主机端默认是forward+的方式，移动端是 mobile,而web端则是 gl_compatibility： opengl兼容模式
``` cpp
default_renderer = renderer_hints.get_slice(",", 0);
GLOBAL_DEF_RST_BASIC(PropertyInfo(Variant::STRING, "rendering/renderer/rendering_method", PROPERTY_HINT_ENUM, renderer_hints), default_renderer);
GLOBAL_DEF_RST_BASIC("rendering/renderer/rendering_method.mobile", default_renderer_mobile);
GLOBAL_DEF_RST_BASIC("rendering/renderer/rendering_method.web", "gl_compatibility"); // This is a bit of a hack until we have WebGPU support.
```
下面就到了重要的 DisplayServer 创建。DisplayServer 是从 OS 分离出来的一个类，用于管理所有与窗体相关的功能。至于分离的原因是一个系统可以有多个窗体。
``` cpp

// rendering_driver now held in static global String in main and initialized in setup()
Error err;
display_server = DisplayServer::create(display_driver_idx, rendering_driver, window_mode, window_vsync_mode, window_flags, window_position, window_size, init_screen, err);
if (err != OK || display_server == nullptr) {
	// We can't use this display server, try other ones as fallback.
	// Skip headless (always last registered) because that's not what users
	// would expect if they didn't request it explicitly.
	for (int i = 0; i < DisplayServer::get_create_function_count() - 1; i++) {
		if (i == display_driver_idx) {
			continue; // Don't try the same twice.
		}
		display_server = DisplayServer::create(i, rendering_driver, window_mode, window_vsync_mode, window_flags, window_position, window_size, init_screen, err);
		if (err == OK && display_server != nullptr) {
			break;
		}
	}
}
```
DisplayServer的创建是通过传入的参数，调用内部的create静态方法实现。当未创建成功，则会轮询DisplayerServer内部存储的create function。这里其实可以猜到DisplayServer内部有一个存储函数指针的数组，至于函数指针是怎么注册到DisplayeServer中，可以顺着代码往下看。先看下 DisplayServer::create 函数内部是怎么实现的。
``` cpp
DisplayServer *DisplayServer::create(int p_index, const String &p_rendering_driver, WindowMode p_mode, VSyncMode p_vsync_mode, uint32_t p_flags, const Vector2i *p_position, const Vector2i &p_resolution, int p_screen, Error &r_error) {
	ERR_FAIL_INDEX_V(p_index, server_create_count, nullptr);
	return server_create_functions[p_index].create_function(p_rendering_driver, p_mode, p_vsync_mode, p_flags, p_position, p_resolution, p_screen, r_error);
}
```
内部果然是差不多的实现方式，但是通过下标得到对象后又可以调其他函数，那么数组内部存储的就不是普通的函数指针了。
现在有两条路可以去追踪代码：
- 找到创建函数是怎么保存到 server_create_functions 这个数组中的
- 直接断点进入 create_function 中

虽然第二种方式更加直接，但是怎么创建函数是怎么注册还是不懂，所以选择第一种，找到注册的路径在哪里。先找到 server_create_functions 是怎么定义的
``` cpp
	typedef DisplayServer *(*CreateFunction)(const String &, WindowMode, VSyncMode, uint32_t, const Point2i *, const Size2i &, int p_screen, Error &r_error);
	typedef Vector<String> (*GetRenderingDriversFunction)();
	enum {
		MAX_SERVERS = 64
	};

	struct DisplayServerCreate {
		const char *name;
		CreateFunction create_function;
		GetRenderingDriversFunction get_rendering_drivers_function;
	};

	static DisplayServerCreate server_create_functions[MAX_SERVERS];
```
DisplayServerCreate 对两个函数指针的封装，一个是真正的创建函数的指针，另一个是获取字符串数组的无参函数指针。
通过断点定位注册函数的调用堆栈信息。发现调用的入口在OS_Windows的构造函数中
``` cpp
OS_Windows::OS_Windows(HINSTANCE _hInstance) {
	hInstance = _hInstance;

	CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);

#ifdef WASAPI_ENABLED
	AudioDriverManager::add_driver(&driver_wasapi);
#endif
#ifdef XAUDIO2_ENABLED
	AudioDriverManager::add_driver(&driver_xaudio2);
#endif

	DisplayServerWindows::register_windows_driver();
    .
    .
    .
}
```
DisplayServerWindows 是 DisplayServer 的子类，register_windows_driver 是静态函数，调用父类的静态函数 register_windows_driver 将DisplayServerWindows 的成员函数 以函数指针的形式进行传递。server_create_count - 1 的赋值操作 是为了保证 预先创建 headless 窗体参数总是处在最后。
``` cpp
DisplayServer *DisplayServerWindows::create_func(const String &p_rendering_driver, WindowMode p_mode, VSyncMode p_vsync_mode, uint32_t p_flags, const Vector2i *p_position, const Vector2i &p_resolution, int p_screen, Error &r_error) {
	DisplayServer *ds = memnew(DisplayServerWindows(p_rendering_driver, p_mode, p_vsync_mode, p_flags, p_position, p_resolution, p_screen, r_error));
	if (r_error != OK) {
		if (p_rendering_driver == "vulkan") {
			String executable_name = OS::get_singleton()->get_executable_path().get_file();
			OS::get_singleton()->alert(
					vformat("Your video card drivers seem not to support the required Vulkan version.\n\n"
							"If possible, consider updating your video card drivers or using the OpenGL 3 driver.\n\n"
							"You can enable the OpenGL 3 driver by starting the engine from the\n"
							"command line with the command:\n\n    \"%s\" --rendering-driver opengl3\n\n"
							"If you have recently updated your video card drivers, try rebooting.",
							executable_name),
					"Unable to initialize Vulkan video driver");
		} else {
			OS::get_singleton()->alert(
					"Your video card drivers seem not to support the required OpenGL 3.3 version.\n\n"
					"If possible, consider updating your video card drivers.\n\n"
					"If you have recently updated your video card drivers, try rebooting.",
					"Unable to initialize OpenGL video driver");
		}
	}
	return ds;
}

Vector<String> DisplayServerWindows::get_rendering_drivers_func() {
	Vector<String> drivers;

#ifdef VULKAN_ENABLED
	drivers.push_back("vulkan");
#endif
#ifdef GLES3_ENABLED
	drivers.push_back("opengl3");
	drivers.push_back("opengl3_angle");
#endif

	return drivers;
}

void DisplayServerWindows::register_windows_driver() {
	register_create_function("windows", create_func, get_rendering_drivers_func);
}
void DisplayServer::register_create_function(const char *p_name, CreateFunction p_function, GetRenderingDriversFunction p_get_drivers) {
	ERR_FAIL_COND(server_create_count == MAX_SERVERS);
	// Headless display server is always last
	server_create_functions[server_create_count] = server_create_functions[server_create_count - 1];
	server_create_functions[server_create_count - 1].name = p_name;
	server_create_functions[server_create_count - 1].create_function = p_function;
	server_create_functions[server_create_count - 1].get_rendering_drivers_function = p_get_drivers;
	server_create_count++;
}
```

已经找到 DisplayServer 中的创建函数是如何注册的，总体上就是子类 DisplayServerWindows 通过静态函数 向父类 DisplayServer 注册自己的创建函数，程序向下执行就到了 DisplayServerWindows::create_func 函数中了。函数创建了 DisplayServerWindows 的实例对象，想了解这个类的具体功能可以去看内部实现，里面涉及到图形的内容是创建了 VulkanContextWindows 实例。
``` cpp
DisplayServerWindows::DisplayServerWindows(const String &p_rendering_driver, WindowMode p_mode, VSyncMode p_vsync_mode, uint32_t p_flags, const Vector2i *p_position, const Vector2i &p_resolution, int p_screen, Error &r_error) {
    .
    .
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
    .
    .
}
```

经过了弯弯绕绕，终于快触及到 vulkan 真正的内容了，Godot在启动期间各种配置、注册、校验。上面的代码我只保留了Godot中最核心的部分，因为这样才可以抓住最主要的逻辑链路。我想把涉及 Vulkan 的内容整合在一起，所以 vulkan 的初始化 以及 win32 surface 的传递 我会写在下一章。