---
title: 3D 描边的 Shader 代码
comments: true
---

# 3D 描边的 Shader 代码


这段代码是一个用于在 Godot 中实现像素艺术轮廓高光后处理效果的 GLSL 着色器。它使用了深度纹理、屏幕纹理和法线纹理作为输入，并生成一个带有阴影颜色和透明度的新帧。

下面是该着色器的主要部分：

getDepth 函数：计算给定屏幕空间坐标处的深度值。

fragment 函数：主要的片段着色函数，它获取当前像素的深度信息、屏幕颜色和法线数据，然后遍历其周围的像素来计算边缘厚度和阴影颜色。

遍历周围像素时，比较当前像素和周围像素的深度差异，并将差异乘以一个系数得到最终的阴影厚度。同时，如果周围像素的深度比当前像素的深度小，则将其颜色设置为新的阴影颜色，并计算相应的法线差异。
最后，根据计算出的阴影厚度、颜色和透明度更新输出的颜色和深度值。


``` cpp
shader_type spatial;
render_mode unshaded, depth_draw_opaque, depth_prepass_alpha;

// Inspired by https://godotshaders.com/shader/3d-pixel-art-outline-highlight-post-processing-shader/

uniform sampler2D DEPTH_TEXTURE : hint_depth_texture, filter_linear_mipmap;
uniform sampler2D SCREEN_TEXTURE : hint_screen_texture, filter_linear_mipmap;
uniform sampler2D NORMAL_TEXTURE : hint_normal_roughness_texture, filter_nearest;


uniform vec3 shadow_color : source_color = vec3(0.0);
uniform float shadow_thickness = 2.0;

vec2 getDepth(vec2 screen_uv, sampler2D depth_texture, mat4 inv_projection_matrix){
	float raw_depth = texture(depth_texture, screen_uv)[0];                              // 从深度纹理中获取当前像素的深度值
	vec3 normalized_device_coordinates = vec3(screen_uv * 2.0 - 1.0, raw_depth);         // 将屏幕空间坐标转换为归一化设备坐标
    vec4 view_space = inv_projection_matrix * vec4(normalized_device_coordinates, 1.0);	 // 将归一化设备坐标转换为视图空间坐标
	view_space.xyz /= view_space.w;	                                                     // 视图空间坐标除以 w 分量得到齐次裁剪坐标
	return vec2(-view_space.z, raw_depth);
}


void fragment() {
	vec2 e = vec2(1./VIEWPORT_SIZE.xy)*1.0;   // 计算用于遍历周围像素的单位向量

	float depth_diff = 0.0;
	float neg_depth_diff = .5;
	
	vec2 depth_data = getDepth(SCREEN_UV, DEPTH_TEXTURE, INV_PROJECTION_MATRIX);  // 获取当前像素的深度数据
	float depth = depth_data.x;
	vec3 color = texture(SCREEN_TEXTURE, SCREEN_UV).rgb;  // 获取当前像素的颜色
	vec3 c = vec3(0.0);
	
	vec2 min_depth_data = depth_data;
	float min_depth = 9999999.9;
	

	vec3 normal = texture(NORMAL_TEXTURE, SCREEN_UV).rgb * 2.0 - 1.0; // 将法线向量的范围扩展到 -1 到 1
	
	for (float x = -shadow_thickness; x <= shadow_thickness;x += 1.0){
		for (float y = -shadow_thickness; y <= shadow_thickness; y += 1.0){
            // 原点 和  距离大于阴影厚度的像素 无需处理
			if ((x == 0.0 && y == 0.0) || (shadow_thickness*shadow_thickness < (x*x + y*y))){
				continue;
			}
			
            // 沿着 x 和 y 方向分别偏移一个像素的距离
			vec2 du_data = getDepth(SCREEN_UV+1.0*vec2(x, y)*e, DEPTH_TEXTURE, INV_PROJECTION_MATRIX);
            // 沿着 x 和 y 方向分别偏移半个像素的距离
			vec2 dd_data = getDepth(SCREEN_UV+0.5*vec2(x, y)*e, DEPTH_TEXTURE, INV_PROJECTION_MATRIX);
			
			float du = du_data.x;
			float dd = dd_data.x;
			
            // 深度的差异值
			float dd_diff = clamp(abs((depth - dd) - (dd - du)), 0.0, 1.0);
            // 得到阴影强度
			float val = clamp(abs(depth - du), 0., 1.)/(x*x + y*y)*dd_diff*dd_diff*5000.0;
			
			val = clamp(val, 0.0, 1.0);

            //将多个像素的阴影效果累积起来，从而实现阴影渐变效果
			depth_diff += val;

			if (du < min_depth){
				min_depth = du;  // 保存最小深度值
				min_depth_data = du_data;
				c = texture(SCREEN_TEXTURE, SCREEN_UV+vec2(x, y)*e).rgb;
				
                //将色彩限制在 0-1 范围内
				c *= clamp(0.5+ 0.5*dot(normalize(vec2(x, y)), (vec2(0.0, 1.0))), 0.0, 1.0);
				
			}
			
			vec3 nu = texture(NORMAL_TEXTURE, SCREEN_UV+vec2(x, y)*e).rgb * 2.0 - 1.0;
			
			depth_diff += (1.0-abs(dot(nu, normal)))/max(min(dd, depth), 2.0);
		}
	}


	depth_diff = smoothstep(0.2, 0.3, depth_diff);

	vec3 final = c*shadow_color;
	ALBEDO = final;

	float alpha_mask = depth_diff;
	DEPTH = min_depth_data.y*alpha_mask + depth_data.y*(1.0-alpha_mask);
	ALPHA = clamp((alpha_mask) * 5., 0., 1.);

}
```