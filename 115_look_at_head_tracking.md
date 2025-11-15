### 115. Look-At / Head Tracking

**Category:** Game Feel - Animation

**Problem It Solves:** Characters with static forward-facing heads feel lifeless and robotic, missing 30-40% of natural human behavior. NPCs that don't acknowledge player presence break immersion and make worlds feel artificial. Eye contact is fundamental to perceived intelligence and social engagement - characters without it register as "dead" or "mannequin-like" in player perception studies.

**Technical Explanation:**
Procedural head/eye tracking system rotates character head/neck bones and eye meshes to look toward target (player, point of interest, enemy). Uses IK rotation constraints to smoothly interpolate from animated pose to look-at pose. System calculates vector from head to target, converts to local space rotation, applies constraints (±90° horizontal, ±45° vertical typical), then blends with animation at configurable weight.

Advanced implementation: Saccadic eye movement (rapid jumps between fixation points), blink timing, attention priority system (danger > player > idle curiosity), smooth weight ramping during state transitions, and spine contribution (subtle torso twist enhances natural look).

**Algorithmic Complexity:**
- Vector calculation: O(1) subtraction + normalize
- Rotation constraint: O(1) clamp operations
- Bone interpolation: O(1) per bone (typically 2-4 bones)
- Total: O(k) where k = bones, <0.01ms per character

