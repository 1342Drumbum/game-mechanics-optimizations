### 117. Ragdoll Blending

**Category:** Game Feel - Animation

**Problem It Solves:** Instant animated-to-ragdoll transitions create jarring pops, breaking immersion and making deaths feel disconnected from gameplay. Characters that snap from running animation directly to physics simulation exhibit unnatural motion discontinuities. Studies show smooth ragdoll transitions increase perceived animation quality by 60-70% and make combat feel more impactful and responsive.

**Technical Explanation:**
Ragdoll blending gradually transitions animated skeleton from kinematic (animation-driven) to dynamic (physics-driven) state over 0.1-0.5s. System copies bone velocities from animation to physics bodies, blends between animated and simulated poses using weight parameter (0=animated, 1=physics), then fully enables physics simulation after blend completes. Critical: Initialize physics bodies with current animation velocity to avoid velocity discontinuities.

Implementation uses PhysicsBody3D per bone with collision shapes, applies external forces (bullet impact, explosion), gradually increases simulation weight while decreasing animation weight, and maintains partial animation control during blend (e.g., keep spine somewhat rigid for first 0.2s).

**Algorithmic Complexity:**
- Bone velocity calculation: O(n) where n = bones
- Physics body sync: O(n)
- Blend interpolation: O(n)
- Total: O(n), typically 0.1-0.3ms for 30-60 bone skeleton

