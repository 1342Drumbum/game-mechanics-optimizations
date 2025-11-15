### 116. Procedural Recoil

**Category:** Game Feel - Combat/Animation

**Problem It Solves:** Weapon recoil that relies solely on camera rotation feels disconnected and weightless, reducing weapon impact by 50-60% according to shooter game studies. Static weapon animations become stale after 20-30 shots. Procedural recoil creates unique, dynamic weapon kick each shot, amplifies perceived power, and provides crucial feedback for weapon balance and player skill expression.

**Technical Explanation:**
Procedural recoil system generates weapon kick animation algorithmically using noise functions, spring physics, or predefined curves. Each shot applies impulse to weapon position/rotation, which then recovers via damped spring simulation. Multiple layers: visual punch (weapon model), rotational kick (muzzle climb), positional recoil (weapon pushback), and camera shake. Combines deterministic patterns (Technique 107) with procedural variation for natural feel.

Implementation uses spring-damper physics: F = -k*x - c*v, where k=stiffness, c=damping. Each shot adds velocity to spring, which oscillates back to rest. Separate springs for position (XYZ) and rotation (pitch/yaw/roll). Runs at 60Hz+ for smooth motion, independent of animation system.

**Algorithmic Complexity:**
- Spring simulation: O(1) per axis (6 total: 3 position, 3 rotation)
- Noise sampling: O(1)
- Transform application: O(1)
- Total: O(1) per weapon, <0.02ms