**Implementation Pattern:**
```gdscript
# Godot look-at / head tracking system
class_name HeadLookAt
extends Node3D

# References
@export var skeleton: Skeleton3D
@export var head_bone_name: String = "Head"
@export var neck_bone_name: String = "Neck"
@export var spine_bone_name: String = "Spine2"  # Upper spine
@export var left_eye_bone_name: String = "LeftEye"
@export var right_eye_bone_name: String = "RightEye"

# Look-at configuration
@export var look_at_enabled: bool = true
@export var look_at_weight: float = 1.0         # Overall blend weight
@export var head_weight: float = 0.7            # How much head rotates
@export var neck_weight: float = 0.5            # How much neck rotates
@export var spine_weight: float = 0.2           # How much spine rotates
@export var eye_weight: float = 1.0             # How much eyes rotate

# Rotation constraints (degrees)
@export var max_horizontal_angle: float = 90.0  # ±90° left/right
@export var max_vertical_angle: float = 60.0    # ±60° up/down
@export var max_head_pitch: float = 45.0        # Separate head limit

# Smoothing
@export var look_speed: float = 3.0             # Rotation interpolation
@export var return_speed: float = 2.0           # Return to neutral

# State
var current_target: Vector3 = Vector3.ZERO
var has_target: bool = false
var current_look_rotation: Basis = Basis.IDENTITY
var target_look_rotation: Basis = Basis.IDENTITY

# Bone indices (cached)
var head_bone_idx: int = -1
var neck_bone_idx: int = -1
var spine_bone_idx: int = -1
var left_eye_idx: int = -1
var right_eye_idx: int = -1

# Saccade system (realistic eye movement)
var saccade_timer: float = 0.0
var saccade_interval: float = 0.3  # Seconds between eye jumps
var saccade_offset: Vector3 = Vector3.ZERO

# Blink system
var blink_timer: float = 0.0
var blink_interval: float = 4.0    # Seconds between blinks
var is_blinking: bool = false
var blink_duration: float = 0.15

func _ready():
	_cache_bone_indices()
	set_process(true)

func _cache_bone_indices():
	if not skeleton:
		return

	head_bone_idx = skeleton.find_bone(head_bone_name)
	neck_bone_idx = skeleton.find_bone(neck_bone_name)
	spine_bone_idx = skeleton.find_bone(spine_bone_name)
	left_eye_idx = skeleton.find_bone(left_eye_bone_name)
	right_eye_idx = skeleton.find_bone(right_eye_bone_name)

func _process(delta: float):
	if not look_at_enabled or not skeleton:
		return

	# Update saccades and blinks
	_update_saccades(delta)
	_update_blinks(delta)

	if has_target:
		_update_look_at(delta)
	else:
		_return_to_neutral(delta)

# Main look-at function
func set_look_target(target_position: Vector3):
	current_target = target_position
	has_target = true

func clear_look_target():
	has_target = false

func _update_look_at(delta: float):
	# Calculate direction to target (with saccade offset)
	var target_with_saccade = current_target + saccade_offset
	var head_pos = skeleton.global_transform * skeleton.get_bone_global_pose(head_bone_idx).origin
	var look_direction = (target_with_saccade - head_pos).normalized()

	# Convert to local space
	var local_direction = skeleton.global_transform.basis.inverse() * look_direction

	# Calculate target rotation
	target_look_rotation = _calculate_look_rotation(local_direction)

	# Smooth interpolation
	current_look_rotation = current_look_rotation.slerp(
		target_look_rotation,
		look_speed * delta
	)

	# Apply to bones with weights
	_apply_look_rotation()

func _calculate_look_rotation(direction: Vector3) -> Basis:
	# Calculate angles from forward vector
	var horizontal_angle = atan2(direction.x, direction.z)
	var vertical_angle = asin(-direction.y)

	# Apply constraints
	horizontal_angle = clamp(horizontal_angle,
	                         deg_to_rad(-max_horizontal_angle),
	                         deg_to_rad(max_horizontal_angle))
	vertical_angle = clamp(vertical_angle,
	                      deg_to_rad(-max_vertical_angle),
	                      deg_to_rad(max_vertical_angle))

	# Build rotation basis
	var rotation = Basis()
	rotation = rotation.rotated(Vector3.UP, horizontal_angle)
	rotation = rotation.rotated(Vector3.RIGHT, vertical_angle)

	return rotation

func _apply_look_rotation():
	# Apply to spine (subtle torso twist)
	if spine_bone_idx != -1:
		_apply_bone_look(spine_bone_idx, spine_weight * 0.3)

	# Apply to neck
	if neck_bone_idx != -1:
		_apply_bone_look(neck_bone_idx, neck_weight * 0.5)

	# Apply to head
	if head_bone_idx != -1:
		_apply_bone_look(head_bone_idx, head_weight)

	# Apply to eyes (independent movement)
	_apply_eye_look()

func _apply_bone_look(bone_idx: int, weight: float):
	var animated_pose = skeleton.get_bone_pose(bone_idx)

	# Blend look rotation with animated rotation
	var look_basis = animated_pose.basis.slerp(
		animated_pose.basis * current_look_rotation,
		weight * look_at_weight
	)

	var new_pose = Transform3D(look_basis, animated_pose.origin)
	skeleton.set_bone_pose(bone_idx, new_pose)

func _apply_eye_look():
	if left_eye_idx == -1 or right_eye_idx == -1:
		return

	# Eyes can look further than head
	var eye_rotation = current_look_rotation
	var extra_rotation_factor = 1.3  # Eyes can look 30% further

	eye_rotation = Basis.IDENTITY.slerp(eye_rotation, extra_rotation_factor)

	# Apply to both eyes
	_apply_bone_look(left_eye_idx, eye_weight * 1.5)
	_apply_bone_look(right_eye_idx, eye_weight * 1.5)

func _return_to_neutral(delta: float):
	# Smoothly return to animated pose
	current_look_rotation = current_look_rotation.slerp(
		Basis.IDENTITY,
		return_speed * delta
	)

	if current_look_rotation.is_equal_approx(Basis.IDENTITY):
		current_look_rotation = Basis.IDENTITY

	_apply_look_rotation()

# Saccadic eye movement (realistic micro-adjustments)
func _update_saccades(delta: float):
	saccade_timer += delta

	if saccade_timer >= saccade_interval:
		saccade_timer = 0.0
		saccade_interval = randf_range(0.2, 0.5)  # Randomize timing

		# Small random offset for eye "searching"
		saccade_offset = Vector3(
			randf_range(-0.1, 0.1),
			randf_range(-0.1, 0.1),
			randf_range(-0.05, 0.05)
		)

# Blinking system
func _update_blinks(delta: float):
	blink_timer += delta

	if not is_blinking and blink_timer >= blink_interval:
		_trigger_blink()

	if is_blinking:
		# Blink animation would go here
		if blink_timer >= blink_interval + blink_duration:
			is_blinking = false
			blink_timer = 0.0
			blink_interval = randf_range(2.0, 6.0)  # Variable blink rate

func _trigger_blink():
	is_blinking = true
	# Close eyelids (if using blend shapes/bones)
	_animate_eyelids(true)

func _animate_eyelids(closed: bool):
	# If using MeshInstance3D with blend shapes
	var mesh = get_parent().get_node_or_null("Head/Face")
	if mesh and mesh is MeshInstance3D:
		var target_blink = 1.0 if closed else 0.0
		# Tween blend shape
		var tween = create_tween()
		tween.tween_property(mesh, "blend_shapes/blink", target_blink, 0.1)

# Advanced: Attention priority system
class LookTarget:
	var position: Vector3
	var priority: int  # Higher = more important
	var duration: float = -1  # How long to look (-1 = until cleared)
	var focus_weight: float = 1.0

enum LookPriority {
	IDLE = 0,           # Random environment scanning
	CURIOSITY = 1,      # Interesting object
	CONVERSATION = 2,   # Talking NPC
	PLAYER = 3,         # Player nearby
	THREAT = 4,         # Enemy/danger
	FORCED = 5          # Scripted event
}

var look_queue: Array[LookTarget] = []

func add_look_target(position: Vector3, priority: LookPriority, duration: float = -1):
	var target = LookTarget.new()
	target.position = position
	target.priority = priority
	target.duration = duration

	look_queue.append(target)
	look_queue.sort_custom(func(a, b): return a.priority > b.priority)

	# Set highest priority target
	if look_queue.size() > 0:
		set_look_target(look_queue[0].position)

func _process_look_queue(delta: float):
	for i in range(look_queue.size() - 1, -1, -1):
		var target = look_queue[i]
		if target.duration > 0:
			target.duration -= delta
			if target.duration <= 0:
				look_queue.remove_at(i)

	# Update current target
	if look_queue.size() > 0:
		set_look_target(look_queue[0].position)
	else:
		clear_look_target()

# Advanced: Look-at with anticipation (look where moving)
func look_at_movement_direction(velocity: Vector3, weight: float = 0.3):
	if velocity.length() < 0.1:
		return

	var look_ahead_distance = 2.0
	var look_ahead_point = global_position + velocity.normalized() * look_ahead_distance

	# Blend movement direction with target
	if has_target:
		current_target = current_target.lerp(look_ahead_point, weight)
	else:
		set_look_target(look_ahead_point)

# Example: NPC awareness system
class_name NPCLookBehavior
extends Node3D

@export var head_look: HeadLookAt
@export var awareness_radius: float = 10.0
@export var scan_interval: float = 1.0

var scan_timer: float = 0.0
var player: Node3D

func _ready():
	player = get_tree().get_first_node_in_group("player")

func _process(delta: float):
	scan_timer += delta

	if scan_timer >= scan_interval:
		scan_timer = 0.0
		_scan_environment()

func _scan_environment():
	# Check if player is nearby
	if player:
		var distance = global_position.distance_to(player.global_position)

		if distance < awareness_radius:
			# Look at player
			head_look.add_look_target(
				player.global_position + Vector3.UP * 1.5,  # Eye level
				HeadLookAt.LookPriority.PLAYER,
				2.0  # Look for 2 seconds
			)
		else:
			# Idle scanning
			_look_at_random_point()

func _look_at_random_point():
	# Random environmental scanning
	var random_offset = Vector3(
		randf_range(-3, 3),
		randf_range(0, 2),
		randf_range(-3, 3)
	)
	var random_point = global_position + random_offset

	head_look.add_look_target(
		random_point,
		HeadLookAt.LookPriority.IDLE,
		randf_range(1.0, 3.0)
	)

# Advanced: Gaze following (NPCs watch what player looks at)
func npc_follow_player_gaze():
	if not player:
		return

	# Get player's look direction
	var player_camera = player.get_node("Camera3D")
	if player_camera:
		var gaze_point = player_camera.global_position + player_camera.global_transform.basis.z * -5.0
		head_look.add_look_target(gaze_point, HeadLookAt.LookPriority.CURIOSITY, 1.5)

# Debug visualization
func debug_draw_look_at():
	if not has_target:
		return

	var head_pos = skeleton.global_transform * skeleton.get_bone_global_pose(head_bone_idx).origin

	# Draw line from head to target
	DebugDraw3D.draw_line(head_pos, current_target, Color.YELLOW)
	DebugDraw3D.draw_sphere(current_target, 0.1, Color.RED)

	# Draw constraint cone
	# (visualization of max angles)

# Performance optimization: LOD for look-at
func set_lod_level(distance: float):
	if distance < 10.0:
		# Full quality
		look_speed = 3.0
		look_at_enabled = true
	elif distance < 30.0:
		# Medium quality
		look_speed = 1.5
		look_at_enabled = true
	else:
		# Disabled
		look_at_enabled = false
```

