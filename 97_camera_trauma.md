### 97. Camera Trauma System (Screen Shake)

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Lack of impact and energy in combat/explosions. Events feel contained and isolated. Players don't feel the "force" of actions. Without camera shake, the world feels static and unresponsive. Weak feedback reduces player satisfaction and makes it harder to parse important events in chaotic situations.

**Technical Explanation:**
Perlin noise-based camera displacement with exponential decay. "Trauma" is a 0-1 value representing shake intensity, decays over time. Camera offset calculated as `trauma^2 * max_offset * noise(time)`. The squared trauma creates natural falloff - low trauma barely shakes, high trauma shakes violently. Uses 2D/3D Perlin noise for smooth, organic motion instead of random jitter. Rotation shake adds tilt. Inspired by Vlambeer's screenshake techniques (Nuclear Throne, Luftrausers).

**Algorithmic Complexity:**
- O(1) per frame: Noise lookup + offset calculation
- O(1) memory: ~6 floats (trauma, offsets, noise seeds)
- GPU cost: Camera transform only, negligible
- CPU: ~0.01-0.05ms per frame during shake

**Implementation Pattern:**
```gdscript
# Godot implementation - Camera Trauma System
class_name CameraTrauma
extends Camera2D

@export var decay_rate: float = 1.5  # Trauma decay per second
@export var max_offset: float = 100.0  # Maximum camera displacement (pixels)
@export var max_rotation: float = 0.1  # Maximum rotation (radians, ~5.7 degrees)
@export var noise_speed: float = 50.0  # How fast noise evolves
@export var trauma_exponent: float = 2.0  # Shake curve (2 = quadratic)

var trauma: float = 0.0  # Current trauma value [0, 1]
var noise: FastNoiseLite
var noise_seed_x: int
var noise_seed_y: int
var noise_seed_rotation: int
var time_offset: float = 0.0

func _ready():
	# Initialize Perlin noise
	noise = FastNoiseLite.new()
	noise.noise_type = FastNoiseLite.TYPE_PERLIN
	noise.frequency = 2.0

	# Random seeds for independent noise channels
	noise_seed_x = randi()
	noise_seed_y = randi()
	noise_seed_rotation = randi()

func _process(delta):
	if trauma > 0.0:
		# Decay trauma over time
		trauma = max(trauma - decay_rate * delta, 0.0)
		_apply_shake(delta)
	else:
		# Reset camera when trauma depletes
		offset = Vector2.ZERO
		rotation = 0.0

func _apply_shake(delta):
	time_offset += delta * noise_speed

	# Calculate shake intensity (exponential curve)
	var shake_intensity = pow(trauma, trauma_exponent)

	# Sample noise for each axis
	noise.seed = noise_seed_x
	var offset_x = noise.get_noise_1d(time_offset) * max_offset * shake_intensity

	noise.seed = noise_seed_y
	var offset_y = noise.get_noise_1d(time_offset + 100.0) * max_offset * shake_intensity

	noise.seed = noise_seed_rotation
	var offset_rotation = noise.get_noise_1d(time_offset + 200.0) * max_rotation * shake_intensity

	# Apply offsets
	offset = Vector2(offset_x, offset_y)
	rotation = offset_rotation

# Add trauma (clamped to 0-1)
func add_trauma(amount: float):
	trauma = min(trauma + amount, 1.0)

# Preset trauma amounts for different events
func shake_light():
	add_trauma(0.2)  # Small hit

func shake_medium():
	add_trauma(0.4)  # Explosion, heavy hit

func shake_heavy():
	add_trauma(0.6)  # Boss attack, player death

func shake_extreme():
	add_trauma(1.0)  # Screen transition, ultimate ability

# Distance-based trauma (for explosions)
func add_trauma_at_position(position: Vector2, radius: float, max_trauma: float):
	var camera_pos = get_screen_center_position()
	var distance = camera_pos.distance_to(position)

	if distance < radius:
		# Falloff: closer = more shake
		var falloff = 1.0 - (distance / radius)
		add_trauma(max_trauma * falloff)

# Directional shake (push camera away from source)
func add_directional_trauma(amount: float, direction: Vector2):
	add_trauma(amount)
	# Optional: Bias noise in direction
	offset += direction.normalized() * max_offset * amount * 0.5
```