**Implementation Pattern:**
```gdscript
# Godot ragdoll blending system
class_name RagdollBlender
extends Node3D

# References
@export var skeleton: Skeleton3D
@export var animation_tree: AnimationTree
@export var physical_bone_root: PhysicalBone3D  # Root physical bone

# Blend configuration
@export var blend_duration: float = 0.3        # Transition time
@export var initial_force_multiplier: float = 1.5  # Extra oomph on transition
@export var maintain_partial_control: bool = true  # Keep some bones animated longer

# State
enum RagdollState { ANIMATED, BLENDING, PHYSICS }
var current_state: RagdollState = RagdollState.ANIMATED
var blend_time: float = 0.0
var blend_weight: float = 0.0  # 0 = animated, 1 = physics

# Bone tracking
var physical_bones: Array[PhysicalBone3D] = []
var bone_velocities: Dictionary = {}  # Bone index -> velocity
var previous_bone_positions: Dictionary = {}

# Impact data
var impact_point: Vector3 = Vector3.ZERO
var impact_force: Vector3 = Vector3.ZERO
var impact_bone: int = -1

func _ready():
	_collect_physical_bones()
	_initialize_ragdoll()
	set_physics_process(true)

func _collect_physical_bones():
	if not physical_bone_root:
		return

	# Recursively find all PhysicalBone3D nodes
	_find_physical_bones_recursive(physical_bone_root)

	print("Found %d physical bones" % physical_bones.size())

func _find_physical_bones_recursive(node: Node):
	if node is PhysicalBone3D:
		physical_bones.append(node)

	for child in node.get_children():
		_find_physical_bones_recursive(child)

func _initialize_ragdoll():
	# Start with all physics disabled (purely animated)
	for phys_bone in physical_bones:
		phys_bone.set_physics_process(false)

	# Initialize tracking
	for i in range(skeleton.get_bone_count()):
		previous_bone_positions[i] = skeleton.get_bone_global_pose(i).origin

func _physics_process(delta: float):
	match current_state:
		RagdollState.ANIMATED:
			_update_bone_velocities(delta)

		RagdollState.BLENDING:
			_update_blend(delta)

		RagdollState.PHYSICS:
			pass  # Pure physics, nothing to do

func _update_bone_velocities(delta: float):
	# Track bone velocities for smooth transition
	for i in range(skeleton.get_bone_count()):
		var current_pos = skeleton.get_bone_global_pose(i).origin
		var previous_pos = previous_bone_positions.get(i, current_pos)

		var velocity = (current_pos - previous_pos) / delta
		bone_velocities[i] = velocity
		previous_bone_positions[i] = current_pos

func _update_blend(delta: float):
	blend_time += delta
	blend_weight = clamp(blend_time / blend_duration, 0.0, 1.0)

	# Update physics influence
	for phys_bone in physical_bones:
		# Gradually increase physics weight
		phys_bone.set_physics_process(true)

		# Blend between animated and physics pose
		if maintain_partial_control:
			_apply_partial_animation_control(phys_bone, 1.0 - blend_weight)

	# Complete transition
	if blend_weight >= 1.0:
		_complete_ragdoll_transition()

func _apply_partial_animation_control(phys_bone: PhysicalBone3D, anim_weight: float):
	# Get animated target position
	var bone_idx = phys_bone.get_bone_id()
	var anim_pose = skeleton.get_bone_global_pose(bone_idx)

	# Apply force to pull physics body toward animated pose
	var target_pos = skeleton.global_transform * anim_pose.origin
	var current_pos = phys_bone.global_position

	var force_direction = (target_pos - current_pos).normalized()
	var force_strength = current_pos.distance_to(target_pos) * 100.0 * anim_weight

	if phys_bone is RigidBody3D:
		phys_bone.apply_central_force(force_direction * force_strength)

func _complete_ragdoll_transition():
	current_state = RagdollState.PHYSICS

	# Disable animation system
	if animation_tree:
		animation_tree.active = false

	# Full physics control
	for phys_bone in physical_bones:
		phys_bone.set_physics_process(true)

	ragdoll_transition_complete.emit()

signal ragdoll_transition_complete

# Main API: Trigger ragdoll
func activate_ragdoll(hit_position: Vector3 = Vector3.ZERO,
                      hit_force: Vector3 = Vector3.ZERO,
                      hit_bone_name: String = ""):
	if current_state != RagdollState.ANIMATED:
		return  # Already ragdolling

	# Store impact data
	impact_point = hit_position
	impact_force = hit_force

	if hit_bone_name != "":
		impact_bone = skeleton.find_bone(hit_bone_name)

	# Start blend
	current_state = RagdollState.BLENDING
	blend_time = 0.0

	# Initialize physics bodies with animation velocities
	_initialize_physics_velocities()

	# Apply impact force
	if hit_force.length() > 0.1:
		_apply_impact_force()

	ragdoll_activated.emit()

signal ragdoll_activated

func _initialize_physics_velocities():
	# Copy animation velocities to physics bodies (critical for smoothness)
	for phys_bone in physical_bones:
		var bone_idx = phys_bone.get_bone_id()
		var velocity = bone_velocities.get(bone_idx, Vector3.ZERO)

		if phys_bone is RigidBody3D:
			phys_bone.linear_velocity = velocity * initial_force_multiplier

func _apply_impact_force():
	# Apply force to impact bone (or closest bone)
	var target_bone: PhysicalBone3D = null

	if impact_bone != -1:
		target_bone = _find_physical_bone_for_skeleton_bone(impact_bone)
	else:
		target_bone = _find_closest_physical_bone(impact_point)

	if target_bone and target_bone is RigidBody3D:
		# Apply impulse at impact point
		var force_point = impact_point - target_bone.global_position
		target_bone.apply_impulse(impact_force, force_point)

		# Apply force to nearby bones for wave effect
		_apply_force_wave(target_bone, impact_force * 0.5, 0.5)

func _find_physical_bone_for_skeleton_bone(bone_idx: int) -> PhysicalBone3D:
	for phys_bone in physical_bones:
		if phys_bone.get_bone_id() == bone_idx:
			return phys_bone
	return null

func _find_closest_physical_bone(position: Vector3) -> PhysicalBone3D:
	var closest: PhysicalBone3D = null
	var closest_dist = INF

	for phys_bone in physical_bones:
		var dist = position.distance_to(phys_bone.global_position)
		if dist < closest_dist:
			closest_dist = dist
			closest = phys_bone

	return closest

func _apply_force_wave(origin_bone: PhysicalBone3D, force: Vector3, radius: float):
	# Apply diminishing force to nearby bones
	for phys_bone in physical_bones:
		if phys_bone == origin_bone:
			continue

		var distance = phys_bone.global_position.distance_to(origin_bone.global_position)
		if distance < radius:
			var falloff = 1.0 - (distance / radius)
			if phys_bone is RigidBody3D:
				phys_bone.apply_central_impulse(force * falloff)

# Reverse: Blend from ragdoll back to animation (get-up)
func deactivate_ragdoll(get_up_animation: String = "get_up"):
	if current_state == RagdollState.ANIMATED:
		return

	# Reverse blend (physics -> animation)
	current_state = RagdollState.BLENDING
	blend_time = 0.0

	# Play get-up animation
	if animation_tree:
		animation_tree.active = true
		# Trigger get-up animation
		var state_machine = animation_tree.get("parameters/playback")
		if state_machine:
			state_machine.travel(get_up_animation)

	# Gradually disable physics
	await _blend_to_animation()

	current_state = RagdollState.ANIMATED

func _blend_to_animation():
	var blend_out_duration = 0.5

	for t in range(0, int(blend_out_duration * 60)):  # 60 fps
		var weight = float(t) / (blend_out_duration * 60)

		for phys_bone in physical_bones:
			# Reduce physics influence
			if phys_bone is RigidBody3D:
				phys_bone.linear_damp = lerp(0.0, 10.0, weight)  # Increase damping

		await get_tree().process_frame

	# Disable all physics
	for phys_bone in physical_bones:
		phys_bone.set_physics_process(false)

# Advanced: Partial ragdoll (only specific limbs)
func activate_partial_ragdoll(bone_names: Array[String]):
	# Only ragdoll specific bones (e.g., just arms)
	for bone_name in bone_names:
		var bone_idx = skeleton.find_bone(bone_name)
		var phys_bone = _find_physical_bone_for_skeleton_bone(bone_idx)

		if phys_bone:
			phys_bone.set_physics_process(true)

# Advanced: Powered ragdoll (maintain some animation control)
var powered_ragdoll_strength: float = 0.3  # How much animation influence

func update_powered_ragdoll(delta: float):
	# Apply forces to make ragdoll follow animation
	for phys_bone in physical_bones:
		var bone_idx = phys_bone.get_bone_id()
		var target_pose = skeleton.get_bone_global_pose(bone_idx)
		var target_pos = skeleton.global_transform * target_pose.origin

		var error = target_pos - phys_bone.global_position
		var force = error * 1000.0 * powered_ragdoll_strength

		if phys_bone is RigidBody3D:
			phys_bone.apply_central_force(force)

# Advanced: Context-aware blend duration
func get_adaptive_blend_duration(hit_force: Vector3) -> float:
	var force_magnitude = hit_force.length()

	if force_magnitude > 50.0:
		return 0.1  # Fast transition for heavy hits
	elif force_magnitude > 20.0:
		return 0.2  # Medium
	else:
		return 0.3  # Slow for weak hits

# Example: Character death integration
class_name CharacterDeath
extends Node3D

@export var ragdoll_blender: RagdollBlender
@export var health_system: Node

func _ready():
	health_system.died.connect(_on_character_died)

func _on_character_died(damage_info: Dictionary):
	var hit_pos = damage_info.get("position", global_position)
	var hit_dir = damage_info.get("direction", Vector3.FORWARD)
	var hit_force = damage_info.get("force", 10.0)

	# Calculate force vector
	var force_vector = hit_dir.normalized() * hit_force

	# Activate ragdoll with impact data
	ragdoll_blender.activate_ragdoll(hit_pos, force_vector)

# Advanced: Ragdoll LOD (distance-based quality)
func set_ragdoll_lod(distance: float):
	if distance < 10.0:
		# Full quality
		blend_duration = 0.3
		maintain_partial_control = true
	elif distance < 30.0:
		# Medium quality
		blend_duration = 0.2
		maintain_partial_control = false
	else:
		# Instant transition (not visible enough for blend)
		blend_duration = 0.05
		maintain_partial_control = false

# Advanced: Ragdoll pooling/recycling
class RagdollPool:
	var inactive_ragdolls: Array[Node3D] = []
	var active_ragdolls: Array[Node3D] = []
	var max_active: int = 10

	func activate_ragdoll(character: Node3D):
		active_ragdolls.append(character)

		# Remove oldest if over limit
		if active_ragdolls.size() > max_active:
			var oldest = active_ragdolls[0]
			active_ragdolls.remove_at(0)
			_cleanup_ragdoll(oldest)

	func _cleanup_ragdoll(character: Node3D):
		character.queue_free()  # Or return to pool

# Debug visualization
func debug_draw_ragdoll():
	for phys_bone in physical_bones:
		# Draw bone positions, velocities, forces
		DebugDraw3D.draw_sphere(phys_bone.global_position, 0.05, Color.RED)

		if phys_bone is RigidBody3D:
			var vel = phys_bone.linear_velocity
			DebugDraw3D.draw_arrow(
				phys_bone.global_position,
				phys_bone.global_position + vel * 0.1,
				Color.YELLOW, 0.02
			)

# Performance monitoring
func get_ragdoll_performance_stats() -> Dictionary:
	return {
		"active_physical_bones": physical_bones.size(),
		"blend_weight": blend_weight,
		"state": RagdollState.keys()[current_state],
	}
```

