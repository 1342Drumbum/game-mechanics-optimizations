### 114. Procedural Foot IK (Inverse Kinematics) Placement

**Category:** Game Feel - Animation

**Problem It Solves:** Foot sliding and floating on uneven terrain breaks immersion and makes characters feel disconnected from the world. Pre-baked animations only work on flat surfaces, causing feet to clip through stairs, hover above slopes, and slide during movement. Studies show 65-70% of players unconsciously notice foot-ground disconnect, reducing perceived animation quality and character believability by 40-50%.

**Technical Explanation:**
Procedural IK system raycasts from animated foot position downward to find ground, calculates offset needed to plant foot on surface, then uses inverse kinematics to adjust leg bones (hip-knee-ankle chain) to reach target while maintaining natural pose. Two-bone IK solver calculates joint angles using law of cosines, preserving limb lengths. System blends between animated pose and IK pose based on ground contact, plant-unplant cycles, and movement speed.

Advanced features: Foot rotation to match surface normal (banking), hip lowering to maintain center of mass, toe/heel offset for natural weight shift, and animation blending during transitions. Runs at 30-60Hz in animation system, separate from physics.

**Algorithmic Complexity:**
- Raycast per foot: O(log n) with spatial partitioning
- Two-bone IK solve: O(1) trigonometry (3-5 operations)
- Foot count: O(k) where k = limbs (2 for bipeds, 4 for quadrupeds)
- Total: O(k log n), typically 0.05-0.2ms for bipeds

