### 113. Doppler Effect

**Category:** Game Feel - Audio

**Problem It Solves:** Lack of motion perception and speed feedback from fast-moving objects. Without Doppler shift, racing games feel slower, projectiles lack weight, and fly-bys lack excitement. Studies show Doppler effect increases perceived speed by 25-35% in racing games. Creates cognitive dissonance when fast-moving sounds don't pitch-shift like real-world experience, reducing immersion and realism.

**Technical Explanation:**
Doppler effect shifts audio pitch based on relative velocity between sound source and listener. Approaching objects pitch up (blue shift), receding objects pitch down (red shift). Formula: f' = f × (v + v_listener) / (v + v_source), where f = original frequency, v = speed of sound (343 m/s). Implementation calculates velocity vector dot product to get approach/recede speed, converts to pitch multiplier, then smoothly interpolates pitch over time to avoid discontinuities.

Clamp pitch to 0.5-2.0x range (octave down/up) for musical preservation. Apply only to diegetic sounds (in-world), not UI or music. Use velocity history to smooth jitter from physics frame variance.

**Algorithmic Complexity:**
- Velocity calculation: O(1) vector math
- Pitch calculation: O(1) formula evaluation
- Interpolation: O(1) lerp
- Total: O(1) per sound source, <0.01ms

