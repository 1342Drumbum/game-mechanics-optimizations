### 102. Damage Feedback Scaling

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** All hits feel identical regardless of damage magnitude. Players can't distinguish between 5 damage and 500 damage without reading numbers. Critical hits don't feel impactful. Monotonous feedback reduces combat satisfaction and makes damage difficult to assess intuitively. Players rely on UI numbers instead of visceral feedback.

**Technical Explanation:**
Scale all feedback systems (hit pause, screen shake, particle count, sound pitch/volume) proportionally to damage dealt. Uses non-linear scaling (square root or logarithmic) to prevent overwhelming feedback at high damage. Formula: `feedback_intensity = base_intensity + (damage / max_damage)^exponent * max_intensity`. Exponent typically 0.5-0.8 creates perceptible differences without extreme values. Combines multiple feedback channels synchronized to same intensity value for coherent experience.

**Algorithmic Complexity:**
- O(1) calculation: Single power function per damage event
- O(1) triggering: Calls to feedback systems
- CPU: <0.01ms per damage calculation
- Memory: Negligible (~4 bytes for intensity value)

**Implementation Pattern:**
```gdscript
# Godot implementation - Damage Feedback Scaling System
class_name DamageScaling
extends Node

# Reference to feedback systems
@onready var hit_pause: HitPauseManager = $"/root/HitPause"
@onready var camera_shake: CameraTrauma = $"/root/Camera"
@onready var particles: SyncedParticleEmitter = $ParticleEmitter
@onready var sound_manager: SoundManager = $"/root/SoundManager"

# Scaling parameters
@export var min_damage: float = 1.0
@export var max_damage: float = 200.0
@export var scaling_exponent: float = 0.6  # 0.5 = sqrt, 1.0 = linear

# Base and max values for each feedback type
const FEEDBACK_RANGES = {
	"hit_pause": {"min": 0.03, "max": 0.20},      # 2-12 frames
	"screen_shake": {"min": 0.1, "max": 0.8},      # trauma amount
	"particle_count": {"min": 5, "max": 50},       # particle count
	"sound_volume": {"min": -10.0, "max": 0.0},    # dB
	"sound_pitch": {"min": 0.9, "max": 1.3},       # pitch scale
	"chromatic_aberration": {"min": 0.0, "max": 0.6},
}

# Damage type multipliers
const TYPE_MULTIPLIERS = {
	"normal": 1.0,
	"critical": 1.5,
	"weak_point": 1.3,
	"overkill": 2.0,
}

func apply_damage_feedback(damage: float, type: String = "normal", position: Vector2 = Vector2.ZERO):
	# Calculate base intensity (0-1)
	var intensity = _calculate_intensity(damage)

	# Apply type multiplier
	intensity *= TYPE_MULTIPLIERS.get(type, 1.0)
	intensity = clamp(intensity, 0.0, 1.0)

	# Trigger all feedback systems with scaled values
	_trigger_hit_pause(intensity)
	_trigger_screen_shake(intensity, position)
	_trigger_particles(intensity, position)
	_trigger_sound(intensity, type)
	_trigger_screen_effects(intensity)

	# Optional: Rumble for gamepad
	if Input.get_connected_joypads().size() > 0:
		_trigger_rumble(intensity)

func _calculate_intensity(damage: float) -> float:
	# Normalize damage to 0-1 range
	var normalized = clamp((damage - min_damage) / (max_damage - min_damage), 0.0, 1.0)

	# Apply non-linear scaling
	# sqrt (0.5) makes small differences more noticeable
	# linear (1.0) is proportional
	# squared (2.0) emphasizes high damage
	return pow(normalized, scaling_exponent)

func _trigger_hit_pause(intensity: float):
	var duration = lerp(
		FEEDBACK_RANGES["hit_pause"]["min"],
		FEEDBACK_RANGES["hit_pause"]["max"],
		intensity
	)
	hit_pause.trigger_pause(duration)

func _trigger_screen_shake(intensity: float, position: Vector2):
	var trauma = lerp(
		FEEDBACK_RANGES["screen_shake"]["min"],
		FEEDBACK_RANGES["screen_shake"]["max"],
		intensity
	)

	# Distance-based shake if position provided
	if position != Vector2.ZERO:
		camera_shake.add_trauma_at_position(position, 500.0, trauma)
	else:
		camera_shake.add_trauma(trauma)

func _trigger_particles(intensity: float, position: Vector2):
	var count = int(lerp(
		FEEDBACK_RANGES["particle_count"]["min"],
		FEEDBACK_RANGES["particle_count"]["max"],
		intensity
	))

	if position != Vector2.ZERO:
		particles.emit_particles(position, intensity)

func _trigger_sound(intensity: float, type: String):
	# Select sound based on intensity threshold
	var sound_name = "hit_light"
	if intensity > 0.7:
		sound_name = "hit_heavy"
	elif intensity > 0.4:
		sound_name = "hit_medium"

	# Critical hits get special sound
	if type == "critical":
		sound_name = "hit_critical"

	# Scale volume and pitch
	var volume = lerp(
		FEEDBACK_RANGES["sound_volume"]["min"],
		FEEDBACK_RANGES["sound_volume"]["max"],
		intensity
	)

	var pitch = lerp(
		FEEDBACK_RANGES["sound_pitch"]["min"],
		FEEDBACK_RANGES["sound_pitch"]["max"],
		intensity
	)

	sound_manager.play_sound(sound_name, volume, pitch)

func _trigger_screen_effects(intensity: float):
	var ca_intensity = lerp(
		FEEDBACK_RANGES["chromatic_aberration"]["min"],
		FEEDBACK_RANGES["chromatic_aberration"]["max"],
		intensity
	)

	# Flash chromatic aberration
	ScreenEffects.flash_chromatic_aberration(ca_intensity)

func _trigger_rumble(intensity: float):
	# Gamepad rumble (0-1 for weak motor, strong motor)
	var weak_motor = intensity * 0.5
	var strong_motor = intensity

	Input.start_joy_vibration(0, weak_motor, strong_motor, 0.1 + intensity * 0.2)

# Advanced: Combo multiplier
class_name ComboScaling extends DamageScaling

var combo_count: int = 0
var combo_timer: float = 0.0
var combo_window: float = 2.0  # Reset after 2s

func _process(delta):
	if combo_timer > 0:
		combo_timer -= delta
		if combo_timer <= 0:
			_reset_combo()

func apply_damage_feedback(damage: float, type: String = "normal", position: Vector2 = Vector2.ZERO):
	# Increment combo
	combo_count += 1
	combo_timer = combo_window

	# Calculate base intensity
	var intensity = _calculate_intensity(damage)

	# Apply combo multiplier (up to 1.5x at 10 hits)
	var combo_multiplier = 1.0 + min(combo_count * 0.05, 0.5)
	intensity *= combo_multiplier

	# Call parent
	super.apply_damage_feedback(damage, type, position)

	# Show combo number
	_show_combo_counter(combo_count, intensity)

func _reset_combo():
	combo_count = 0

# Damage number display with scaling
class_name DamageNumbers extends Node2D

const DamageLabel = preload("res://ui/damage_label.tscn")

func show_damage(damage: float, position: Vector2, intensity: float):
	var label = DamageLabel.instantiate()
	add_child(label)

	label.global_position = position
	label.text = str(int(damage))

	# Scale font size by intensity
	var font_size = int(lerp(16, 48, intensity))
	label.add_theme_font_size_override("font_size", font_size)

	# Color by intensity (white → yellow → red)
	var color = Color.WHITE
	if intensity > 0.7:
		color = Color.RED
	elif intensity > 0.4:
		color = Color.YELLOW

	label.modulate = color

	# Animate: rise and fade
	var tween = create_tween()
	tween.set_parallel(true)
	tween.tween_property(label, "position:y", label.position.y - 50, 1.0)
	tween.tween_property(label, "modulate:a", 0.0, 1.0)
	tween.chain().tween_callback(label.queue_free)
```