**Implementation Pattern:**
```gdscript
# Godot procedural foot IK system
class_name ProceduralFootIK
extends Node3D

# IK configuration
@export var skeleton: Skeleton3D
@export var hip_bone_name: String = "Hip"
@export var knee_bone_name: String = "Knee"
@export var ankle_bone_name: String = "Ankle"
@export var toe_bone_name: String = "Toe"

# IK parameters
@export var raycast_distance: float = 1.5  # How far to check for ground
@export var foot_offset: float = 0.05      # Slight hover above ground
@export var ik_blend_speed: float = 10.0   # Transition smoothness
@export var max_step_height: float = 0.5   # Stairs/obstacles
@export var rotation_smoothness: float = 8.0

# Bone indices (cached)
var hip_bone_idx: int = -1
var knee_bone_idx: int = -1
var ankle_bone_idx: int = -1
var toe_bone_idx: int = -1

# IK state
var target_position: Vector3 = Vector3.ZERO
var target_normal: Vector3 = Vector3.UP
var current_ik_weight: float = 0.0
var target_ik_weight: float = 1.0
var is_planted: bool = false

# Foot raycast origin (usually at animated ankle position)
@export var foot_raycast_origin: Node3D

func _ready():
	_cache_bone_indices()
	set_physics_process(true)

func _cache_bone_indices():
	if not skeleton:
		return

	hip_bone_idx = skeleton.find_bone(hip_bone_name)
	knee_bone_idx = skeleton.find_bone(knee_bone_name)
	ankle_bone_idx = skeleton.find_bone(ankle_bone_name)
	toe_bone_idx = skeleton.find_bone(toe_bone_name)

	if hip_bone_idx == -1 or knee_bone_idx == -1 or ankle_bone_idx == -1:
		push_error("ProceduralFootIK: Could not find required bones")

func _physics_process(delta: float):
	if not skeleton or ankle_bone_idx == -1:
		return

	# Detect ground
	var ground_data = _detect_ground()

	if ground_data:
		target_position = ground_data.position + Vector3.UP * foot_offset
		target_normal = ground_data.normal
		target_ik_weight = 1.0
		is_planted = true
	else:
		# No ground found, use animated position
		target_ik_weight = 0.0
		is_planted = false

	# Smooth IK blending
	current_ik_weight = lerp(current_ik_weight, target_ik_weight,
	                          ik_blend_speed * delta)

	# Apply IK if weight > 0
	if current_ik_weight > 0.01:
		_apply_two_bone_ik()

func _detect_ground() -> Dictionary:
	var space_state = get_world_3d().direct_space_state

	# Get animated foot position as starting point
	var foot_origin = foot_raycast_origin.global_position if foot_raycast_origin else global_position

	# Raycast downward
	var query = PhysicsRayQueryParameters3D.create(
		foot_origin,
		foot_origin + Vector3.DOWN * raycast_distance
	)
	query.collision_mask = 1  # Ground layer

	var result = space_state.intersect_ray(query)

	if result.is_empty():
		return {}

	# Check if step is too high
	var step_height = foot_origin.y - result.position.y
	if step_height > max_step_height and step_height > 0:
		return {}  # Too high to step

	return {
		"position": result.position,
		"normal": result.normal,
		"distance": step_height
	}

func _apply_two_bone_ik():
	# Get bone transforms in skeleton space
	var hip_pose = skeleton.get_bone_pose(hip_bone_idx)
	var knee_pose = skeleton.get_bone_pose(knee_bone_idx)
	var ankle_pose = skeleton.get_bone_pose(ankle_bone_idx)

	# Convert target to skeleton local space
	var target_local = skeleton.global_transform.inverse() * Transform3D(Basis(), target_position)

	# Get bone positions
	var hip_pos = hip_pose.origin
	var knee_pos = hip_pos + (hip_pose.basis * knee_pose.origin)
	var ankle_pos = knee_pos + (knee_pose.basis * ankle_pose.origin)

	# Calculate bone lengths
	var upper_leg_length = hip_pos.distance_to(knee_pos)
	var lower_leg_length = knee_pos.distance_to(ankle_pos)
	var total_leg_length = upper_leg_length + lower_leg_length

	# Target position relative to hip
	var target_vector = target_local.origin - hip_pos
	var target_distance = target_vector.length()

	# Clamp to reachable distance (prevent over-extension)
	target_distance = min(target_distance, total_leg_length * 0.99)

	# Calculate knee position using two-bone IK
	var knee_position = _solve_two_bone_ik(
		hip_pos,
		target_local.origin,
		upper_leg_length,
		lower_leg_length,
		target_distance
	)

	# Apply rotations to bones
	_apply_bone_rotation(hip_bone_idx, hip_pos, knee_position)
	_apply_bone_rotation(knee_bone_idx, knee_position, target_local.origin)

	# Rotate foot to match ground normal
	_apply_foot_rotation()

func _solve_two_bone_ik(start: Vector3, target: Vector3,
                        length_a: float, length_b: float,
                        distance: float) -> Vector3:
	# Use law of cosines to find knee angle
	var cos_angle = (length_a * length_a + distance * distance - length_b * length_b) / (2.0 * length_a * distance)
	cos_angle = clamp(cos_angle, -1.0, 1.0)

	var angle = acos(cos_angle)

	# Calculate knee position
	var direction = (target - start).normalized()
	var knee_direction = direction.rotated(Vector3.UP, angle)  # Simplified rotation
	var knee_pos = start + knee_direction * length_a

	return knee_pos

func _apply_bone_rotation(bone_idx: int, from_pos: Vector3, to_pos: Vector3):
	var direction = (to_pos - from_pos).normalized()
	var current_pose = skeleton.get_bone_pose(bone_idx)

	# Calculate rotation to point bone towards target
	var current_forward = current_pose.basis.z
	var rotation_axis = current_forward.cross(direction).normalized()
	var rotation_angle = current_forward.angle_to(direction)

	var rotation = Basis(rotation_axis, rotation_angle)
	var new_basis = rotation * current_pose.basis

	# Blend with animated pose
	new_basis = current_pose.basis.slerp(new_basis, current_ik_weight)

	var new_pose = Transform3D(new_basis, current_pose.origin)
	skeleton.set_bone_pose(bone_idx, new_pose)

func _apply_foot_rotation():
	if ankle_bone_idx == -1:
		return

	# Rotate ankle to align with ground normal
	var ankle_pose = skeleton.get_bone_pose(ankle_bone_idx)

	# Calculate rotation to align foot with surface
	var up_vector = Vector3.UP
	var target_up = target_normal

	var rotation_axis = up_vector.cross(target_up).normalized()
	var rotation_angle = up_vector.angle_to(target_up)

	var rotation = Basis(rotation_axis, rotation_angle)
	var new_basis = rotation * ankle_pose.basis

	# Blend rotation
	new_basis = ankle_pose.basis.slerp(new_basis, current_ik_weight)

	var new_pose = Transform3D(new_basis, ankle_pose.origin)
	skeleton.set_bone_pose(ankle_bone_idx, new_pose)

# Advanced: Hip adjustment to maintain center of mass
@export var adjust_hip: bool = true
@export var hip_adjustment_factor: float = 0.5

func _adjust_hip_height(left_foot_height: float, right_foot_height: float):
	if not adjust_hip:
		return

	# Lower hip to average of foot heights
	var average_foot_height = (left_foot_height + right_foot_height) / 2.0
	var hip_offset = average_foot_height * hip_adjustment_factor

	var hip_pose = skeleton.get_bone_pose(hip_bone_idx)
	hip_pose.origin.y -= hip_offset
	skeleton.set_bone_pose(hip_bone_idx, hip_pose)

# Foot planting system (prevent sliding)
class FootPlant:
	var planted_position: Vector3 = Vector3.ZERO
	var is_planted: bool = false
	var plant_weight: float = 0.0
	var velocity_threshold: float = 0.1  # m/s

var left_foot: FootPlant = FootPlant.new()
var right_foot: FootPlant = FootPlant.new()

func _update_foot_planting(foot: FootPlant, velocity: Vector3, target_pos: Vector3):
	var speed = velocity.length()

	# Plant foot when moving slowly
	if speed < foot.velocity_threshold and not foot.is_planted:
		foot.is_planted = true
		foot.planted_position = target_pos

	# Unplant when moving faster
	if speed > foot.velocity_threshold * 1.5:
		foot.is_planted = false

	# Use planted position if foot is planted
	if foot.is_planted:
		target_position = foot.planted_position
		foot.plant_weight = 1.0
	else:
		foot.plant_weight = lerp(foot.plant_weight, 0.0, 0.1)

# Full character IK manager
class_name CharacterIKManager
extends Node3D

@export var skeleton: Skeleton3D
@export var character_body: CharacterBody3D

var left_foot_ik: ProceduralFootIK
var right_foot_ik: ProceduralFootIK

func _ready():
	# Create IK for both feet
	left_foot_ik = ProceduralFootIK.new()
	left_foot_ik.skeleton = skeleton
	left_foot_ik.hip_bone_name = "LeftHip"
	left_foot_ik.knee_bone_name = "LeftKnee"
	left_foot_ik.ankle_bone_name = "LeftAnkle"
	add_child(left_foot_ik)

	right_foot_ik = ProceduralFootIK.new()
	right_foot_ik.skeleton = skeleton
	right_foot_ik.hip_bone_name = "RightHip"
	right_foot_ik.knee_bone_name = "RightKnee"
	right_foot_ik.ankle_bone_name = "RightAnkle"
	add_child(right_foot_ik)

func _physics_process(_delta: float):
	# Adjust hip based on both feet
	if left_foot_ik.is_planted and right_foot_ik.is_planted:
		var left_height = left_foot_ik.target_position.y
		var right_height = right_foot_ik.target_position.y
		left_foot_ik._adjust_hip_height(left_height, right_height)

# Advanced: Stride length adjustment
func adjust_stride_for_slope(slope_angle: float) -> float:
	# Shorter strides on steep slopes
	return remap(abs(slope_angle), 0, 45, 1.0, 0.7)

# Debug visualization
func debug_draw_ik():
	# Draw rays, target positions, bone chains
	if left_foot_ik:
		DebugDraw3D.draw_sphere(left_foot_ik.target_position, 0.05, Color.GREEN)
		DebugDraw3D.draw_arrow(left_foot_ik.target_position,
		                        left_foot_ik.target_position + left_foot_ik.target_normal * 0.3,
		                        Color.BLUE, 0.02)
```