**Key Parameters:**
- **Blend duration:** 0.15-0.4s (fast to smooth)
- **Initial force multiplier:** 1.2-2.0x (extra impact on transition)
- **Partial control:** 0.2-0.4 animation weight during blend
- **Physics damping:** 0.5-2.0 (affects ragdoll settling)
- **Max active ragdolls:** 5-15 (performance limit)
- **Bone collision complexity:** 1-3 primitives per bone (sphere, capsule, box)
- **Get-up blend:** 0.4-0.8s (animation back to control)

**Edge Cases:**
- **Already ragdolled:** Ignore subsequent ragdoll triggers
- **Instant death (headshot):** Skip to full physics immediately (0.05s blend)
- **Underwater:** Different physics parameters (buoyancy)
- **On moving platforms:** Account for platform velocity
- **Ledge/cliff deaths:** Ensure ragdoll clears ledge (extra force)
- **Ragdoll spam (explosions):** Limit total active ragdolls, cull oldest
- **Network sync:** Approximate ragdoll state, don't sync every bone

**When NOT to Use:**
- **2D games:** Simple sprite flip/fade sufficient
- **Stylized/cartoony:** May prefer animated death sequences
- **Top-down distant camera:** Detail not visible enough
- **Performance critical:** Mobile low-end may struggle with physics
- **Competitive shooters:** Some avoid ragdolls for visual clarity
- **Horror games:** May prefer controlled animated deaths for scares

