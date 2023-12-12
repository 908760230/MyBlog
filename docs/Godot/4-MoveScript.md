---
title: xz平面方向移动
comments: true
---
# xz平面方向移动
GDscript 代码, 
```python
extend CharacterBody3D

var input_dir := Input.get_vector("move_left","move_right","move_forward","move_down");
var direction := (transform.basis * Vector3(input_dir.x,0,input_dir.y)).normalized()
if direction:
	velocity.x = direction.x * speed
	velocity.z = direction.z * speed
else:
	velocity.x = move_toward(velocity.x,0,speed)
	velocity.z = move_toward(velocity.z,0,speed)
move_and_slide()
```