**Implementation Pattern:**
```gdscript
# Godot Doppler effect system
class_name DopplerEffect
extends Node3D

# Configuration
@export var enable_doppler: bool = true
@export var speed_of_sound: float = 343.0  # m/s in air
@export var doppler_intensity: float = 1.0  # Multiplier for effect strength (0-2)
@export var min_pitch: float = 0.5  # Minimum pitch shift (half speed)
@export var max_pitch: float = 2.0  # Maximum pitch shift (double speed)
@export var smooth_speed: float = 10.0  # Pitch interpolation speed

# Components
@export var audio_player: AudioStreamPlayer3D
@export var listener: Node3D  # Camera3D or AudioListener3D
@export var source_body: Node3D  # RigidBody3D, CharacterBody3D, etc.

# State tracking
var previous_position: Vector3
var current_velocity: Vector3 = Vector3.ZERO
var smoothed_velocity: Vector3 = Vector3.ZERO
var target_pitch: float = 1.0
var current_pitch: float = 1.0
var base_pitch: float = 1.0  # Original pitch scale

# Velocity history for smoothing
var velocity_history: Array[Vector3] = []
var history_size: int = 5

func _ready():
	if audio_player:
		base_pitch = audio_player.pitch_scale

	previous_position = global_position
	set_physics_process(true)

func _physics_process(delta: float):
	if not enable_doppler or not audio_player or not listener:
		return

	# Calculate source velocity
	var current_position = global_position
	current_velocity = (current_position - previous_position) / delta
	previous_position = current_position

	# Smooth velocity to reduce jitter
	_smooth_velocity(current_velocity)

	# Calculate Doppler shift
	var doppler_pitch = _calculate_doppler_shift()

	# Smooth pitch transitions
	target_pitch = doppler_pitch
	current_pitch = lerp(current_pitch, target_pitch, smooth_speed * delta)

	# Apply pitch to audio player
	audio_player.pitch_scale = base_pitch * current_pitch

func _smooth_velocity(new_velocity: Vector3):
	# Add to history
	velocity_history.append(new_velocity)
	if velocity_history.size() > history_size:
		velocity_history.remove_at(0)

	# Average velocity over history
	var sum = Vector3.ZERO
	for vel in velocity_history:
		sum += vel

	smoothed_velocity = sum / velocity_history.size()

func _calculate_doppler_shift() -> float:
	# Get listener velocity (if listener is moving)
	var listener_velocity = Vector3.ZERO
	if listener is CharacterBody3D or listener is RigidBody3D:
		listener_velocity = listener.velocity if "velocity" in listener else Vector3.ZERO

	# Vector from source to listener
	var to_listener = listener.global_position - global_position
	var distance = to_listener.length()

	if distance < 0.1:
		return 1.0  # Avoid division by zero

	var direction = to_listener.normalized()

	# Velocity component along direction to listener
	# Positive = approaching, negative = receding
	var source_radial_velocity = -smoothed_velocity.dot(direction)
	var listener_radial_velocity = listener_velocity.dot(direction)

	# Doppler formula: f' = f * (v + v_listener) / (v + v_source)
	var numerator = speed_of_sound + listener_radial_velocity
	var denominator = speed_of_sound + source_radial_velocity

	# Prevent division issues
	if denominator <= 0.01:
		denominator = 0.01

	var pitch_shift = numerator / denominator

	# Apply intensity multiplier and clamp
	pitch_shift = 1.0 + (pitch_shift - 1.0) * doppler_intensity
	pitch_shift = clamp(pitch_shift, min_pitch, max_pitch)

	return pitch_shift

# Alternative: Simplified Doppler (faster, less accurate)
func _calculate_doppler_simple() -> float:
	var to_listener = listener.global_position - global_position
	var approach_speed = -smoothed_velocity.dot(to_listener.normalized())

	# Simple linear approximation
	var pitch_factor = 1.0 + (approach_speed / speed_of_sound) * doppler_intensity

	return clamp(pitch_factor, min_pitch, max_pitch)

# Advanced: Distance-based Doppler strength
func _calculate_doppler_with_distance() -> float:
	var distance = global_position.distance_to(listener.global_position)

	# Stronger effect at close range
	var distance_factor = remap(distance, 0, 50, 1.0, 0.3)
	distance_factor = clamp(distance_factor, 0.3, 1.0)

	var base_doppler = _calculate_doppler_shift()

	# Blend towards neutral (1.0) at distance
	return lerp(1.0, base_doppler, distance_factor)

# Debug visualization
func debug_draw_velocity():
	# Draw velocity vector
	print("Velocity: %v, Pitch: %.2f" % [smoothed_velocity, current_pitch])

# Presets for common scenarios
func setup_for_vehicle():
	doppler_intensity = 1.0
	min_pitch = 0.7
	max_pitch = 1.4
	smooth_speed = 8.0

func setup_for_projectile():
	doppler_intensity = 1.2  # Exaggerated for effect
	min_pitch = 0.6
	max_pitch = 1.8
	smooth_speed = 15.0  # Fast response

func setup_for_flyby():
	doppler_intensity = 1.5  # Very pronounced
	min_pitch = 0.5
	max_pitch = 2.0
	smooth_speed = 12.0

# Example: Racing game engine sound
class_name RacingCarAudio
extends Node3D

@export var engine_audio: AudioStreamPlayer3D
@export var doppler: DopplerEffect
@export var car_body: VehicleBody3D

var engine_base_pitch: float = 1.0
var rpm_pitch: float = 1.0

func _ready():
	doppler.setup_for_vehicle()
	engine_base_pitch = engine_audio.pitch_scale

func _process(_delta: float):
	# Engine pitch from RPM
	var rpm = car_body.engine_force  # Simplified
	rpm_pitch = remap(rpm, 0, 100, 0.8, 1.4)

	# Combine RPM pitch with Doppler
	engine_audio.pitch_scale = engine_base_pitch * rpm_pitch * doppler.current_pitch

# Example: Bullet whizz-by sound
class_name BulletDoppler
extends Node3D

@export var whizz_audio: AudioStreamPlayer3D
@export var bullet_velocity: Vector3 = Vector3(0, 0, -100)  # m/s

var doppler: DopplerEffect

func _ready():
	# Create doppler instance
	doppler = DopplerEffect.new()
	doppler.audio_player = whizz_audio
	doppler.listener = get_viewport().get_camera_3d()
	doppler.setup_for_projectile()
	add_child(doppler)

func _physics_process(delta: float):
	# Move bullet
	global_position += bullet_velocity * delta

	# Doppler automatically calculates from position change
	# Approaching: high pitch, passing: neutral, receding: low pitch

# Advanced: Sonic boom effect
func check_for_sonic_boom() -> bool:
	var speed = smoothed_velocity.length()
	if speed > speed_of_sound:
		# Object exceeds speed of sound
		_trigger_sonic_boom()
		return true
	return false

func _trigger_sonic_boom():
	# Play special boom sound
	var boom = AudioStreamPlayer3D.new()
	get_parent().add_child(boom)
	boom.stream = preload("res://audio/sonic_boom.ogg")
	boom.global_position = global_position
	boom.play()
	boom.finished.connect(func(): boom.queue_free())

# Advanced: Per-sound-type Doppler settings
class DopplerProfile:
	var name: String
	var intensity: float = 1.0
	var min_pitch: float = 0.5
	var max_pitch: float = 2.0
	var smooth_speed: float = 10.0

static func get_profile(sound_type: String) -> DopplerProfile:
	var profiles = {
		"vehicle": DopplerProfile.new(),
		"projectile": DopplerProfile.new(),
		"aircraft": DopplerProfile.new(),
	}

	profiles["vehicle"].intensity = 0.8
	profiles["vehicle"].min_pitch = 0.7
	profiles["vehicle"].max_pitch = 1.3

	profiles["projectile"].intensity = 1.2
	profiles["projectile"].min_pitch = 0.6
	profiles["projectile"].max_pitch = 1.8
	profiles["projectile"].smooth_speed = 15.0

	profiles["aircraft"].intensity = 1.5
	profiles["aircraft"].min_pitch = 0.5
	profiles["aircraft"].max_pitch = 2.0
	profiles["aircraft"].smooth_speed = 6.0

	return profiles.get(sound_type, profiles["vehicle"])

# Integration with audio pooling
class DopplerAudioPool:
	var doppler_instances: Array[DopplerEffect] = []
	var pool_size: int = 20

	func get_doppler_instance() -> DopplerEffect:
		for doppler in doppler_instances:
			if not doppler.audio_player or not doppler.audio_player.playing:
				return doppler

		# Create new if pool exhausted
		var new_doppler = DopplerEffect.new()
		doppler_instances.append(new_doppler)
		return new_doppler

# Performance optimization: LOD for Doppler
func update_doppler_lod(distance: float):
	if distance < 20.0:
		# Close: Full quality
		enable_doppler = true
		smooth_speed = 10.0
	elif distance < 50.0:
		# Medium: Reduced quality
		enable_doppler = true
		smooth_speed = 5.0
		history_size = 3
	else:
		# Far: Disable
		enable_doppler = false

# Network multiplayer considerations
func _get_network_velocity() -> Vector3:
	# Use interpolated network position for smoother Doppler
	# Avoid jitter from network updates
	if has_node("NetworkPositionSync"):
		var sync = get_node("NetworkPositionSync")
		return sync.interpolated_velocity
	return current_velocity
```