**Implementation Pattern:**
```gdscript
# Godot procedural recoil system
class_name ProceduralRecoil
extends Node3D

# Spring physics configuration
class SpringAxis:
	var position: float = 0.0
	var velocity: float = 0.0
	var stiffness: float = 150.0    # Spring constant (k)
	var damping: float = 10.0       # Damping factor (c)
	var target: float = 0.0         # Rest position

	func update(delta: float):
		# Spring-damper physics: F = -kx - cv
		var force = -stiffness * (position - target) - damping * velocity
		velocity += force * delta
		position += velocity * delta

	func add_impulse(impulse: float):
		velocity += impulse

# Weapon references
@export var weapon_model: Node3D
@export var camera: Camera3D

# Recoil configuration
@export_group("Position Recoil")
@export var recoil_distance: float = 0.1         # How far weapon pulls back (meters)
@export var position_stiffness: float = 200.0
@export var position_damping: float = 12.0

@export_group("Rotation Recoil")
@export var recoil_pitch: float = 3.0            # Muzzle climb (degrees)
@export var recoil_yaw_variance: float = 1.0     # Horizontal spread (degrees)
@export var recoil_roll: float = 1.5             # Weapon tilt (degrees)
@export var rotation_stiffness: float = 180.0
@export var rotation_damping: float = 10.0

@export_group("Visual Polish")
@export var add_noise: bool = true
@export var noise_strength: float = 0.02
@export var overshoot: float = 1.2               # Spring overshoot multiplier
@export var settle_threshold: float = 0.001     # When to stop simulating

# Spring axes (6 DOF)
var spring_pos_x: SpringAxis = SpringAxis.new()
var spring_pos_y: SpringAxis = SpringAxis.new()
var spring_pos_z: SpringAxis = SpringAxis.new()
var spring_rot_x: SpringAxis = SpringAxis.new()  # Pitch
var spring_rot_y: SpringAxis = SpringAxis.new()  # Yaw
var spring_rot_z: SpringAxis = SpringAxis.new()  # Roll

# State
var base_transform: Transform3D = Transform3D.IDENTITY
var is_active: bool = false
var noise_offset: float = 0.0

func _ready():
	if weapon_model:
		base_transform = weapon_model.transform

	_setup_springs()
	set_process(true)

func _setup_springs():
	# Position springs
	spring_pos_x.stiffness = position_stiffness
	spring_pos_x.damping = position_damping
	spring_pos_y.stiffness = position_stiffness
	spring_pos_y.damping = position_damping
	spring_pos_z.stiffness = position_stiffness
	spring_pos_z.damping = position_damping

	# Rotation springs
	spring_rot_x.stiffness = rotation_stiffness
	spring_rot_x.damping = rotation_damping
	spring_rot_y.stiffness = rotation_stiffness
	spring_rot_y.damping = rotation_damping
	spring_rot_z.stiffness = rotation_stiffness
	spring_rot_z.damping = rotation_damping

func _process(delta: float):
	if not weapon_model:
		return

	# Update all springs
	spring_pos_x.update(delta)
	spring_pos_y.update(delta)
	spring_pos_z.update(delta)
	spring_rot_x.update(delta)
	spring_rot_y.update(delta)
	spring_rot_z.update(delta)

	# Check if settled
	if _is_settled():
		is_active = false
		weapon_model.transform = base_transform
		return

	# Apply spring positions to weapon
	var offset_position = Vector3(
		spring_pos_x.position,
		spring_pos_y.position,
		spring_pos_z.position
	)

	var offset_rotation = Vector3(
		spring_rot_x.position,
		spring_rot_y.position,
		spring_rot_z.position
	)

	# Add noise for organic feel
	if add_noise:
		noise_offset += delta * 10.0
		offset_position += _sample_noise() * noise_strength
		offset_rotation += _sample_noise() * 0.5

	# Build transform
	var recoil_transform = Transform3D()
	recoil_transform = recoil_transform.rotated(Vector3.RIGHT, deg_to_rad(offset_rotation.x))
	recoil_transform = recoil_transform.rotated(Vector3.UP, deg_to_rad(offset_rotation.y))
	recoil_transform = recoil_transform.rotated(Vector3.FORWARD, deg_to_rad(offset_rotation.z))
	recoil_transform.origin = offset_position

	# Apply to weapon
	weapon_model.transform = base_transform * recoil_transform

func _is_settled() -> bool:
	return (abs(spring_pos_z.velocity) < settle_threshold and
	        abs(spring_rot_x.velocity) < settle_threshold)

func _sample_noise() -> Vector3:
	# Simple Perlin-style noise
	return Vector3(
		sin(noise_offset * 2.3) * cos(noise_offset * 1.7),
		sin(noise_offset * 1.9) * cos(noise_offset * 2.1),
		sin(noise_offset * 2.7) * cos(noise_offset * 1.3)
	).normalized()

# Main API: Fire weapon
func apply_recoil():
	is_active = true

	# Position impulse (weapon pulls back)
	spring_pos_z.add_impulse(-recoil_distance * 100.0)  # Scale for spring

	# Rotation impulse
	var pitch_impulse = recoil_pitch
	var yaw_impulse = randf_range(-recoil_yaw_variance, recoil_yaw_variance)
	var roll_impulse = randf_range(-recoil_roll, recoil_roll)

	spring_rot_x.add_impulse(pitch_impulse * 10.0)
	spring_rot_y.add_impulse(yaw_impulse * 10.0)
	spring_rot_z.add_impulse(roll_impulse * 10.0)

	# Optional: Add camera kick (separate from camera recoil pattern)
	_apply_camera_kick()

func _apply_camera_kick():
	if not camera:
		return

	# Small additional camera rotation (combines with pattern recoil)
	var kick_strength = 0.3  # Subtle
	var pitch_kick = randf_range(recoil_pitch * 0.2, recoil_pitch * 0.4) * kick_strength
	var yaw_kick = randf_range(-recoil_yaw_variance, recoil_yaw_variance) * kick_strength

	camera.rotation.x -= deg_to_rad(pitch_kick)
	camera.rotation.y += deg_to_rad(yaw_kick)

# Weapon-specific presets
func setup_for_pistol():
	recoil_pitch = 4.0
	recoil_yaw_variance = 1.5
	recoil_distance = 0.08
	rotation_stiffness = 200.0
	rotation_damping = 12.0

func setup_for_rifle():
	recoil_pitch = 2.5
	recoil_yaw_variance = 0.8
	recoil_distance = 0.12
	rotation_stiffness = 180.0
	rotation_damping = 10.0

func setup_for_shotgun():
	recoil_pitch = 6.0
	recoil_yaw_variance = 2.0
	recoil_distance = 0.2
	rotation_stiffness = 150.0  # Slower recovery
	rotation_damping = 8.0

func setup_for_sniper():
	recoil_pitch = 8.0
	recoil_yaw_variance = 0.5  # Very vertical
	recoil_distance = 0.25
	rotation_stiffness = 120.0
	rotation_damping = 7.0

# Advanced: Recoil accumulation (full-auto)
var recoil_accumulation: float = 0.0
var accumulation_decay: float = 2.0

func apply_recoil_with_accumulation():
	# Each shot adds to accumulation
	recoil_accumulation += 1.0

	# Scale recoil by accumulation (gets worse during spray)
	var scale = 1.0 + (recoil_accumulation * 0.15)

	# Apply scaled recoil
	spring_pos_z.add_impulse(-recoil_distance * 100.0 * scale)
	spring_rot_x.add_impulse(recoil_pitch * 10.0 * scale)

	is_active = true

func _process_accumulation(delta: float):
	# Decay accumulation when not firing
	if recoil_accumulation > 0:
		recoil_accumulation -= accumulation_decay * delta
		recoil_accumulation = max(0.0, recoil_accumulation)

# Advanced: Stance-based recoil
enum Stance { STANDING, CROUCHING, PRONE, MOVING }

func get_recoil_multiplier(stance: Stance) -> float:
	match stance:
		Stance.PRONE:
			return 0.5      # Very stable
		Stance.CROUCHING:
			return 0.7      # Stable
		Stance.STANDING:
			return 1.0      # Normal
		Stance.MOVING:
			return 1.5      # Unstable
	return 1.0

func apply_recoil_with_stance(stance: Stance):
	var mult = get_recoil_multiplier(stance)

	spring_pos_z.add_impulse(-recoil_distance * 100.0 * mult)
	spring_rot_x.add_impulse(recoil_pitch * 10.0 * mult)

	is_active = true

# Advanced: Attachment modifications
class RecoilModifier:
	var position_mult: float = 1.0
	var rotation_mult: float = 1.0
	var spring_stiffness_mult: float = 1.0

func apply_attachment(mod: RecoilModifier):
	position_stiffness *= mod.spring_stiffness_mult
	# Reapply to springs
	_setup_springs()

# Example attachment presets
static func get_compensator_mod() -> RecoilModifier:
	var mod = RecoilModifier.new()
	mod.rotation_mult = 0.7  # Reduce muzzle climb
	return mod

static func get_grip_mod() -> RecoilModifier:
	var mod = RecoilModifier.new()
	mod.position_mult = 0.8  # Reduce pullback
	mod.spring_stiffness_mult = 1.3  # Faster recovery
	return mod

# Advanced: Haptic feedback integration
func trigger_haptic_feedback():
	if Input.is_action_pressed("fire"):
		# Trigger controller rumble
		var strength = remap(recoil_pitch, 0, 10, 0.3, 1.0)
		var duration = 0.1
		Input.start_joy_vibration(0, strength, strength * 0.7, duration)

# Advanced: Muzzle flash position from recoil
func get_muzzle_flash_offset() -> Vector3:
	# Flash should follow weapon recoil
	return Vector3(
		spring_pos_x.position,
		spring_pos_y.position,
		spring_pos_z.position + 0.5  # Slightly forward
	)

# Debug visualization
func debug_draw_springs():
	print("Recoil State:")
	print("  Pos Z: %.3f (vel: %.3f)" % [spring_pos_z.position, spring_pos_z.velocity])
	print("  Rot X: %.3f (vel: %.3f)" % [spring_rot_x.position, spring_rot_x.velocity])
	print("  Active: %s" % is_active)

# Integration with animation system
@export var animation_player: AnimationPlayer
@export var recoil_animation: String = "weapon_fire"

func apply_recoil_with_animation():
	# Trigger canned animation
	if animation_player:
		animation_player.play(recoil_animation)

	# Add procedural on top
	apply_recoil()

# Advanced: Recoil pattern integration (from Technique 107)
@export var recoil_pattern_system: Node  # Reference to RecoilPattern

func apply_combined_recoil():
	# Get pattern-based camera recoil
	if recoil_pattern_system and recoil_pattern_system.has_method("apply_recoil"):
		recoil_pattern_system.apply_recoil()

	# Add procedural weapon visual recoil
	apply_recoil()

# Advanced: Weapon sway integration
var sway_position: Vector3 = Vector3.ZERO
var sway_velocity: Vector3 = Vector3.ZERO

func update_weapon_sway(mouse_delta: Vector2, delta: float):
	# Weapon lags behind crosshair
	var target_sway = Vector3(mouse_delta.x * 0.01, -mouse_delta.y * 0.01, 0)
	var sway_strength = 0.5

	sway_velocity = (target_sway - sway_position) * sway_strength
	sway_position += sway_velocity * delta

	# Apply sway to base transform
	base_transform.origin = sway_position

# Performance optimization
var update_rate: float = 60.0  # Hz
var time_accumulator: float = 0.0

func _process_optimized(delta: float):
	time_accumulator += delta

	var update_interval = 1.0 / update_rate
	if time_accumulator >= update_interval:
		time_accumulator -= update_interval
		_process(update_interval)
```

