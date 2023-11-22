# Render Device 类介绍
在这一章节，我的想法是按照这个类的功能来划分，如果照着代码一行一行解释会显得非常的冗长，不适合阅读。

RenderDevice 是 Object 的子类，其最主要的作用是定义类的接口。Object 类 的作用我不继续写了，今后也会写 Object 类的文章，就不赘述了。
类定义的下一行是如下代码，
```cpp
	GDCLASS(RenderingDevice, Object)
```
它的作用是实现反射，至于怎么实现的可以去看 Godot 反射实现的文章.

## 1、设备信息
代码中通过这个定义了设别的基本型号，是什么类型的。目前预留了 DirectX，之前看了 The future of Render in Godot 的演讲，后面应该还会有 metal 等关键字。
```cpp
enum DeviceFamily {
		DEVICE_UNKNOWN,
		DEVICE_OPENGL,
		DEVICE_VULKAN,
		DEVICE_DIRECTX
	};	
struct Capabilities {
	// main device info
	DeviceFamily device_family = DEVICE_UNKNOWN;
	uint32_t version_major = 1.0;
	uint32_t version_minor = 0.0;
};

Capabilities device_capabilities;
```

## 2、Texture的创建
可以看到类中其实之定义了 Texture 的属性格式和创建的相关接口，具体的实现实际上是在子类中。
```cpp
struct TextureFormat {
	DataFormat format;
	uint32_t width;
	uint32_t height;
	uint32_t depth;
	uint32_t array_layers;
	uint32_t mipmaps;
	TextureType texture_type;
	TextureSamples samples;
	uint32_t usage_bits;
	Vector<DataFormat> shareable_formats;
	bool is_resolve_buffer = false;

	bool operator==(const TextureFormat &b) const {
		if (format != b.format) {
			return false;
		} else if (width != b.width) {
			return false;
		} else if (height != b.height) {
			return false;
		} else if (depth != b.depth) {
			return false;
		} else if (array_layers != b.array_layers) {
			return false;
		} else if (mipmaps != b.mipmaps) {
			return false;
		} else if (texture_type != b.texture_type) {
			return false;
		} else if (samples != b.samples) {
			return false;
		} else if (usage_bits != b.usage_bits) {
			return false;
		} else if (shareable_formats != b.shareable_formats) {
			return false;
		} else {
			return true;
		}
	}

	TextureFormat() {
		format = DATA_FORMAT_R8_UNORM;
		width = 1;
		height = 1;
		depth = 1;
		array_layers = 1;
		mipmaps = 1;
		texture_type = TEXTURE_TYPE_2D;
		samples = TEXTURE_SAMPLES_1;
		usage_bits = 0;
	}
};
``` 
跟 Texture 相关的接口：
```cpp
	virtual RID texture_create(const TextureFormat &p_format, const TextureView &p_view, const Vector<Vector<uint8_t>> &p_data = Vector<Vector<uint8_t>>()) = 0;
	virtual RID texture_create_shared(const TextureView &p_view, RID p_with_texture) = 0;
	virtual RID texture_create_from_extension(TextureType p_type, DataFormat p_format, TextureSamples p_samples, BitField<RenderingDevice::TextureUsageBits> p_flags, uint64_t p_image, uint64_t p_width, uint64_t p_height, uint64_t p_depth, uint64_t p_layers) = 0;

	virtual RID texture_create_shared_from_slice(const TextureView &p_view, RID p_with_texture, uint32_t p_layer, uint32_t p_mipmap, uint32_t p_mipmaps = 1, TextureSliceType p_slice_type = TEXTURE_SLICE_2D, uint32_t p_layers = 0) = 0;

	virtual Error texture_update(RID p_texture, uint32_t p_layer, const Vector<uint8_t> &p_data, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
	virtual Vector<uint8_t> texture_get_data(RID p_texture, uint32_t p_layer) = 0; // CPU textures will return immediately, while GPU textures will most likely force a flush

	virtual bool texture_is_format_supported_for_usage(DataFormat p_format, BitField<RenderingDevice::TextureUsageBits> p_usage) const = 0;
	virtual bool texture_is_shared(RID p_texture) = 0;
	virtual bool texture_is_valid(RID p_texture) = 0;
	virtual TextureFormat texture_get_format(RID p_texture) = 0;
	virtual Size2i texture_size(RID p_texture) = 0;
	virtual uint64_t texture_get_native_handle(RID p_texture) = 0;

	virtual Error texture_copy(RID p_from_texture, RID p_to_texture, const Vector3 &p_from, const Vector3 &p_to, const Vector3 &p_size, uint32_t p_src_mipmap, uint32_t p_dst_mipmap, uint32_t p_src_layer, uint32_t p_dst_layer, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
	virtual Error texture_clear(RID p_texture, const Color &p_color, uint32_t p_base_mipmap, uint32_t p_mipmaps, uint32_t p_base_layer, uint32_t p_layers, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
	virtual Error texture_resolve_multisample(RID p_from_texture, RID p_to_texture, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
```