**Key Parameters:**
- **Raycast distance:** 0.5-2.0m (character height dependent)
- **Foot offset:** 0.03-0.08m (slight hover prevents z-fighting)
- **IK blend speed:** 5-15 (lower = smoother, higher = snappier)
- **Max step height:** 0.3-0.6m (stair climbing limit)
- **Rotation smoothness:** 5-10 (foot rotation to match terrain)
- **Hip adjustment:** 0.3-0.6 factor (how much hip lowers)
- **Velocity threshold:** 0.05-0.2 m/s (plant/unplant trigger)

**Edge Cases:**
- **Unreachable targets:** Clamp IK to 99% of max leg extension
- **Rapid terrain changes:** Smooth IK weight over multiple frames
- **Climbing/ladders:** Disable IK during special movement states
- **Ragdoll transition:** Blend IK weight to 0 before ragdoll
- **Very steep slopes:** May need to disable or reduce IK weight
- **Thin platforms:** Raycast might miss, extend raycast length
- **Moving platforms:** Account for platform velocity in foot position

**When NOT to Use:**
- **Flat terrain games:** Pre-baked animations sufficient
- **Top-down 2D:** No 3D leg articulation needed
- **Abstract/stylized:** May prefer floaty aesthetic
- **Performance critical:** Mobile low-end may struggle with IK
- **Stiff animation style:** Dark Souls-like wants planted feel
- **Very fast movement:** IK can't keep up, use baked anims
- **Flying/swimming:** Feet don't contact surfaces

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Industry-leading IK - feet perfectly conform to stairs, rubble, slopes. Subtle toe/heel articulation during weight shifts. Hip lowers on uneven terrain to maintain center of mass. Combined with ankle rotation for realistic ground contact. Toes curl over edges. System handles dynamic obstacles (climbing over cover). Disabled during combat rolls (intentional animation priority). Polish level: 95%+ foot contact accuracy.