**Examples from Shipped Games:**

1. **Grand Theft Auto V:** Smooth transitions with euphoria-style animation. Characters stumble before full ragdoll (powered ragdoll phase). Context-aware: Car hits = fast transition, gunshots = slower. Ragdoll pool of 15-20 active bodies, oldest fade/remove. Blend duration adapts: 0.1s for explosions, 0.4s for tripping. Technical achievement: 30+ NPCs can be ragdolled simultaneously.

2. **Red Dead Redemption 2:** Industry-leading ragdoll blending - characters maintain balance briefly before falling. Partial ragdoll: Shot arm goes limp while character stays upright. Location-specific reactions (leg shot = stumble, chest = backward fall). Blend varies: 0.2-0.6s based on damage type. Euphoria middleware for natural motion. Performance: ~0.3ms per active ragdoll.

3. **Halo Infinite:** Fast arcade-style transitions (0.15s blend typical). Enemies launch dramatically from explosions (high force multiplier). Partial ragdoll: Elite armor pieces separate on death. Network-friendly: Server determines ragdoll trigger, clients approximate locally. 60fps requirement means aggressive LOD (instant ragdoll >20m). Spartans have more blend time than Grunts (visual priority).

4. **The Last of Us Part II:** Cinematic ragdoll with contextual behaviors. Enemies grab wounds before ragdolling. Blend duration 0.3-0.5s for realism. Partial control during blend (keep spine rigid for 0.2s). Falls from height trigger mid-air ragdoll start. Ragdoll combines with hit reactions (technique layering). Performance budget: 8-10 active ragdolls maximum.

5. **Overgrowth (indie):** Powered ragdoll throughout gameplay - rabbit characters always partially animated. Active ragdoll: AI controls physics through forces (no pure animation). Blend is gradual consciousness loss rather than death. Showcases procedural animation extreme. Open-source ragdoll system studied by many devs. Performance: <0.1ms per character (optimized solver).