## 3、FrameBuffer 
FrameBuffer的创建，定义的基本上都是接口。
```cpp

	struct AttachmentFormat {
		enum { UNUSED_ATTACHMENT = 0xFFFFFFFF };
		DataFormat format;
		TextureSamples samples;
		uint32_t usage_flags;
		AttachmentFormat() {
			format = DATA_FORMAT_R8G8B8A8_UNORM;
			samples = TEXTURE_SAMPLES_1;
			usage_flags = 0;
		}
	};

	typedef int64_t FramebufferFormatID;

	// This ID is warranted to be unique for the same formats, does not need to be freed
	virtual FramebufferFormatID framebuffer_format_create(const Vector<AttachmentFormat> &p_format, uint32_t p_view_count = 1) = 0;
	struct FramebufferPass {
		enum {
			ATTACHMENT_UNUSED = -1
		};
		Vector<int32_t> color_attachments;
		Vector<int32_t> input_attachments;
		Vector<int32_t> resolve_attachments;
		Vector<int32_t> preserve_attachments;
		int32_t depth_attachment = ATTACHMENT_UNUSED;
		int32_t vrs_attachment = ATTACHMENT_UNUSED; // density map for VRS, only used if supported
	};

	virtual FramebufferFormatID framebuffer_format_create_multipass(const Vector<AttachmentFormat> &p_attachments, const Vector<FramebufferPass> &p_passes, uint32_t p_view_count = 1) = 0;
	virtual FramebufferFormatID framebuffer_format_create_empty(TextureSamples p_samples = TEXTURE_SAMPLES_1) = 0;
	virtual TextureSamples framebuffer_format_get_texture_samples(FramebufferFormatID p_format, uint32_t p_pass = 0) = 0;

	virtual RID framebuffer_create(const Vector<RID> &p_texture_attachments, FramebufferFormatID p_format_check = INVALID_ID, uint32_t p_view_count = 1) = 0;
	virtual RID framebuffer_create_multipass(const Vector<RID> &p_texture_attachments, const Vector<FramebufferPass> &p_passes, FramebufferFormatID p_format_check = INVALID_ID, uint32_t p_view_count = 1) = 0;
	virtual RID framebuffer_create_empty(const Size2i &p_size, TextureSamples p_samples = TEXTURE_SAMPLES_1, FramebufferFormatID p_format_check = INVALID_ID) = 0;
	virtual bool framebuffer_is_valid(RID p_framebuffer) const = 0;
	virtual void framebuffer_set_invalidation_callback(RID p_framebuffer, InvalidationCallback p_callback, void *p_userdata) = 0;

	virtual FramebufferFormatID framebuffer_get_format(RID p_framebuffer) = 0;
```