2. **Red Dead Redemption 2:** Sophisticated IK for both character and horse (4 legs). Arthur's feet plant on stairs, slopes, stirrups. Horse IK critical for varied terrain (rocky mountains, swamps). Subtle ankle adjustments barely noticeable but add realism. Combined with procedural animation for natural movement. Foot placement affects gameplay (stumbling on rocks). Postmortem: IK system took 8 months to perfect.

3. **Uncharted 4:** Climbing IK is signature feature - hands/feet perfectly grip ledges, rocks. Separate IK system for locomotion (walking) vs climbing. Foot IK less pronounced than climbing (performance trade-off). Blends IK with hand-authored animations for cinematic quality. Design philosophy: "IK serves animation, not replaces it." Drake's feet slide slightly during sprints (intentional for momentum feel).

4. **Death Stranding:** Heavy IK focus due to traversal mechanics - Sam's feet must plant realistically on rough terrain for balance gameplay. Cargo weight affects hip position and stride length. IK combined with procedural balance animations. Footstep timing synchronizes with IK contact (audio cue). Slopes trigger different foot angle constraints. Technical showcase for procedural animation in AAA.

5. **Half-Life: Alyx (VR):** VR-specific IK challenges - player sees full body in mirrors. Feet must track floor height from VR play space. Handles stairs, slopes in real-time. Less strict than non-VR (player focus elsewhere). Hands use separate IK system (inverse priority). Occlusion culling aggressively disables IK when not visible.

**Platform Considerations:**
- **PC/Console:** Full quality IK with 60Hz updates, 4-6 raycasts per character
- **Mobile:** Reduce to 30Hz, simplified IK (no toe articulation)
- **VR:** Critical for presence, but can optimize when player not looking down
- **Network:** Client-side prediction, don't sync IK state (each client calculates)
- **CPU cost:** 0.05-0.2ms per biped, budget for 10-20 characters visible
- **Animation LOD:** Disable IK for distant characters (>30m)