**Key Parameters:**
- **Speed of sound:** 343 m/s (air), 1480 m/s (water), 5960 m/s (steel)
- **Doppler intensity:** 0.8-1.0 (realistic), 1.2-1.5 (exaggerated/arcade)
- **Pitch range:** 0.5-2.0x (musical octave), 0.7-1.4x (subtle)
- **Smooth speed:** 5-10 (smooth), 15-20 (responsive)
- **Velocity history:** 3-7 samples for smoothing
- **Min speed threshold:** >5 m/s to activate (avoid idle jitter)

**Edge Cases:**
- **Near-zero velocity:** Disable or use previous pitch to avoid flicker
- **Supersonic speeds:** Clamp pitch, trigger sonic boom effect
- **Rapid direction changes:** Smooth velocity is critical
- **Listener moving:** Include listener velocity in calculation
- **Audio streaming:** Pitch shift may affect compressed audio quality
- **Looping sounds:** Ensure loop points work at all pitch ranges
- **Network jitter:** Use interpolated positions, not raw network data

**When NOT to Use:**
- **Static sound sources:** No velocity = no Doppler
- **Slow-moving objects:** <5 m/s threshold, effect imperceptible
- **Non-diegetic audio:** UI, music, narration should never Doppler
- **Horror games:** May break tension with arcade-like effect
- **Top-down 2D:** Visual disconnect from audio makes it confusing
- **Mobile low-end:** Pitch shifting has higher CPU cost

**Examples from Shipped Games:**

1. **Gran Turismo 7:** Highly realistic Doppler - approaching cars pitch up dramatically, receding pitch down smoothly. Intensity varies by camera angle (chase cam less pronounced, trackside exaggerated). Combines with RPM-based engine pitch for complex audio. Different for AI cars vs player car. Strength: 0.8-1.0 (realistic) with smooth 8.0. Critical for speed perception and immersion. Used in replays for dramatic fly-bys.

2. **Battlefield series:** Pronounced Doppler on jets, helicopters, bullets. Bullet whizz-by uses aggressive 1.5 intensity for "holy crap that was close" effect. Vehicle fly-bys at 200+ km/h have dramatic pitch shift. Distant explosions have subtle Doppler based on shockwave speed. Combined with distance attenuation and occlusion. Enhances battlefield chaos and sense of danger.

3. **Star Wars: Squadrons:** Exaggerated Doppler (intensity 1.8) for arcade space combat feel. TIE fighters have iconic pitch shift when passing. Blaster bolts use Doppler despite being light-speed (artistic license). Sound design balances realism vs. Star Wars aesthetic. Vader's TIE has unique Doppler profile (lower pitch shift, menacing). Space combat needs exaggeration since no real reference.

4. **Need for Speed series:** Aggressive Doppler for opponent cars (intensity 1.2-1.5), minimal for player car (0.5-0.7) to avoid disorientation. Drift-bys have pronounced effect combined with tire squeal. Nitrous boost affects Doppler calculation (increased base pitch + Doppler). Traffic cars use simplified Doppler (less CPU). Camera mode dramatically affects perception (bumper cam vs chase cam).

