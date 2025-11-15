### 103. Animation Blending Transitions

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Jarring animation snaps between states (idle → run → jump) break immersion and feel robotic. Instant transitions make character movement feel disconnected and unnatural. Players perceive character as puppet, not living entity. Without blending, responsive gameplay conflicts with smooth animation, forcing compromise between feel and visuals.

**Technical Explanation:**
Interpolate between animation states over time (typically 0.05-0.2s) instead of instant switching. Uses weighted bone/sprite position blending: `final_position = animation_A_position * (1 - blend_weight) + animation_B_position * blend_weight`. Blend weight transitions from 0 to 1 using easing curves. State machine tracks current/next animation and blend progress. Advanced systems use blend trees with 2D blending (walk/run + direction) and additive blending (base animation + overlay like aiming).

**Algorithmic Complexity:**
- O(n) where n = number of bones/animated properties
- O(1) per bone: Linear interpolation (lerp)
- CPU: 0.01-0.5ms for 10-100 bones
- Memory: ~100 bytes per animation state

**Implementation Pattern:**
```gdscript
# Godot implementation - Animation Blending System
class_name AnimationBlender
extends Node

@export var animation_tree: AnimationTree
@export var animation_player: AnimationPlayer
@export var blend_time: float = 0.15  # Default blend duration

var current_animation: String = "idle"
var target_animation: String = "idle"
var blend_progress: float = 0.0
var is_blending: bool = false

# Different blend times for different transition types
const BLEND_TIMES = {
	"idle_to_walk": 0.1,
	"walk_to_run": 0.15,
	"ground_to_air": 0.05,  # Quick for responsive jump
	"air_to_ground": 0.12,  # Slower for landing impact
	"any_to_attack": 0.08,  # Fast attack startup
	"attack_to_idle": 0.2,  # Slower recovery
}

func _ready():
	if animation_tree:
		animation_tree.active = true

func _process(delta):
	if is_blending:
		_update_blend(delta)

func transition_to(new_animation: String, custom_blend_time: float = -1.0):
	if new_animation == current_animation and not is_blending:
		return

	# Determine blend time
	var blend_duration = custom_blend_time
	if blend_duration < 0:
		var key = current_animation + "_to_" + new_animation
		blend_duration = BLEND_TIMES.get(key, blend_time)

	# Start blending
	target_animation = new_animation
	blend_progress = 0.0
	is_blending = true

	# Emit signal for other systems
	animation_transition_started.emit(current_animation, target_animation)

func _update_blend(delta):
	# Advance blend progress
	var blend_duration = _get_current_blend_time()
	blend_progress += delta / blend_duration

	if blend_progress >= 1.0:
		# Blend complete
		_finish_blend()
	else:
		# Update blend weight
		_apply_blend(blend_progress)

func _apply_blend(weight: float):
	# Ease the blend weight for smoother transitions
	var eased_weight = ease(weight, -2.0)  # Ease in-out

	if animation_tree:
		# Use AnimationTree blend nodes
		animation_tree.set("parameters/blend_position", eased_weight)
	else:
		# Manual blending with AnimationPlayer
		_manual_blend(eased_weight)

func _finish_blend():
	current_animation = target_animation
	blend_progress = 0.0
	is_blending = false

	animation_transition_completed.emit(current_animation)

func _manual_blend(weight: float):
	# Fallback: Cross-fade between animations
	animation_player.play(current_animation)
	animation_player.queue(target_animation)
	animation_player.advance(weight)

func _get_current_blend_time() -> float:
	var key = current_animation + "_to_" + target_animation
	return BLEND_TIMES.get(key, blend_time)

# Advanced: 2D Blend Space (movement blending)
class_name BlendSpace2D_Movement extends Node

@export var animation_tree: AnimationTree
@export var blend_speed: float = 10.0  # How fast to blend to new position

var current_blend_pos: Vector2 = Vector2.ZERO
var target_blend_pos: Vector2 = Vector2.ZERO

# Blend space: X = direction (-1 left, 1 right), Y = speed (0 idle, 1 run)
func update_movement(direction: float, speed: float):
	# Normalize speed to 0-1
	var normalized_speed = clamp(speed / max_speed, 0.0, 1.0)

	# Target position in blend space
	target_blend_pos = Vector2(direction, normalized_speed)

func _process(delta):
	# Smooth blend to target position
	current_blend_pos = current_blend_pos.lerp(target_blend_pos, blend_speed * delta)

	# Update AnimationTree blend space
	animation_tree.set("parameters/movement/blend_position", current_blend_pos)

# Additive animation blending (aiming while moving)
class_name AdditiveBlending extends Node

@export var animation_tree: AnimationTree
@export var base_animation: String = "walk"
@export var additive_animation: String = "aim"
@export var additive_strength: float = 1.0

func _process(_delta):
	# Base animation plays normally
	animation_tree.set("parameters/base/current", base_animation)

	# Additive animation overlays (e.g., upper body aiming while legs walk)
	animation_tree.set("parameters/additive/add_amount", additive_strength)

# State machine with automatic blending
class_name AnimationStateMachine extends Node

enum State { IDLE, WALK, RUN, JUMP, FALL, ATTACK, HURT }

var current_state: State = State.IDLE
var previous_state: State = State.IDLE

@export var animation_tree: AnimationTree

# State transition graph
var valid_transitions: Dictionary = {
	State.IDLE: [State.WALK, State.JUMP, State.ATTACK],
	State.WALK: [State.IDLE, State.RUN, State.JUMP, State.ATTACK],
	State.RUN: [State.WALK, State.JUMP],
	State.JUMP: [State.FALL],
	State.FALL: [State.IDLE, State.WALK],
	State.ATTACK: [State.IDLE, State.WALK],
	State.HURT: [State.IDLE, State.FALL],
}

func change_state(new_state: State):
	if new_state == current_state:
		return

	# Check if transition is valid
	if new_state not in valid_transitions.get(current_state, []):
		push_warning("Invalid state transition: %s -> %s" % [current_state, new_state])
		return

	previous_state = current_state
	current_state = new_state

	# Trigger blend
	var state_name = State.keys()[new_state].to_lower()
	_blend_to_state(state_name)

func _blend_to_state(state_name: String):
	animation_tree.set("parameters/StateMachine/transition_request", state_name)

# Bone masking (blend only specific bones)
class_name BoneMaskBlending extends Node

@export var animation_tree: AnimationTree
@export var skeleton: Skeleton3D

# Blend upper body (aim) with lower body (walk)
func setup_bone_masks():
	# Create blend filter for upper body
	var upper_body_bones = ["spine", "shoulder_l", "shoulder_r", "arm_l", "arm_r"]

	# Lower body uses walk animation
	# Upper body uses aim animation
	for bone_name in upper_body_bones:
		var bone_idx = skeleton.find_bone(bone_name)
		# Set bone mask in AnimationTree (Godot 4.x)
		# animation_tree.set_bone_mask(bone_idx, true)
```