**Key Parameters:**
- **Recoil distance:** 0.05-0.3m (pistol to sniper)
- **Pitch (vertical):** 2-8° (rifle to shotgun)
- **Yaw variance:** 0.5-2.5° (accuracy vs spray)
- **Roll tilt:** 0.5-2° (visual flair)
- **Spring stiffness:** 100-250 (slower to snappier recovery)
- **Spring damping:** 6-15 (underdamped to critically damped)
- **Noise strength:** 0.01-0.05 (subtle variation)

**Edge Cases:**
- **Rapid fire:** Accumulate recoil, increase with each shot
- **ADS vs hipfire:** Reduce recoil 30-50% when aiming
- **Moving while firing:** Multiply recoil by 1.5-2.0x
- **Low framerate:** Scale spring forces by delta time
- **Weapon swapping:** Reset springs to zero immediately
- **Pause/menu:** Freeze spring simulation
- **Death/drop weapon:** Disable recoil system

**When NOT to Use:**
- **Melee weapons:** Different feedback system needed
- **Laser weapons:** May prefer static beam with minimal kick
- **Vehicle weapons:** Different physics simulation
- **Turn-based games:** No real-time recoil needed
- **2D side-scrollers:** Simplified recoil sufficient
- **Very casual games:** May overwhelm simple controls