**Key Parameters:**
- **Light shake:** 0.15-0.25 trauma (gunshot, small impact)
- **Medium shake:** 0.3-0.5 trauma (explosion, heavy melee)
- **Heavy shake:** 0.6-0.8 trauma (boss attack, screen transition)
- **Extreme shake:** 0.9-1.0 trauma (death, ultimate ability)
- **Decay rate:** 1.0-3.0 (higher = faster recovery)
- **Max offset:** 50-150 pixels (depends on resolution)
- **Max rotation:** 0.05-0.15 radians (3-9 degrees)
- **Trauma exponent:** 2.0-3.0 (higher = more dramatic curve)

**Edge Cases:**
- **Trauma stacking:** Multiple explosions add trauma, cap at 1.0
- **Continuous shake sources:** Decay must exceed input to prevent perpetual shake
- **Cutscenes:** Disable shake or use separate cutscene camera
- **UI elements:** Make UI child of separate CanvasLayer, not camera
- **Splitscreen:** Each camera has independent trauma system
- **Pause menu:** Pause shake decay during pause
- **Motion sickness:** Provide accessibility option to reduce/disable

**When NOT to Use:**
- **VR games:** Causes severe nausea - NEVER use
- **Precision aiming sections:** Frustrating when trying to aim carefully
- **Reading text:** Don't shake during dialogue/tutorial
- **Accessibility concerns:** Always provide disable option
- **Overuse:** Every action shaking = visual noise, loses impact
- **Puzzle games:** Distracting and unnecessary

**Examples from Shipped Games:**

1. **Nuclear Throne:** Extreme shake on weapon fire (0.3-0.8 trauma), explosions add 1.0 trauma with screen freeze. Defines the game's visceral feel. Weapons feel powerful even when statistically weak due to shake feedback.

2. **Celeste:** Subtle shake on dash (0.15 trauma), moderate on ground pound (0.4), heavy on Badeline encounters (0.7). Conservative use maintains precision platforming while adding impact to key moments.

3. **Enter the Gungeon:** Gun-specific shake (pistol 0.1, shotgun 0.4, rocket launcher 0.8). Dodge roll adds 0.15 trauma with directional bias. Blanks add 0.6 trauma with radial blast effect. Helps differentiate weapon feel.

4. **Hyper Light Drifter:** Heavy shake on sword slash (0.3), gunshots (0.4-0.6), dash impacts (0.5). Combined with chromatic aberration for distinctive style. Shake is aggressive but short-lived (decay 3.0).

5. **Dead Cells:** Weapon-dependent shake (daggers 0.1, greatswords 0.5), parry successful adds 0.3, critical hits 0.6. Telegraphed enemy attacks shake camera slightly to aid player awareness.

**Platform Considerations:**
- **Desktop (1080p-4K):** 80-120 pixel max offset scales with resolution
- **Mobile:** Reduce max offset by 30-40%, smaller screens amplify perceived shake
- **Switch handheld:** Lower max offset (40-60 pixels), higher in docked mode
- **Console:** Standard implementation, consider rumble sync
- **Ultra-wide monitors:** Scale offset based on aspect ratio
- **High refresh rate:** Noise speed may need adjustment (scale by framerate)