5. **Half-Life 2:** Subtle Doppler on Combine gunships and dropships (intensity 0.6-0.8). Gravity Gun-launched objects have Doppler for weight perception. Trains and vehicles use realistic Doppler. Minimal on weapons to avoid player confusion. Design philosophy: "Realistic unless it hurts gameplay." Critical moments (helicopter chase) amp up Doppler for drama.

**Platform Considerations:**
- **PC/Console:** Full quality Doppler with 10-20 simultaneous sources
- **Mobile:** Reduce to 5-10 sources, simpler calculation, lower smooth rate
- **VR:** More pronounced to enhance speed perception and presence
- **Pitch shift cost:** ~0.01-0.02ms per source (negligible)
- **Physics tick rate:** Higher rates (60Hz+) give smoother velocity calc
- **Audio engine:** Some engines have built-in Doppler (Unity, Wwise)

**Godot-Specific Notes:**
- Use `AudioStreamPlayer3D.pitch_scale` for pitch modification
- Calculate velocity in `_physics_process()` for consistent timing
- `CharacterBody3D.velocity` or `RigidBody3D.linear_velocity` for bodies
- Manual velocity calc: `(current_pos - prev_pos) / delta` for other nodes
- Vector math: `Vector3.dot()` for radial velocity component
- Smoothing: Use running average of 3-7 samples to reduce jitter
- Network: Use `MultiplayerSynchronizer` interpolated positions
- Performance: Doppler calculation is <0.01ms per source

**Synergies:**
- **Distance Attenuation:** Combine with Doppler for realistic falloff
- **Audio Occlusion (Technique 112):** Doppler + occlusion = ultra-realism
- **Engine Sounds:** Layer RPM pitch with Doppler pitch
- **Camera Systems:** Different Doppler profiles per camera mode
- **Slow-Motion:** Scale Doppler calculations with time scale
- **Reverb/Echo:** Doppler affects reflections timing too
- **Particle VFX:** Match visual speed with audio Doppler

**Measurement/Profiling:**
- **Pitch accuracy:** Compare to real-world Doppler recordings
- **Performance cost:** <0.01ms per source target
- **Smooth vs. responsive:** Balance smoothing vs reaction time
- **Player perception:** Survey whether players notice speed increase
- **A/B testing:** Compare engagement with/without Doppler
- **Velocity jitter:** Measure variance, should be <5% per frame
- **CPU budget:** Total Doppler system <0.2ms for 20 sources

**Advanced Techniques:**
```gdscript
# Relative Doppler (both source and listener moving)
func calculate_relative_doppler(source_vel: Vector3, listener_vel: Vector3,
                                source_pos: Vector3, listener_pos: Vector3) -> float:
	var direction = (listener_pos - source_pos).normalized()
	var source_radial = source_vel.dot(direction)
	var listener_radial = listener_vel.dot(direction)

	var pitch = (speed_of_sound - listener_radial) / (speed_of_sound - source_radial)
	return clamp(pitch, min_pitch, max_pitch)

# Predictive Doppler (account for audio latency)
func calculate_predictive_doppler(latency_ms: float) -> float:
	var latency_sec = latency_ms / 1000.0
	var predicted_position = global_position + smoothed_velocity * latency_sec
	# Calculate Doppler using predicted position
	return _calculate_doppler_at_position(predicted_position)

# Environmental Doppler (different speed of sound)
func get_speed_of_sound_for_environment(env: String) -> float:
	match env:
		"air": return 343.0
		"water": return 1480.0
		"vacuum": return 999999.0  # No Doppler in space (unless Star Wars)
		"dense_fog": return 320.0  # Slightly slower
	return 343.0

# Dynamic intensity based on camera mode
func get_doppler_intensity_for_camera(mode: String) -> float:
	match mode:
		"first_person": return 0.6  # Subtle for player
		"chase": return 1.0         # Normal
		"cinematic": return 1.5     # Exaggerated
		"trackside": return 1.8     # Very pronounced
	return 1.0

# Doppler with acceleration consideration
func calculate_advanced_doppler() -> float:
	var acceleration = (current_velocity - smoothed_velocity) / get_physics_process_delta_time()
	var acceleration_factor = acceleration.length() / 50.0  # Scale factor
	var base_doppler = _calculate_doppler_shift()

	# Acceleration adds to perceived speed change
	return base_doppler * (1.0 + acceleration_factor * 0.1)
```

---