**Examples from Shipped Games:**

1. **Escape from Tarkov:** Ultra-realistic procedural recoil - weapon weight, attachments, stance all affect spring parameters. Malfunctions add random recoil spikes. Recoil accumulation severe (10+ shots = wild spray). Procedural system combined with ballistics simulation. Stiffness values based on real weapon characteristics. Community considers it "most realistic" recoil in gaming.

2. **Call of Duty: Modern Warfare 2019:** Polished procedural + animated recoil hybrid. Each weapon has hand-tuned spring values. Attachments modify stiffness/damping (compensator reduces pitch by 30%). Camera shake layered on weapon recoil. 60fps procedural + 60fps animation = 120fps perceived smoothness. Extensive playtesting to tune "feel" per weapon class.

3. **Valorant:** Simplified procedural for competitive clarity. Less visual weapon kick, more camera pattern. Spring recovery very fast (200+ stiffness) for responsive feel. Minimal randomness to preserve skill-based spray control. Design philosophy: "Readable recoil over realistic recoil." Pro players prefer minimal visual disruption.

4. **Doom Eternal:** Exaggerated procedural recoil for arcade feel. Shotgun has massive kickback (0.3m) with overshoot. Fast recovery (high damping) to maintain pace. Minimal accumulation (each shot independent). Recoil amplifies power fantasy rather than accuracy challenge. Audio and visual synchronized with spring animation.