## 4、Sampler
抽象了 VkSample 的重要属性
```cpp
struct SamplerState {
	SamplerFilter mag_filter;
	SamplerFilter min_filter;
	SamplerFilter mip_filter;
	SamplerRepeatMode repeat_u;
	SamplerRepeatMode repeat_v;
	SamplerRepeatMode repeat_w;
	float lod_bias;
	bool use_anisotropy;
	float anisotropy_max;
	bool enable_compare;
	CompareOperator compare_op;
	float min_lod;
	float max_lod;
	SamplerBorderColor border_color;
	bool unnormalized_uvw;

	SamplerState() {
		mag_filter = SAMPLER_FILTER_NEAREST;
		min_filter = SAMPLER_FILTER_NEAREST;
		mip_filter = SAMPLER_FILTER_NEAREST;
		repeat_u = SAMPLER_REPEAT_MODE_CLAMP_TO_EDGE;
		repeat_v = SAMPLER_REPEAT_MODE_CLAMP_TO_EDGE;
		repeat_w = SAMPLER_REPEAT_MODE_CLAMP_TO_EDGE;
		lod_bias = 0;
		use_anisotropy = false;
		anisotropy_max = 1.0;
		enable_compare = false;
		compare_op = COMPARE_OP_ALWAYS;
		min_lod = 0;
		max_lod = 1e20; //something very large should do
		border_color = SAMPLER_BORDER_COLOR_FLOAT_OPAQUE_BLACK;
		unnormalized_uvw = false;
	}
};

virtual RID sampler_create(const SamplerState &p_state) = 0;
virtual bool sampler_is_format_supported_for_filter(DataFormat p_format, SamplerFilter p_sampler_filter) const = 0;
```
## 5、Vertex 和 Index Buffer 
```cpp
struct VertexAttribute {
	uint32_t location; //shader location
	uint32_t offset;
	DataFormat format;
	uint32_t stride;
	VertexFrequency frequency;
	VertexAttribute() {
		location = 0;
		offset = 0;
		stride = 0;
		format = DATA_FORMAT_MAX;
		frequency = VERTEX_FREQUENCY_VERTEX;
	}
};
virtual RID vertex_buffer_create(uint32_t p_size_bytes, const Vector<uint8_t> &p_data = Vector<uint8_t>(), bool p_use_as_storage = false) = 0;

typedef int64_t VertexFormatID;

// This ID is warranted to be unique for the same formats, does not need to be freed
virtual VertexFormatID vertex_format_create(const Vector<VertexAttribute> &p_vertex_formats) = 0;
virtual RID vertex_array_create(uint32_t p_vertex_count, VertexFormatID p_vertex_format, const Vector<RID> &p_src_buffers, const Vector<uint64_t> &p_offsets = Vector<uint64_t>()) = 0;

enum IndexBufferFormat {
	INDEX_BUFFER_FORMAT_UINT16,
	INDEX_BUFFER_FORMAT_UINT32,
};

virtual RID index_buffer_create(uint32_t p_size_indices, IndexBufferFormat p_format, const Vector<uint8_t> &p_data = Vector<uint8_t>(), bool p_use_restart_indices = false) = 0;
virtual RID index_array_create(RID p_index_buffer, uint32_t p_index_offset, uint32_t p_index_count) = 0;
```