**Key Parameters:**
- **Scaling exponent:** 0.5 (sqrt - emphasizes low damage), 0.7-0.8 (balanced), 1.0 (linear)
- **Min damage threshold:** 1-10 (below this uses minimum feedback)
- **Max damage cap:** 100-500 (above this uses maximum feedback)
- **Critical multiplier:** 1.3-2.0x normal feedback
- **Combo multiplier:** 1.05-1.1x per hit, cap at 1.5-2.0x
- **Particle scaling:** 5-50 particles (10x range)
- **Shake scaling:** 0.1-0.8 trauma (8x range)

**Edge Cases:**
- **Zero damage:** Show "IMMUNE" or "MISS", no feedback
- **Overkill damage:** Cap feedback at max even if damage exceeds max_damage
- **Negative damage (healing):** Different feedback color/sound, positive feel
- **Percentage-based damage:** Scale based on target's max HP, not absolute value
- **Multi-hit:** Stagger feedback or sum damage before triggering
- **Damage over time:** Lower intensity continuous feedback vs burst
- **Environmental damage:** Different scaling curve (falls, spikes, etc.)

**When NOT to Use:**
- **Percentage-based games:** 1% damage vs 50% more meaningful than absolute values
- **Puzzle games:** Damage is binary (solved/not solved)
- **Turn-based strategy:** Feedback timing less critical
- **Text-heavy RPGs:** Players read numbers, not feel
- **Accessibility concerns:** Intense feedback at high damage may cause issues
- **Stealth games:** Dramatic feedback reveals player position