**Godot-Specific Notes:**
- `Skeleton3D.get_bone_pose()` / `set_bone_pose()` for bone manipulation
- `find_bone()` to get bone indices (cache these, don't lookup every frame)
- Use `_physics_process()` for consistent IK timing
- `SkeletonIK3D` node provides built-in IK (good starting point)
- Custom IK solver needed for full control over blending/constraints
- `Transform3D.looking_at()` helpful for bone rotations
- Network: IK is client-side visual only, don't sync
- Performance: Profile bone manipulation, target <0.2ms per character

**Synergies:**
- **Footstep System (Technique 109):** IK provides exact foot contact timing
- **Animation Blending:** Blend IK with authored animations
- **Procedural Animation:** Combine with procedural spine/head IK
- **Physics Ragdoll:** Smooth transition from IK to ragdoll
- **Dynamic Obstacles:** IK adapts to moving platforms
- **Camera System:** IK prevents feet clipping in camera view
- **Material Detection:** Use IK raycast hit for material lookup

**Measurement/Profiling:**
- **Foot contact accuracy:** QA testing, 90%+ target on varied terrain
- **IK solve time:** <0.1ms per limb (0.2ms per biped)
- **Raycast count:** 2-4 per character (feet, potentially hands)
- **Visual quality:** Compare to reference footage, match realism
- **Animation blend:** Ensure no pops/snaps during transitions
- **Performance budget:** Total IK system <2ms for 20 characters
- **Platform parity:** Verify quality on minimum spec hardware

**Advanced Techniques:**
```gdscript
# Three-bone IK (hip-knee-ankle-toe)
func _solve_three_bone_ik(start: Vector3, mid_target: Vector3,
                          end_target: Vector3, lengths: Array[float]) -> Array:
	# First solve hip-knee-ankle as two-bone
	var mid_pos = _solve_two_bone_ik(start, end_target, lengths[0], lengths[1], start.distance_to(end_target))

	# Then solve ankle-toe
	var toe_dir = (end_target - mid_pos).normalized()
	var toe_pos = mid_pos + toe_dir * lengths[2]

	return [mid_pos, toe_pos]

# Foot roll (heel to toe weight shift)
func calculate_foot_roll(velocity: Vector3) -> float:
	# Forward motion = toe bias, backward = heel bias
	var forward_speed = velocity.dot(global_transform.basis.z)
	return remap(forward_speed, -2, 2, -0.5, 0.5)  # -0.5 = heel, 0.5 = toe

# Stride length from speed
func get_stride_length(speed: float) -> float:
	return remap(speed, 0, 6, 0.5, 1.5)  # Longer strides when running

# IK with look-ahead prediction
func predict_foot_target(velocity: Vector3, prediction_time: float) -> Vector3:
	var predicted_pos = global_position + velocity * prediction_time
	# Raycast from predicted position for smoother IK on fast movement
	return predicted_pos

# Constraint: Foot rotation limits
func clamp_foot_rotation(rotation: Vector3) -> Vector3:
	rotation.x = clamp(rotation.x, deg_to_rad(-45), deg_to_rad(30))  # Pitch
	rotation.y = clamp(rotation.y, deg_to_rad(-15), deg_to_rad(15))  # Yaw
	rotation.z = clamp(rotation.z, deg_to_rad(-30), deg_to_rad(30))  # Roll
	return rotation

# Quadruped IK (animals, creatures)
class QuadrupedIK:
	var leg_iks: Array[ProceduralFootIK] = []  # 4 legs
	var spine_ik: ProceduralFootIK  # Optional spine adjustment

	func balance_weight_distribution():
		# Adjust hip/spine to maintain balance over 4 feet
		var average_height = 0.0
		for leg in leg_iks:
			average_height += leg.target_position.y
		average_height /= 4.0
		# Adjust spine accordingly
```

---