## 6、Uniform Buffer
```cpp
virtual RID uniform_buffer_create(uint32_t p_size_bytes, const Vector<uint8_t> &p_data = Vector<uint8_t>()) = 0;
virtual RID storage_buffer_create(uint32_t p_size, const Vector<uint8_t> &p_data = Vector<uint8_t>(), BitField<StorageBufferUsage> p_usage = 0) = 0;
virtual RID texture_buffer_create(uint32_t p_size_elements, DataFormat p_format, const Vector<uint8_t> &p_data = Vector<uint8_t>()) = 0;

struct Uniform {
	UniformType uniform_type;
	int binding; // Binding index as specified in shader.

private:
	// In most cases only one ID is provided per binding, so avoid allocating memory unnecessarily for performance.
	RID id; // If only one is provided, this is used.
	Vector<RID> ids; // If multiple ones are provided, this is used instead.

public:
	_FORCE_INLINE_ uint32_t get_id_count() const {
		return (id.is_valid() ? 1 : ids.size());
	}

	_FORCE_INLINE_ RID get_id(uint32_t p_idx) const {
		if (id.is_valid()) {
			ERR_FAIL_COND_V(p_idx != 0, RID());
			return id;
		} else {
			return ids[p_idx];
		}
	}
	_FORCE_INLINE_ void set_id(uint32_t p_idx, RID p_id) {
		if (id.is_valid()) {
			ERR_FAIL_COND(p_idx != 0);
			id = p_id;
		} else {
			ids.write[p_idx] = p_id;
		}
	}

	_FORCE_INLINE_ void append_id(RID p_id) {
		if (ids.is_empty()) {
			if (id == RID()) {
				id = p_id;
			} else {
				ids.push_back(id);
				ids.push_back(p_id);
				id = RID();
			}
		} else {
			ids.push_back(p_id);
		}
	}

	_FORCE_INLINE_ void clear_ids() {
		id = RID();
		ids.clear();
	}

	_FORCE_INLINE_ Uniform(UniformType p_type, int p_binding, RID p_id) {
		uniform_type = p_type;
		binding = p_binding;
		id = p_id;
	}
	_FORCE_INLINE_ Uniform(UniformType p_type, int p_binding, const Vector<RID> &p_ids) {
		uniform_type = p_type;
		binding = p_binding;
		ids = p_ids;
	}
	_FORCE_INLINE_ Uniform() {
		uniform_type = UNIFORM_TYPE_IMAGE;
		binding = 0;
	}
};

virtual RID uniform_set_create(const Vector<Uniform> &p_uniforms, RID p_shader, uint32_t p_shader_set) = 0;
virtual bool uniform_set_is_valid(RID p_uniform_set) = 0;
virtual void uniform_set_set_invalidation_callback(RID p_uniform_set, InvalidationCallback p_callback, void *p_userdata) = 0;

virtual Error buffer_copy(RID p_src_buffer, RID p_dst_buffer, uint32_t p_src_offset, uint32_t p_dst_offset, uint32_t p_size, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
virtual Error buffer_update(RID p_buffer, uint32_t p_offset, uint32_t p_size, const void *p_data, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
virtual Error buffer_clear(RID p_buffer, uint32_t p_offset, uint32_t p_size, BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;
virtual Vector<uint8_t> buffer_get_data(RID p_buffer, uint32_t p_offset = 0, uint32_t p_size = 0) = 0; // This causes stall, only use to retrieve large buffers for saving.
```
## 7、Shader
```cpp
const Capabilities *get_device_capabilities() const { return &device_capabilities; };

enum Features {
	SUPPORTS_MULTIVIEW,
	SUPPORTS_FSR_HALF_FLOAT,
	SUPPORTS_ATTACHMENT_VRS,
	// If not supported, a fragment shader with only side effets (i.e., writes  to buffers, but doesn't output to attachments), may be optimized down to no-op by the GPU driver.
	SUPPORTS_FRAGMENT_SHADER_WITH_ONLY_SIDE_EFFECTS,
};
virtual bool has_feature(const Features p_feature) const = 0;

virtual Vector<uint8_t> shader_compile_spirv_from_source(ShaderStage p_stage, const String &p_source_code, ShaderLanguage p_language = SHADER_LANGUAGE_GLSL, String *r_error = nullptr, bool p_allow_cache = true);
virtual String shader_get_spirv_cache_key() const;

static void shader_set_compile_to_spirv_function(ShaderCompileToSPIRVFunction p_function);
static void shader_set_spirv_cache_function(ShaderCacheFunction p_function);
static void shader_set_get_cache_key_function(ShaderSPIRVGetCacheKeyFunction p_function);

struct ShaderStageSPIRVData {
	ShaderStage shader_stage;
	Vector<uint8_t> spir_v;

	ShaderStageSPIRVData() {
		shader_stage = SHADER_STAGE_VERTEX;
	}
};

virtual String shader_get_binary_cache_key() const = 0;
virtual Vector<uint8_t> shader_compile_binary_from_spirv(const Vector<ShaderStageSPIRVData> &p_spirv, const String &p_shader_name = "") = 0;

virtual RID shader_create_from_spirv(const Vector<ShaderStageSPIRVData> &p_spirv, const String &p_shader_name = "");
virtual RID shader_create_from_bytecode(const Vector<uint8_t> &p_shader_binary, RID p_placeholder = RID()) = 0;
virtual RID shader_create_placeholder() = 0;

virtual uint64_t shader_get_vertex_input_attribute_mask(RID p_shader) = 0;
```
## 8、Pipeline Constant
```cpp
struct PipelineSpecializationConstant {
	PipelineSpecializationConstantType type;
	uint32_t constant_id;
	union {
		uint32_t int_value;
		float float_value;
		bool bool_value;
	};

	PipelineSpecializationConstant() {
		type = PIPELINE_SPECIALIZATION_CONSTANT_TYPE_BOOL;
		constant_id = 0;
		int_value = 0;
	}
};
```
## 9、Pipeline
```cpp
	virtual bool render_pipeline_is_valid(RID p_pipeline) = 0;
	virtual RID render_pipeline_create(RID p_shader, FramebufferFormatID p_framebuffer_format, VertexFormatID p_vertex_format, RenderPrimitive p_render_primitive, const PipelineRasterizationState &p_rasterization_state, const PipelineMultisampleState &p_multisample_state, const PipelineDepthStencilState &p_depth_stencil_state, const PipelineColorBlendState &p_blend_state, BitField<PipelineDynamicStateFlags> p_dynamic_state_flags = 0, uint32_t p_for_render_pass = 0, const Vector<PipelineSpecializationConstant> &p_specialization_constants = Vector<PipelineSpecializationConstant>()) = 0;

	/**************************/
	/**** COMPUTE PIPELINE ****/
	/**************************/

	virtual RID compute_pipeline_create(RID p_shader, const Vector<PipelineSpecializationConstant> &p_specialization_constants = Vector<PipelineSpecializationConstant>()) = 0;
	virtual bool compute_pipeline_is_valid(RID p_pipeline) = 0;
```
## 10、DrawList
```cpp
virtual DrawListID draw_list_begin_for_screen(DisplayServer::WindowID p_screen = 0, const Color &p_clear_color = Color()) = 0;
virtual DrawListID draw_list_begin(RID p_framebuffer, InitialAction p_initial_color_action, FinalAction p_final_color_action, InitialAction p_initial_depth_action, FinalAction p_final_depth_action, const Vector<Color> &p_clear_color_values = Vector<Color>(), float p_clear_depth = 1.0, uint32_t p_clear_stencil = 0, const Rect2 &p_region = Rect2(), const Vector<RID> &p_storage_textures = Vector<RID>()) = 0;
virtual Error draw_list_begin_split(RID p_framebuffer, uint32_t p_splits, DrawListID *r_split_ids, InitialAction p_initial_color_action, FinalAction p_final_color_action, InitialAction p_initial_depth_action, FinalAction p_final_depth_action, const Vector<Color> &p_clear_color_values = Vector<Color>(), float p_clear_depth = 1.0, uint32_t p_clear_stencil = 0, const Rect2 &p_region = Rect2(), const Vector<RID> &p_storage_textures = Vector<RID>()) = 0;

virtual void draw_list_set_blend_constants(DrawListID p_list, const Color &p_color) = 0;
virtual void draw_list_bind_render_pipeline(DrawListID p_list, RID p_render_pipeline) = 0;
virtual void draw_list_bind_uniform_set(DrawListID p_list, RID p_uniform_set, uint32_t p_index) = 0;
virtual void draw_list_bind_vertex_array(DrawListID p_list, RID p_vertex_array) = 0;
virtual void draw_list_bind_index_array(DrawListID p_list, RID p_index_array) = 0;
virtual void draw_list_set_line_width(DrawListID p_list, float p_width) = 0;
virtual void draw_list_set_push_constant(DrawListID p_list, const void *p_data, uint32_t p_data_size) = 0;

virtual void draw_list_draw(DrawListID p_list, bool p_use_indices, uint32_t p_instances = 1, uint32_t p_procedural_vertices = 0) = 0;

virtual void draw_list_enable_scissor(DrawListID p_list, const Rect2 &p_rect) = 0;
virtual void draw_list_disable_scissor(DrawListID p_list) = 0;

virtual uint32_t draw_list_get_current_pass() = 0;
virtual DrawListID draw_list_switch_to_next_pass() = 0;
virtual Error draw_list_switch_to_next_pass_split(uint32_t p_splits, DrawListID *r_split_ids) = 0;

virtual void draw_list_end(BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;

/***********************/
/**** COMPUTE LISTS ****/
/***********************/

typedef int64_t ComputeListID;

virtual ComputeListID compute_list_begin(bool p_allow_draw_overlap = false) = 0;
virtual void compute_list_bind_compute_pipeline(ComputeListID p_list, RID p_compute_pipeline) = 0;
virtual void compute_list_bind_uniform_set(ComputeListID p_list, RID p_uniform_set, uint32_t p_index) = 0;
virtual void compute_list_set_push_constant(ComputeListID p_list, const void *p_data, uint32_t p_data_size) = 0;
virtual void compute_list_dispatch(ComputeListID p_list, uint32_t p_x_groups, uint32_t p_y_groups, uint32_t p_z_groups) = 0;
virtual void compute_list_dispatch_threads(ComputeListID p_list, uint32_t p_x_threads, uint32_t p_y_threads, uint32_t p_z_threads) = 0;
virtual void compute_list_dispatch_indirect(ComputeListID p_list, RID p_buffer, uint32_t p_offset) = 0;
virtual void compute_list_add_barrier(ComputeListID p_list) = 0;

virtual void compute_list_end(BitField<BarrierMask> p_post_barrier = BARRIER_MASK_ALL_BARRIERS) = 0;

virtual void barrier(BitField<BarrierMask> p_from = BARRIER_MASK_ALL_BARRIERS, BitField<BarrierMask> p_to = BARRIER_MASK_ALL_BARRIERS) = 0;
virtual void full_barrier() = 0;
```