**Examples from Shipped Games:**

1. **Borderlands series:** Damage numbers scale in size and color (white → yellow → orange → red). Critical hits get larger numbers + "CRITICAL" text + special sound. Overkill damage shows in purple. Font size ranges from 20pt to 100pt+.

2. **Devil May Cry 5:** Combo counter with style rating (D, C, B, A, S, SS, SSS). Higher ratings increase hit pause, particle effects, and screen shake intensity. SSS rank attacks have 2-3x feedback intensity of D rank.

3. **Hades:** Weak attacks: small hit flash. Strong attacks: larger flash + shake. Critical hits: red flash + "CRITICAL" popup + extended pause. Duo boons create unique feedback signatures. Damage scaling is very clear even without numbers.

4. **Dead Cells:** Critical hits flash screen white, extended hit pause (12 frames vs 3), larger particle burst, distinct sound. Combo counter increases feedback intensity. 100-hit combos have dramatic feedback.

5. **Monster Hunter World:** Different weapon damage ranges normalized - Greatsword 200 damage gets same feedback as Dual Blades 20 damage (proportional to weapon). Critical hits spark differently. Stagger/break effects have unique feedback.

**Platform Considerations:**
- **Console:** Full haptic feedback support (rumble scaling)
- **PC:** Rumble support varies, prioritize visual/audio scaling
- **Mobile:** Haptic feedback on supported devices, reduce visual intensity
- **Switch:** HD Rumble allows nuanced vibration scaling
- **Accessibility:** Provide option to disable/reduce scaled feedback
- **Performance:** Extreme damage shouldn't drop frames

**Godot-Specific Notes:**
- Use `@export` ranges for easy designer tuning
- Store scaling curves in `Curve` resource for visual editing
- `Input.start_joy_vibration()` for rumble scaling
- Autoload singleton for global damage feedback manager
- Signal-based architecture: `damage_dealt.emit(amount, type, position)`
- Use `RichTextLabel` for colored/scaled damage numbers
- Godot 4.x: `PhysicalBone3D` can react to damage with physics

**Synergies:**
- **Hit Pause (Technique 95):** Scale pause duration by damage
- **Screen Shake (Technique 97):** Scale trauma by damage
- **Particle Effects (Technique 98):** Scale particle count/velocity
- **Chromatic Aberration (Technique 99):** Scale effect intensity
- **Hit Confirmation (Technique 106):** All confirmations scale together
- **Sound Design:** Pitch/volume scaling creates cohesive feedback

**Measurement/Profiling:**
- **Damage distribution analysis:** Plot damage values over playtest session
- **Perceptibility testing:** Can players distinguish 20 vs 40 damage by feel?
- **Curve tuning:** Adjust exponent until differences feel "right"
- **Playtest surveys:** "Can you tell damage differences without reading numbers?"
- **A/B testing:** Linear vs sqrt vs log scaling, measure preference
- **Performance:** Monitor worst-case feedback (max damage spike)
- **Balance:** Ensure high damage doesn't create visual spam