**Key Parameters:**
- **Fast transitions:** 0.05-0.1s (jump, attack)
- **Normal transitions:** 0.1-0.2s (walk ↔ run)
- **Slow transitions:** 0.2-0.4s (recovery, stance change)
- **Blend curve:** Linear, ease-in-out (most common), custom curves
- **2D blend space resolution:** 5x5 grid (25 animation samples)
- **Additive blend strength:** 0.0-1.0 (how much overlay affects base)
- **Bone mask count:** 10-50 bones for complex rigs

**Edge Cases:**
- **Interrupting blend:** New transition during active blend
- **Looping vs one-shot:** Different blend behavior at end
- **Animation length mismatch:** Shorter animation completes during blend
- **Extreme blend positions:** Extrapolation can cause distortion
- **Bone hierarchy issues:** Child bones inherit parent transforms
- **Networked games:** Sync state, not blend progress (too much data)
- **Dead character:** Disable blending, play death immediately

**When NOT to Use:**
- **Pixel art:** Frame-by-frame animation, no interpolation
- **Fighting games:** Frame-perfect precision required, no blending
- **Retro aesthetic:** Instant snaps are intentional style
- **Simple 2D sprites:** Single sprite swap, no skeleton
- **Performance critical:** 1000+ characters, disable blending
- **Stylized snaps:** Some games use instant transitions intentionally

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Industry-leading blend times (0.05-0.15s), 2D blend spaces for movement in 8 directions. Motion matching system blends between 1000s of animation clips. Upper body (aiming) blends independently from lower body (movement).