**Key Parameters:**
- **Look weight:** 0.6-1.0 (realistic), 0.3-0.5 (subtle)
- **Head rotation:** 60-80% of total look
- **Neck rotation:** 40-60% of total look
- **Spine rotation:** 15-25% (subtle torso twist)
- **Max horizontal:** 75-110° (±90° typical)
- **Max vertical:** 45-70° up/down
- **Look speed:** 2-5 (smooth), 6-10 (responsive)
- **Saccade interval:** 0.2-0.5s randomized

**Edge Cases:**
- **Target behind character:** Clamp to 90° to avoid unnatural twist
- **Extreme angles:** Blend weight to 0 beyond comfortable range
- **Animation conflicts:** Reduce weight during combat/scripted anims
- **Multiple targets:** Priority system to avoid rapid switching
- **Network multiplayer:** Predict target on clients to hide latency
- **Cutscenes:** Override with scripted look-at targets
- **Death/ragdoll:** Disable look-at immediately

**When NOT to Use:**
- **Helmet/masked characters:** No visible face/eyes to track
- **Non-humanoid creatures:** May need custom bone setup
- **Performance critical:** Mobile low-end may struggle
- **Intentional stiff aesthetic:** Robots, zombies may want fixed gaze
- **Top-down camera:** Player can't see head tracking detail
- **Very fast gameplay:** Changes too rapid to perceive
- **Abstract characters:** No realistic anatomy to track

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Industry-leading facial animation + look-at. NPCs make eye contact during conversations, track player during stealth. Subtle head tilts and micro-expressions combined with look-at. Combat enemies track player position with head turns. Attention system: Allies look where they're moving, enemies look toward last known player position. Eyes have saccadic micro-movements. Performance capture combined with procedural look-at for natural behavior.