**Godot-Specific Notes:**
- Use `Camera2D` for 2D, `Camera3D` for 3D (same principles)
- `FastNoiseLite` in Godot 4.x, use `OpenSimplexNoise` in Godot 3.x
- Set camera `process_mode = PROCESS_MODE_ALWAYS` to shake during pause if desired
- For UI immunity: UI as `CanvasLayer` child, not camera child
- Consider `smoothing_enabled = true` for less jittery motion (conflicts with shake)
- Godot 4.x: `get_screen_center_position()` replaces 3.x `get_camera_screen_center()`
- Multiply shake by `Engine.time_scale` if using hit pause

**Synergies:**
- **Hit Pause (Technique 95):** Freeze + shake = maximum impact
- **Chromatic Aberration (Technique 99):** Flash aberration on trauma spike
- **Particle Effects (Technique 98):** Spawn particles when trauma added
- **Rumble (Technique 106):** Sync controller vibration with trauma amount
- **Sound Effects:** Play impact sound scaled by trauma amount
- **Damage Scaling (Technique 102):** Higher damage = higher trauma

**Measurement/Profiling:**
- **Performance:** Godot profiler, should be <0.05ms per frame
- **Playtest feedback:** "Is shake too much/too little?" on 1-5 scale
- **Motion sickness reports:** Track accessibility complaints
- **Visual verification:** Record gameplay, watch for nauseating patterns
- **Frequency tracking:** Log shake events per second (should be <5-10)
- **Trauma value graphing:** Plot trauma over time, check for perpetual shake
- **A/B testing:** Same level with different decay rates, measure player preference

**Advanced Techniques:**
```gdscript
# Frequency-based shake (different shake types)
enum ShakeType { LOW_FREQ, HIGH_FREQ, RANDOM }

var current_shake_type: ShakeType = ShakeType.LOW_FREQ

func _apply_shake_with_frequency(delta):
	match current_shake_type:
		ShakeType.LOW_FREQ:
			# Slow, rolling motion (explosions)
			noise.frequency = 1.0
		ShakeType.HIGH_FREQ:
			# Fast, jittery motion (electric shock)
			noise.frequency = 10.0
		ShakeType.RANDOM:
			# Pure random (old-school shake)
			offset = Vector2(
				randf_range(-1, 1),
				randf_range(-1, 1)
			) * max_offset * pow(trauma, trauma_exponent)
			return

	# Apply normal Perlin shake
	_apply_shake(delta)

# Directional bias (push camera in direction)
var directional_bias: Vector2 = Vector2.ZERO
var bias_decay: float = 5.0

func add_directional_shake(amount: float, direction: Vector2):
	add_trauma(amount)
	directional_bias = direction.normalized() * amount

func _process(delta):
	# Decay bias
	directional_bias = directional_bias.lerp(Vector2.ZERO, bias_decay * delta)

	# Apply normal shake
	if trauma > 0.0:
		trauma = max(trauma - decay_rate * delta, 0.0)
		_apply_shake(delta)
		# Add bias on top
		offset += directional_bias * max_offset

# Trauma zones (areas that constantly shake camera)
class_name TraumaZone extends Area2D

@export var trauma_per_second: float = 0.3

func _on_body_entered(body):
	if body is CharacterBody2D:
		_start_trauma()

func _start_trauma():
	while player_in_zone:
		camera.add_trauma(trauma_per_second * get_process_delta_time())
		await get_tree().process_frame

# Impulse shake (sudden spike then decay)
func add_trauma_impulse(amount: float, impulse_duration: float = 0.1):
	var tween = create_tween()
	tween.tween_method(add_trauma, amount, 0.0, impulse_duration)
```

**Accessibility Options:**
```gdscript
# Settings menu options
@export var shake_intensity_multiplier: float = 1.0  # 0.0 = disabled
@export var rotation_shake_enabled: bool = true
@export var max_trauma_cap: float = 1.0  # Limit maximum shake

func add_trauma(amount: float):
	if shake_intensity_multiplier == 0.0:
		return

	var adjusted_trauma = amount * shake_intensity_multiplier
	trauma = min(trauma + adjusted_trauma, max_trauma_cap)
```