**Advanced Techniques:**
```gdscript
# Contextual scaling (damage relative to target health)
func calculate_contextual_intensity(damage: float, target_max_hp: float) -> float:
	# 10 damage to 100 HP enemy = 0.1 intensity
	# 10 damage to 20 HP enemy = 0.5 intensity (more significant)
	var health_percent = damage / target_max_hp
	return clamp(health_percent, 0.0, 1.0)

# Multi-tier feedback system
func get_feedback_tier(intensity: float) -> String:
	if intensity < 0.2:
		return "minimal"  # Weak attacks
	elif intensity < 0.5:
		return "standard"  # Normal attacks
	elif intensity < 0.8:
		return "heavy"  # Strong attacks
	else:
		return "extreme"  # Critical/finisher

	# Each tier triggers different effect sets

# Damage type-specific scaling curves
const TYPE_CURVES = {
	"physical": 0.6,    # Standard sqrt-ish curve
	"fire": 0.8,        # More linear (dramatic)
	"poison": 0.4,      # Emphasis on small ticks
	"explosive": 1.2,   # Exponential (burst damage)
}

func calculate_intensity_typed(damage: float, type: String) -> float:
	var normalized = clamp(damage / max_damage, 0.0, 1.0)
	var exponent = TYPE_CURVES.get(type, 0.6)
	return pow(normalized, exponent)

# Stacking damage feedback (rapid hits)
var damage_accumulator: float = 0.0
var accumulator_timer: float = 0.0
const ACCUMULATOR_WINDOW: float = 0.1  # 100ms window

func apply_damage_stacked(damage: float, type: String, position: Vector2):
	# Accumulate rapid hits
	damage_accumulator += damage
	accumulator_timer = ACCUMULATOR_WINDOW

	# Wait for window to expire
	if accumulator_timer > 0:
		await get_tree().create_timer(accumulator_timer).timeout

	# Apply feedback for total accumulated damage
	apply_damage_feedback(damage_accumulator, type, position)
	damage_accumulator = 0.0

# Curve resource for visual editing
@export var damage_curve: Curve  # Edit in inspector

func calculate_intensity_curve(damage: float) -> float:
	var normalized = clamp(damage / max_damage, 0.0, 1.0)
	return damage_curve.sample(normalized)  # Use authored curve

# Logarithmic scaling (large damage ranges)
func calculate_intensity_log(damage: float) -> float:
	# Useful for games with 1-1000000 damage ranges
	var log_damage = log(damage + 1.0) / log(max_damage + 1.0)
	return clamp(log_damage, 0.0, 1.0)

# Distance-falloff scaling
func apply_damage_feedback_with_falloff(
	damage: float,
	impact_position: Vector2,
	player_position: Vector2,
	max_distance: float = 1000.0
):
	var base_intensity = _calculate_intensity(damage)

	# Reduce intensity based on distance
	var distance = impact_position.distance_to(player_position)
	var falloff = 1.0 - clamp(distance / max_distance, 0.0, 1.0)

	var final_intensity = base_intensity * falloff

	apply_damage_feedback(damage, "normal", impact_position)
```

**Scaling Curve Visualization:**
```gdscript
# Debug tool to visualize scaling curves
func _draw():
	if Engine.is_editor_hint():
		var size = Vector2(400, 200)
		draw_rect(Rect2(Vector2.ZERO, size), Color.WHITE, false, 2.0)

		# Draw curve
		var points: PackedVector2Array = []
		for i in range(int(size.x)):
			var damage = (i / size.x) * max_damage
			var intensity = _calculate_intensity(damage)
			var y = size.y - (intensity * size.y)
			points.append(Vector2(i, y))

		draw_polyline(points, Color.GREEN, 2.0)

		# Draw linear reference
		var linear_points = PackedVector2Array([
			Vector2(0, size.y),
			Vector2(size.x, 0)
		])
		draw_polyline(linear_points, Color.RED.darkened(0.5), 1.0, true)
```