2. **Red Dead Redemption 2:** NPCs acknowledge Arthur with head turns and eye contact. Townspeople watch player pass by (radius-based trigger). Conversation system uses look-at for eye contact during dialogue. Gang members look at who's speaking during camp scenes. Horse head tracks direction of travel (separate quadruped system). Saloon patrons follow player with eyes (subtle, unsettling effect). Critical for immersion in social world.

3. **Half-Life: Alyx (VR):** VR-specific implementation - NPCs maintain eye contact with player headset position. Subtle head tilts follow player movement. Combines look-at with lip-sync for believable characters. Performance: All nearby NPCs (10+) track player simultaneously. Enemies use look-at for threat tracking (gameplay feedback). Alyx (companion) uses complex attention system (looks at objectives, enemies, player).

4. **Hellblade: Senua's Sacrifice:** Senua's head tracks audio hallucinations (voices position). Combat enemies use aggressive look-at (locks onto player for intimidation). Subtle eye darting contributes to psychological horror. Look-at weight increases during psychosis episodes. Mirror reflections maintain proper look-at (technical challenge). Design philosophy: "Eyes tell the story."

5. **Horizon Forbidden West:** Aloy's head tracks climbing paths (player feedback). NPCs in settlements watch Aloy pass by (living world feel). Conversation system with dynamic camera uses look-at for eye contact. Machines use look-at for threat detection (red eye glow follows player). Facial mo-cap blended with real-time look-at. Facial animation + look-at budget: ~15% of character animation cost.

