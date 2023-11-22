# Godot渲染流程
### 1.3 Vulkan Render Device 的创建

RenderingDeviceVulkan 继承于 RenderingDevice，而 RenderDevcice 则抽象了渲染了大部分的功能。我在翻译 The future of Render in Godot 视频的时候, 看到负责人有介绍下一代的 RD 渲染器将会有更高级别的抽象，Metal 和 DX12 都会基于RD 渲染器。但是粗略的查看代码之后，RenderingDevice 这个类其实还是没有抽象得很干净。

RenderDevice 负责的内容很多，它具有底层渲染的版本信息，是CPU，还是GPU。是 Vulkan 还是 OpenGL. Texture、FrameBuffer、Buffer 和 DrawList等的构建。我觉得还是有必要详细的解释 RenderDevice的代码

#### 1.3.1 RenderDevice
我把它单独写成一个文件，方便后面的引用，详细内容可以点击链接进入
[RenderDevice类介绍](04render_device.md)
#### 1.3.2 Vulkan Render Device