2. **Uncharted series:** 0.1-0.2s blends between locomotion states, additive aiming layer, procedural hand placement. Seamless transitions between climbing/combat/exploration. Responsive yet smooth.

3. **Celeste:** Despite being 2D pixel art, uses frame interpolation for sprite position blending. Madeline's hair uses interpolation between states. 0.08s transitions feel instant but avoid pops.

4. "Ori and the Blind Forest:** Sprite-based but uses skeleton animation with blending (0.1-0.15s). Complex blend trees for movement. Triple jump has unique blend curves for each jump.

5. **Dark Souls:** Deliberate slow blends (0.2-0.4s) for attack recovery, faster blends (0.1s) for movement. Blending creates "weight" and commitment to actions. Intentionally slower for design reasons.

**Platform Considerations:**
- **Desktop:** Full blend trees with 100+ bones supported
- **Console:** Optimize to 50-80 bones for 60fps
- **Mobile:** Limit to 20-30 bones, simpler blend trees
- **Switch:** 30-50 bones recommended
- **VR:** Fast blends (0.05-0.08s) prevent motion sickness
- **High refresh rate:** Smoother blends more noticeable at 120fps+

**Godot-Specific Notes:**
- `AnimationTree` node with `AnimationNodeBlendTree` or `AnimationNodeStateMachine`
- `AnimationNodeBlend2/Blend3` for weighted blending
- `AnimationNodeBlendSpace1D/2D` for parametric blending
- `AnimationNodeTransition` with `xfade_time` parameter
- `Skeleton2D/3D` for bone-based animation
- `blend_mode` property: interpolated vs discrete
- Godot 4.x: Improved `AnimationMixer` and blend tree performance

**Synergies:**
- **Squash & Stretch (Technique 96):** Blend squash values between states
- **Follow-Through (Technique 101):** Blend secondary motion smoothly
- **Anticipation (Technique 100):** Blend into anticipation frames
- **Weapon Weight (Technique 105):** Different blend times per weapon
- **Movement Systems:** Smooth transitions between speeds/directions
- **IK Systems:** Blend procedural IK with authored animations

**Measurement/Profiling:**
- **Visual testing:** Record transitions, verify no pops/jitter
- **Blend time tuning:** Playtest each transition type, adjust to feel
- **Performance:** Godot profiler, target <0.5ms for animation system
- **Frame timing:** Verify blend completes in intended time ±1 frame
- **Bone count:** Monitor skeleton complexity, optimize if >100 bones
- **State machine complexity:** Limit to 10-20 states for readability
- **Playtest feedback:** "Do movements feel smooth and responsive?"