**Platform Considerations:**
- **PC/Console:** Full quality, 60Hz updates, all nearby characters
- **Mobile:** Reduce to 30Hz, limit to 3-5 characters, skip eye bones
- **VR:** Critical for presence, prioritize even on lower-end
- **Network:** Client-side prediction, sync target position not rotation
- **CPU cost:** 0.01-0.02ms per character (bone manipulation)
- **Animation LOD:** Disable for characters >30m away

**Godot-Specific Notes:**
- `Skeleton3D.get_bone_pose()` / `set_bone_pose()` for manipulation
- `Transform3D.looking_at()` can help calculate look rotations
- Use `Basis.slerp()` for smooth rotation interpolation
- `SkeletonIK3D` can be used but custom solution gives more control
- Store bone indices on `_ready()`, don't lookup every frame
- `_process()` for smooth 60fps look updates (not _physics_process)
- Network: Use `rpc_unreliable()` for target position updates
- Performance: Profile with 10-20 characters, target <0.3ms total

**Synergies:**
- **Facial Animation:** Combine look-at with blend shape expressions
- **Lip Sync:** Coordinate with dialogue for believable conversations
- **AI Perception:** Look-at reflects what AI "sees"
- **Stealth Systems:** Enemies look toward suspicious sounds
- **Dialogue System:** Auto-look at conversation participants
- **Animation State Machine:** Reduce weight during certain states
- **Attention System:** Visual feedback of NPC awareness

**Measurement/Profiling:**
- **Look accuracy:** Should track within 5-10° of target
- **Smoothness:** No visible pops/snaps during transitions
- **CPU cost:** <0.02ms per character target
- **Visual quality:** QA review for natural vs uncanny feel
- **Player feedback:** Survey whether NPCs feel "alive"
- **Performance budget:** Total look-at <0.5ms for 30 characters
- **Network bandwidth:** Target sync packets <50 bytes/update

**Advanced Techniques:**
```gdscript
# Gaze avoidance (shy NPCs look away)
func calculate_gaze_avoidance(target_pos: Vector3, personality: float) -> Vector3:
	var offset_angle = lerp(0, 30, personality)  # 0 = confident, 1 = shy
	var offset = Vector3.RIGHT * sin(deg_to_rad(offset_angle))
	return target_pos + offset

# Predict moving target position
func predict_target_position(target: Node3D, prediction_time: float) -> Vector3:
	if target is CharacterBody3D or target is RigidBody3D:
		var velocity = target.velocity if "velocity" in target else Vector3.ZERO
		return target.global_position + velocity * prediction_time
	return target.global_position

# Emotion-based look behavior
func get_look_speed_for_emotion(emotion: String) -> float:
	match emotion:
		"afraid": return 8.0    # Rapid, darting
		"calm": return 2.0      # Slow, deliberate
		"angry": return 5.0     # Focused, quick
		"curious": return 4.0   # Moderate
	return 3.0

# Interest decay (look away after time)
var interest_level: float = 1.0
var interest_decay_rate: float = 0.1

func update_interest(delta: float):
	if has_target:
		interest_level -= interest_decay_rate * delta
		look_at_weight = max(interest_level, 0.3)  # Min 30% weight

# Conversation system integration
func start_conversation(other_character: Node3D):
	look_at_weight = 1.0
	set_look_target(other_character.global_position + Vector3.UP * 1.5)
	# Lock target during conversation
	look_priority = LookPriority.CONVERSATION

# Injured/tired look behavior
func apply_fatigue_to_look(fatigue: float):
	# Slower, less accurate tracking when tired
	look_speed = lerp(3.0, 1.0, fatigue)
	max_vertical_angle = lerp(60.0, 30.0, fatigue)  # Drooping head
```

---
