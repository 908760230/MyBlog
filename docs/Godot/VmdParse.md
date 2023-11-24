---
title: Vmd 文件解析
comments: true
---

# Vmd 文件解析
MikuMikuDance vmd 文件解析代码，用GDScript 编写

## 代码
``` python
class_name VMD
var version:int
var model_name

var bones:Array
var morphs:Array
var cameras:Array 

func parse_file(path):
	var pos:int = 0
	var value = FileAccess.file_exists(path)
	var	file = FileAccess.get_file_as_bytes(path)
	var head_string = file.slice(0,30).get_string_from_ascii()
	pos += 30
	if head_string == "Vocaloid Motion Data file":
		version = 1
		model_name = file.slice(pos,10).get_string_from_utf8()
		pos += 10
	elif  head_string  == "Vocaloid Motion Data 0002":
		version = 2
		model_name = file.slice(pos,20).get_string_from_utf8()
		pos += 20
	
	pos = parse_bone(file, pos)
	pos = parse_morph(file, pos)
	pos = parse_camera(file,pos)
	
	print("version: ", version)
	print("model_name ", model_name)

	
# every bone has 111 bytes
func parse_bone(file,pos):
	var bone_record = file.decode_u32(pos)
	pos += 4
	for index in range(bone_record):

		var bone_name = file.slice(pos,15).get_string_from_utf8()

		print("bone name: ", bone_name)
		pos += 15
		var frame_time:int = file.decode_32(pos)
		pos += 4
		
		var tx: float = file.decode_float(pos)
		pos += 4
		var ty: float = file.decode_float(pos)
		pos += 4
		var tz: float = file.decode_float(pos)
		pos += 4
		
		var rx: float = file.decode_float(pos)
		pos += 4
		var ry: float = file.decode_float(pos)
		pos += 4
		var rz: float = file.decode_float(pos)
		pos += 4
		var rw: float = file.decode_float(pos)
		pos += 4
		
		var txc1: int = file.decode_32(pos)
		pos += 4
		var txc2: int = file.decode_32(pos)
		pos += 4
		var txc3: int = file.decode_32(pos)
		pos += 4
		var txc4: int = file.decode_32(pos)
		pos += 4
		
		var tyc1: int = file.decode_32(pos)
		pos += 4
		var tyc2: int = file.decode_32(pos)
		pos += 4
		var tyc3: int = file.decode_32(pos)
		pos += 4
		var tyc4: int = file.decode_32(pos)
		pos += 4
		
		var tzc1: int = file.decode_32(pos)
		pos += 4
		var tzc2: int = file.decode_32(pos)
		pos += 4
		var tzc3: int = file.decode_32(pos)
		pos += 4
		var tzc4: int = file.decode_32(pos)
		pos += 4
		
		var rc1: int = file.decode_32(pos)
		pos += 4
		var rc2: int = file.decode_32(pos)
		pos += 4
		var rc3: int = file.decode_32(pos)
		pos += 4
		var rc4: int = file.decode_32(pos)
		pos += 4
		
		var bone_data = [bone_name,frame_time,tx,ty,tz,rx,ry,rz,rw,txc1,txc2,txc3,txc4,tyc1,tyc2,tyc3,tyc4,tzc1,tzc2,tzc3,tzc4,rc1,rc2,rc3,rc4]
		bones.append_array(bone_data)
	
	return pos

# morph has 23 bytes	
func parse_morph(file, pos):
	var number:int = file.decode_u32(pos)
	print("morph number: ", number)
	pos += 4
	for index in range(number):
		var name = file.slice(pos, 15).get_string_from_utf8()
		pos += 15
		var frame_time:int = file.decode_32(pos)
		pos += 4
		var weight: float = file.decode_float(pos +1)
		pos += 4
		var morph_array = [name, frame_time, weight]
		morphs.append_array(morph_array)
	return pos
	
# camera has 61 bytes	
func parse_camera(file, pos):
	var number: int = file.decode_u32(pos)
	print("camera number: ", number)
	pos += 4
	
	for index in range(number):
		var frame_time:int = file.decode_s32(pos)
		pos += 4
		
		var distance:float = file.decode_float(pos)
		pos += 4
		
		var px: float = file.decode_float(pos)
		pos += 4
		var py: float = file.decode_float(pos)
		pos += 4
		var pz: float = file.decode_float(pos)
		pos += 4
		
		var rx: float = file.decode_float(pos)
		pos += 4
		var ry: float = file.decode_float(pos)
		pos += 4
		var rz: float = file.decode_float(pos)
		pos += 4
		
		var txc1: int = file.decode_s8(pos)
		pos += 1
		var txc2: int = file.decode_s8(pos)
		pos += 1
		var txc3: int = file.decode_s8(pos)
		pos += 1
		var txc4: int = file.decode_s8(pos)
		pos += 1
		
		var tyc1: int = file.decode_s8(pos)
		pos += 1
		var tyc2: int = file.decode_s8(pos)
		pos += 1
		var tyc3: int = file.decode_s8(pos)
		pos += 1
		var tyc4: int = file.decode_s8(pos)
		pos += 1
		
		var tzc1: int = file.decode_s8(pos)
		pos += 1
		var tzc2: int = file.decode_s8(pos)
		pos += 1
		var tzc3: int = file.decode_s8(pos)
		pos += 1
		var tzc4: int = file.decode_s8(pos)
		pos += 1
		
		var qc1: int = file.decode_s8(pos)
		pos += 1
		var qc2: int = file.decode_s8(pos)
		pos += 1
		var qc3: int = file.decode_s8(pos)
		pos += 1
		var qc4: int = file.decode_s8(pos)
		pos += 1
		
		var dc1: int = file.decode_s8(pos)
		pos += 1
		var dc2: int = file.decode_s8(pos)
		pos += 1
		var dc3: int = file.decode_s8(pos)
		pos += 1
		var dc4: int = file.decode_s8(pos)
		pos += 1
		
		var vc1: int = file.decode_s8(pos)
		pos += 1
		var vc2: int = file.decode_s8(pos)
		pos += 1
		var vc3: int = file.decode_s8(pos)
		pos += 1
		var vc4: int = file.decode_s8(pos)
		pos += 1
		
		var view__angle = file.decode_float(pos)
		pos += 4
		
		var orthgraphic = file.decode_s8(pos)
		pos += 1
		
		var camera_data = [frame_time, distance,px,py,pz,rx,ry,rz,txc1,txc2,txc3,txc4,tyc1,tyc2,tyc3,tyc4,tzc1,tzc2,tzc3,tzc4,qc1,qc2,qc3,qc4,dc1,dc2,dc3,dc4,vc1,vc2,vc3,vc4]
		cameras.append_array(camera_data)
	return pos

```