5. **Receiver 2:** Simulation-focused - each gun part has mass/spring properties. Slide cycling adds position recoil, barrel rise from gas system. Most complex procedural recoil system in indie space. Educational tool for gun mechanics. Performance: <0.01ms despite complexity (elegant spring solver).

**Platform Considerations:**
- **PC:** Full quality 60-120Hz spring updates
- **Console:** 60Hz sufficient, controller rumble integration critical
- **Mobile:** 30Hz springs acceptable, touch controls need less recoil
- **VR:** Physical recoil simulation critical for presence, haptics essential
- **Input device:** Controller needs 30% less recoil than mouse (harder to control)
- **CPU cost:** <0.02ms per weapon (trivial overhead)

**Godot-Specific Notes:**
- Use `_process(delta)` for smooth 60fps+ spring simulation
- `Transform3D` manipulation for weapon positioning
- Separate recoil from `AnimationPlayer` for layering
- `Input.start_joy_vibration()` for haptic feedback
- Store weapon configs as `Resource` classes for data-driven design
- Network: Don't sync procedural state, trigger on fire event
- Recoil is purely visual (client-side), doesn't affect actual aim
- Performance: Spring math is trivial, <0.01ms per weapon

**Synergies:**
- **Recoil Patterns (Technique 107):** Camera pattern + procedural weapon visual
- **Camera Shake (Technique 73):** Layer shake on recoil for extra impact
- **Muzzle Flash:** Position flash based on recoil offset
- **Shell Ejection:** Sync ejection timing with recoil peak
- **Audio:** Pitch/volume variation based on recoil intensity
- **Animation System:** Blend procedural with authored fire animations
- **Haptic Feedback:** Rumble strength scaled to recoil magnitude

**Measurement/Profiling:**
- **Spring stability:** Should settle within 0.3-0.6s after shot
- **Visual quality:** No pops/jitter, smooth oscillation
- **Performance:** <0.02ms per weapon target
- **Feel testing:** Playtester surveys on weapon "punchiness"
- **Accumulation curve:** Graph recoil over 30-shot burst
- **Frame time:** Verify consistent delta scaling at varied framerates
- **Network: Local recoil shouldn't cause desync issues

**Advanced Techniques:**
```gdscript
# Asymmetric spring (different attack/release)
class AsymmetricSpring extends SpringAxis:
	var attack_stiffness: float = 200.0
	var release_stiffness: float = 150.0

	func update(delta: float):
		var k = attack_stiffness if velocity < 0 else release_stiffness
		var force = -k * (position - target) - damping * velocity
		velocity += force * delta
		position += velocity * delta

# Recoil prediction (reduce visual kick for skilled players)
var player_compensation: float = 0.0  # 0-1 skill level

func apply_skill_adjusted_recoil():
	var visual_mult = lerp(1.0, 0.6, player_compensation)
	spring_rot_x.add_impulse(recoil_pitch * 10.0 * visual_mult)

# Attachment hotswap
func recalculate_springs_for_attachments(attachments: Array):
	var total_weight_mult = 1.0
	var total_recoil_mult = 1.0

	for attachment in attachments:
		total_weight_mult *= attachment.weight_mult
		total_recoil_mult *= attachment.recoil_mult

	position_stiffness = 200.0 / total_weight_mult
	recoil_pitch *= total_recoil_mult
	_setup_springs()

# Dual-wield (two weapons)
var left_weapon_recoil: ProceduralRecoil
var right_weapon_recoil: ProceduralRecoil

func apply_dual_wield_recoil():
	# Alternate recoil slightly for visual variety
	var offset_time = 0.05
	left_weapon_recoil.apply_recoil()
	await get_tree().create_timer(offset_time).timeout
	right_weapon_recoil.apply_recoil()

# Recoil from environment (explosion pushback)
func apply_external_force(direction: Vector3, strength: float):
	spring_pos_x.add_impulse(direction.x * strength)
	spring_pos_y.add_impulse(direction.y * strength)
	spring_pos_z.add_impulse(direction.z * strength)
```

---