**Platform Considerations:**
- **PC/Console:** Full quality, 8-15 active ragdolls, 30-60 bone skeletons
- **Mobile:** Reduce to 3-5 ragdolls, 15-30 bones, instant transitions
- **VR:** Higher priority (player sees bodies close), but performance limited
- **Network:** Client-side ragdoll, sync trigger point + initial force only
- **Physics engine:** Godot's Bullet/Jolt handles 20+ active ragdolls at 60fps
- **Memory:** Each ragdoll ~50-100KB (collision shapes + state)

**Godot-Specific Notes:**
- `PhysicalBone3D` nodes for ragdoll bones (built-in system)
- `Skeleton3D.physical_bones_start_simulation()` for quick activation
- `RigidBody3D.linear_velocity` to initialize physics with animation speed
- `PhysicalBone3D.set_bone_id()` to link to skeleton bone
- Collision shapes: Use `CapsuleShape3D` for limbs, `SphereShape3D` for joints
- `joint_type` property for bone connections (hinge, ball, etc.)
- Network: Use `rpc()` to trigger ragdoll, don't sync transform each frame
- Performance: Profile physics tick, target <2ms for all ragdolls

**Synergies:**
- **Hit Reactions:** Blend from hit reaction to ragdoll smoothly
- **Procedural Animation:** Powered ragdoll uses IK/procedural forces
- **Death Effects:** Blood, sparks, particles sync with ragdoll trigger
- **Audio:** Impact sounds based on bone collision velocity
- **AI System:** AI stops controlling character on ragdoll
- **Respawn System:** Fade ragdoll before respawning character
- **Camera:** Dynamic camera shake based on ragdoll impact force

**Measurement/Profiling:**
- **Transition smoothness:** No visible pops/discontinuities (QA testing)
- **Performance cost:** <0.3ms per active ragdoll target
- **Physics stability:** Ragdolls shouldn't jitter or explode
- **Max concurrent:** Test with 20+ simultaneous ragdolls
- **Memory footprint:** Monitor total ragdoll memory usage
- **Network bandwidth:** Ragdoll triggers should be <100 bytes
- **Visual quality:** Compare to reference games (RDR2, GTAV)

**Advanced Techniques:**
```gdscript
# Staged ragdoll (gradual loss of control)
func activate_staged_ragdoll():
	# Stage 1: Legs only (stumbling)
	activate_partial_ragdoll(["LeftLeg", "RightLeg"])
	await get_tree().create_timer(0.3).timeout

	# Stage 2: Torso
	activate_partial_ragdoll(["Spine", "Chest"])
	await get_tree().create_timer(0.2).timeout

	# Stage 3: Full ragdoll
	activate_ragdoll()

# Ragdoll with animation layers (keep upper body animated)
func activate_lower_body_ragdoll():
	var lower_body_bones = ["Hip", "LeftLeg", "RightLeg", "LeftFoot", "RightFoot"]

	for bone_name in lower_body_bones:
		var bone_idx = skeleton.find_bone(bone_name)
		var phys_bone = _find_physical_bone_for_skeleton_bone(bone_idx)
		if phys_bone:
			phys_bone.set_physics_process(true)

	# Upper body remains animated (shooting while falling)

# Environmental ragdoll (wind, water current affects physics)
func apply_environmental_forces(wind_direction: Vector3, strength: float):
	for phys_bone in physical_bones:
		if phys_bone is RigidBody3D:
			phys_bone.apply_central_force(wind_direction * strength * phys_bone.mass)

# Cinematic ragdoll (scripted death sequences with physics)
func cinematic_death(target_position: Vector3, duration: float):
	# Blend to target position over time using forces
	powered_ragdoll_strength = 0.8
	var start_time = Time.get_ticks_msec()

	while (Time.get_ticks_msec() - start_time) < (duration * 1000):
		update_powered_ragdoll(get_physics_process_delta_time())
		await get_tree().physics_frame

	powered_ragdoll_strength = 0.0

# Ragdoll recovery (wounded but not dead)
func activate_temporary_ragdoll(duration: float):
	activate_ragdoll()
	await get_tree().create_timer(duration).timeout
	deactivate_ragdoll("get_up_struggle")

# Bone-specific physics properties
func configure_bone_physics(bone_name: String, mass: float, friction: float):
	var phys_bone = _find_physical_bone_for_skeleton_bone(skeleton.find_bone(bone_name))
	if phys_bone and phys_bone is RigidBody3D:
		phys_bone.mass = mass
		# Set physics material friction
```

---