**Advanced Techniques:**
```gdscript
# Inertial blending (momentum-preserving transitions)
class_name InertialBlend extends Node

var bone_velocities: Dictionary = {}  # bone_name: Vector3

func blend_with_inertia(from_anim: String, to_anim: String, blend_time: float):
	# Calculate bone velocities at transition point
	for bone in skeleton.get_bone_count():
		var bone_name = skeleton.get_bone_name(bone)
		var from_pos = _get_bone_position(from_anim, bone)
		var to_pos = _get_bone_position(to_anim, bone)

		# Velocity = position delta
		bone_velocities[bone_name] = (to_pos - from_pos) / blend_time

	# Blend includes velocity offset to prevent foot sliding
	_apply_velocity_blend(blend_time)

# Directional blend space (8-way movement)
func setup_8_way_blend_space():
	# Blend space positions for 8 directions
	var directions = [
		Vector2(0, 1),    # Forward
		Vector2(1, 1),    # Forward-right
		Vector2(1, 0),    # Right
		Vector2(1, -1),   # Back-right
		Vector2(0, -1),   # Back
		Vector2(-1, -1),  # Back-left
		Vector2(-1, 0),   # Left
		Vector2(-1, 1),   # Forward-left
	]

	# Assign animation to each blend point
	for i in range(directions.size()):
		var point_name = "walk_" + str(i)
		animation_tree.set("parameters/movement/blend_points/" + point_name, directions[i])

# Animation layering (multiple blend layers)
class_name AnimationLayering extends Node

@export var animation_tree: AnimationTree

# Layer priorities
const LAYER_BASE = 0      # Locomotion
const LAYER_UPPER = 1     # Aiming/gestures
const LAYER_FACIAL = 2    # Facial expressions
const LAYER_ADDITIVE = 3  # Hits/reactions

func play_layered_animation(layer: int, anim_name: String, blend_time: float = 0.1):
	var layer_path = "parameters/layer_" + str(layer) + "/blend"
	animation_tree.set(layer_path, anim_name)

# Cross-fade groups (multiple animations fade together)
class_name CrossFadeGroup extends Node

var active_animations: Array[String] = []
var animation_weights: Array[float] = []

func cross_fade_to_group(new_animations: Array[String], fade_time: float):
	# Fade out old group
	for anim in active_animations:
		_fade_out_animation(anim, fade_time)

	# Fade in new group
	active_animations = new_animations
	for anim in new_animations:
		_fade_in_animation(anim, fade_time)

# Blend curve customization
@export var blend_curve: Curve  # Edit in inspector

func blend_with_curve(weight: float) -> float:
	# Use custom curve instead of linear blend
	return blend_curve.sample(weight)

# State transition conditions
class_name ConditionalTransitions extends Node

func can_transition(from_state: State, to_state: State) -> bool:
	match [from_state, to_state]:
		[State.ATTACK, State.JUMP]:
			return false  # Can't jump during attack
		[State.HURT, _]:
			return hurt_recovery_complete  # Only transition after hurt
		[State.FALL, State.IDLE]:
			return is_on_floor  # Can't idle in air
		_:
			return true

# Animation retargeting (different skeleton proportions)
class_name AnimationRetargeting extends Node

func retarget_animation(source_skeleton: Skeleton3D, target_skeleton: Skeleton3D):
	# Map bone names between skeletons
	var bone_map: Dictionary = {
		"mixamorig:Hips": "root",
		"mixamorig:Spine": "spine_01",
		# ... etc
	}

	# Apply source animation to target skeleton
	for source_bone in bone_map.keys():
		var target_bone = bone_map[source_bone]
		_copy_bone_transform(source_skeleton, source_bone, target_skeleton, target_bone)
```

**Blend Tree Example (AnimationTree setup):**
```gdscript
# Code to construct blend tree programmatically
func setup_blend_tree():
	var blend_tree = AnimationNodeBlendTree.new()

	# Create nodes
	var anim_idle = AnimationNodeAnimation.new()
	anim_idle.animation = "idle"

	var anim_walk = AnimationNodeAnimation.new()
	anim_walk.animation = "walk"

	var anim_run = AnimationNodeAnimation.new()
	anim_run.animation = "run"

	# Create blend node
	var blend_2d = AnimationNodeBlendSpace2D.new()
	blend_2d.add_blend_point(anim_idle, Vector2(0, 0))
	blend_2d.add_blend_point(anim_walk, Vector2(0, 0.5))
	blend_2d.add_blend_point(anim_run, Vector2(0, 1.0))

	# Add to tree
	blend_tree.add_node("movement", blend_2d)
	animation_tree.tree_root = blend_tree
```
