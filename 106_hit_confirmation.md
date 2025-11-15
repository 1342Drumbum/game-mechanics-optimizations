### 106. Hit Confirmation (Hitmarker, Sound, Rumble)

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Players uncertain if attacks connected. In fast-paced combat, visual clutter obscures hits. Weak feedback creates disconnect between action and result. Players miss successful attacks due to lack of clear confirmation. Uncertainty reduces satisfaction and makes damage assessment difficult. Critical for ranged combat where projectile travel time delays feedback.

**Technical Explanation:**
Multi-channel synchronized feedback triggered on successful hit: visual (hitmarker sprite/flash), audio (impact sound), haptic (controller rumble), and numeric (damage number). All channels fire simultaneously with intensity scaled to damage value. Hitmarker appears at impact point with 0.2-0.5s lifetime, fades with easing curve. Sound pitch/volume varies by damage (0.8-1.2x pitch, -15dB to 0dB volume). Rumble duration/intensity scales (0.05-0.3s, 0.2-1.0 intensity). System combines Techniques 95, 97, 98, 99, 102 into cohesive multi-sensory response.

**Algorithmic Complexity:**
- O(1) per hit: Spawn marker + play sound + trigger effects
- O(n) active markers where n = on-screen hitmarkers
- CPU: 0.01-0.05ms per hit event
- Memory: ~500 bytes per active hitmarker

**Implementation Pattern:**
```gdscript
# Godot implementation - Hit Confirmation System
class_name HitConfirmation
extends Node

# References to feedback systems
@onready var hit_pause: HitPauseManager = $"/root/HitPause"
@onready var camera_shake: CameraTrauma = $"/root/Camera"
@onready var screen_effects: ScreenEffectsManager = $"/root/ScreenEffects"

# Hitmarker scene
const HITMARKER_SCENE = preload("res://ui/hitmarker.tscn")
const DAMAGE_NUMBER_SCENE = preload("res://ui/damage_number.tscn")

# Sound pools
@export var light_hit_sounds: Array[AudioStream] = []
@export var heavy_hit_sounds: Array[AudioStream] = []
@export var critical_hit_sounds: Array[AudioStream] = []

# Confirmation settings
@export var show_hitmarkers: bool = true
@export var show_damage_numbers: bool = true
@export var play_hit_sounds: bool = true
@export var enable_rumble: bool = true
@export var enable_screen_flash: bool = true

# Hitmarker container (UI layer)
@onready var hitmarker_container: CanvasLayer = $HitmarkerContainer

func confirm_hit(
	damage: float,
	position: Vector2,
	is_critical: bool = false,
	hit_type: String = "normal"
):
	# Calculate intensity from damage (0-1)
	var intensity = _calculate_intensity(damage, is_critical)

	# Trigger all feedback channels simultaneously
	_spawn_hitmarker(position, intensity, is_critical, hit_type)
	_spawn_damage_number(damage, position, is_critical)
	_play_hit_sound(intensity, is_critical)
	_trigger_rumble(intensity, is_critical)
	_trigger_visual_effects(intensity, position, is_critical)

	# Emit signal for other systems
	hit_confirmed.emit(damage, position, is_critical)

func _calculate_intensity(damage: float, is_critical: bool) -> float:
	# Base intensity from damage
	var intensity = clamp(damage / 100.0, 0.0, 1.0)
	intensity = sqrt(intensity)  # Non-linear scaling

	# Critical hits boost intensity
	if is_critical:
		intensity = min(intensity * 1.5, 1.0)

	return intensity

func _spawn_hitmarker(position: Vector2, intensity: float, is_critical: bool, hit_type: String):
	if not show_hitmarkers:
		return

	var hitmarker = HITMARKER_SCENE.instantiate()
	hitmarker_container.add_child(hitmarker)

	# Position (world to screen)
	hitmarker.global_position = position

	# Configure appearance
	hitmarker.configure(intensity, is_critical, hit_type)

func _spawn_damage_number(damage: float, position: Vector2, is_critical: bool):
	if not show_damage_numbers:
		return

	var damage_num = DAMAGE_NUMBER_SCENE.instantiate()
	hitmarker_container.add_child(damage_num)

	damage_num.global_position = position
	damage_num.set_damage(int(damage), is_critical)

func _play_hit_sound(intensity: float, is_critical: bool):
	if not play_hit_sounds:
		return

	# Select sound based on intensity/critical
	var sound_pool: Array[AudioStream]

	if is_critical:
		sound_pool = critical_hit_sounds
	elif intensity > 0.6:
		sound_pool = heavy_hit_sounds
	else:
		sound_pool = light_hit_sounds

	if sound_pool.is_empty():
		return

	# Random variation
	var sound = sound_pool[randi() % sound_pool.size()]

	# Create audio player
	var audio = AudioStreamPlayer.new()
	add_child(audio)
	audio.stream = sound

	# Scale volume and pitch by intensity
	audio.volume_db = lerp(-15.0, 0.0, intensity)
	audio.pitch_scale = lerp(0.9, 1.2, intensity)

	# Play and cleanup
	audio.play()
	audio.finished.connect(func(): audio.queue_free())

func _trigger_rumble(intensity: float, is_critical: bool):
	if not enable_rumble or Input.get_connected_joypads().is_empty():
		return

	# Scale rumble by intensity
	var weak_motor = intensity * 0.4
	var strong_motor = intensity * 0.8

	# Critical hits use strong motor
	if is_critical:
		strong_motor = 1.0

	# Duration scales with intensity
	var duration = lerp(0.05, 0.2, intensity)

	# Trigger rumble on all connected controllers
	for joy_index in Input.get_connected_joypads():
		Input.start_joy_vibration(joy_index, weak_motor, strong_motor, duration)

func _trigger_visual_effects(intensity: float, position: Vector2, is_critical: bool):
	# Hit pause
	var pause_duration = lerp(0.03, 0.15, intensity)
	if is_critical:
		pause_duration *= 1.5
	hit_pause.trigger_pause(pause_duration)

	# Camera shake
	var shake_trauma = lerp(0.1, 0.5, intensity)
	if is_critical:
		shake_trauma *= 1.3
	camera_shake.add_trauma_at_position(position, 500.0, shake_trauma)

	# Screen flash (chromatic aberration)
	if enable_screen_flash:
		var flash_intensity = lerp(0.0, 0.4, intensity)
		if is_critical:
			flash_intensity = 0.6
		screen_effects.flash_chromatic_aberration(flash_intensity)

	# Critical hit exclusive effects
	if is_critical:
		_trigger_critical_effects(position)

func _trigger_critical_effects(position: Vector2):
	# Spawn special particle burst
	var particles = load("res://vfx/critical_burst.tscn").instantiate()
	get_tree().current_scene.add_child(particles)
	particles.global_position = position
	particles.emitting = true

	# Screen flash
	var flash = ColorRect.new()
	flash.color = Color(1, 1, 1, 0.3)
	flash.set_anchors_preset(Control.PRESET_FULL_RECT)
	hitmarker_container.add_child(flash)

	# Fade out flash
	var tween = create_tween()
	tween.tween_property(flash, "modulate:a", 0.0, 0.15)
	tween.tween_callback(flash.queue_free)

# Hitmarker scene implementation
class_name Hitmarker extends Control

@export var sprite: Sprite2D
@export var lifetime: float = 0.3
@export var rise_speed: float = 50.0

var timer: float = 0.0
var start_position: Vector2

func configure(intensity: float, is_critical: bool, hit_type: String):
	start_position = position

	# Scale by intensity
	var size = lerp(1.0, 2.0, intensity)
	sprite.scale = Vector2.ONE * size

	# Color coding
	if is_critical:
		sprite.modulate = Color.YELLOW
		lifetime = 0.5  # Critical markers last longer
	elif hit_type == "headshot":
		sprite.modulate = Color.RED
	elif hit_type == "weak_point":
		sprite.modulate = Color.CYAN
	else:
		sprite.modulate = Color.WHITE

	# Rotation variation
	sprite.rotation = randf_range(-PI/4, PI/4)

func _process(delta):
	timer += delta

	# Rise up
	position.y -= rise_speed * delta

	# Fade out
	var alpha = 1.0 - (timer / lifetime)
	modulate.a = alpha

	# Scale punch (grow then shrink)
	var scale_curve = sin(timer / lifetime * PI)
	sprite.scale = sprite.scale.lerp(Vector2.ONE * 0.5, 1.0 - scale_curve)

	# Cleanup
	if timer >= lifetime:
		queue_free()

# Damage number implementation
class_name DamageNumber extends Label

@export var lifetime: float = 1.0
@export var rise_distance: float = 80.0

var timer: float = 0.0
var start_position: Vector2

func set_damage(damage: int, is_critical: bool):
	text = str(damage)

	# Font size scaling
	if is_critical:
		add_theme_font_size_override("font_size", 48)
		add_theme_color_override("font_color", Color.YELLOW)
		add_theme_color_override("font_outline_color", Color.RED)
		text = "CRIT!\n" + text
	elif damage > 50:
		add_theme_font_size_override("font_size", 32)
		add_theme_color_override("font_color", Color.ORANGE)
	else:
		add_theme_font_size_override("font_size", 24)
		add_theme_color_override("font_color", Color.WHITE)

	start_position = position

func _process(delta):
	timer += delta

	# Ease out rise
	var rise_progress = ease(timer / lifetime, 0.5)
	position = start_position + Vector2(0, -rise_distance * rise_progress)

	# Fade out
	var alpha = 1.0 - (timer / lifetime)
	modulate.a = alpha

	if timer >= lifetime:
		queue_free()

# Advanced: Directional hitmarkers (indicates hit direction)
class_name DirectionalHitmarker extends Hitmarker

func configure_directional(intensity: float, hit_direction: Vector2):
	# Rotate hitmarker to point at hit direction
	var angle = hit_direction.angle()
	sprite.rotation = angle

	# Offset slightly in hit direction
	position += hit_direction.normalized() * 20.0

	configure(intensity, false, "normal")

# Kill confirmation (special feedback for kills)
func confirm_kill(position: Vector2, enemy_name: String):
	# Spawn kill marker
	var kill_marker = Label.new()
	kill_marker.text = "ELIMINATED"
	kill_marker.add_theme_font_size_override("font_size", 36)
	kill_marker.add_theme_color_override("font_color", Color.RED)
	hitmarker_container.add_child(kill_marker)

	kill_marker.global_position = position

	# Animate
	var tween = create_tween()
	tween.set_parallel(true)
	tween.tween_property(kill_marker, "position:y", kill_marker.position.y - 100, 1.5)
	tween.tween_property(kill_marker, "modulate:a", 0.0, 1.5)
	tween.chain().tween_callback(kill_marker.queue_free)

	# Enhanced feedback
	hit_pause.trigger_pause(0.2)
	camera_shake.add_trauma(0.6)
	_play_special_sound("kill_confirm")

	# Extra rumble
	for joy_index in Input.get_connected_joypads():
		Input.start_joy_vibration(joy_index, 0.5, 1.0, 0.3)

# Combo counter
var combo_count: int = 0
var combo_timer: float = 0.0
var combo_window: float = 2.0

func confirm_hit_with_combo(damage: float, position: Vector2, is_critical: bool):
	# Increment combo
	combo_count += 1
	combo_timer = combo_window

	# Standard confirmation
	confirm_hit(damage, position, is_critical)

	# Show combo counter
	if combo_count > 1:
		_show_combo_counter(combo_count, position)

func _process(delta):
	if combo_timer > 0:
		combo_timer -= delta
		if combo_timer <= 0:
			combo_count = 0

func _show_combo_counter(count: int, position: Vector2):
	var combo_label = Label.new()
	combo_label.text = str(count) + "x COMBO"
	combo_label.add_theme_font_size_override("font_size", 28)
	combo_label.add_theme_color_override("font_color", Color.CYAN)
	hitmarker_container.add_child(combo_label)

	combo_label.global_position = position + Vector2(0, -50)

	# Pulse animation
	var tween = create_tween()
	tween.tween_property(combo_label, "scale", Vector2.ONE * 1.5, 0.1)
	tween.tween_property(combo_label, "scale", Vector2.ONE, 0.1)
	tween.tween_property(combo_label, "modulate:a", 0.0, 0.5).set_delay(0.3)
	tween.tween_callback(combo_label.queue_free)
```

**Key Parameters:**
- **Hitmarker lifetime:** 0.2-0.5s (normal), 0.5-0.8s (critical)
- **Hitmarker size:** 16-32px (normal), 32-64px (critical)
- **Rise speed:** 30-80 pixels/sec
- **Sound volume:** -20dB to 0dB (scaled by intensity)
- **Sound pitch:** 0.8-1.3x (varied by intensity)
- **Rumble duration:** 0.05-0.3s
- **Rumble intensity:** 0.2-1.0 (weak/strong motor)
- **Damage number size:** 20-48pt font

**Edge Cases:**
- **Rapid hits:** Limit on-screen hitmarkers to 10-20, queue/skip excess
- **Zero damage:** Show "IMMUNE" or "BLOCKED" instead of hitmarker
- **Overkill damage:** Cap display at enemy max HP, show "OVERKILL"
- **Healing:** Green numbers rising, different sound/marker
- **Friendly fire:** Different color (blue), no positive feedback
- **Miss/dodge:** Show "MISS" or "DODGE" in gray
- **Network lag:** Predict hit, confirm/rollback on server response

**When NOT to Use:**
- **Minimalist UI:** Contradicts design philosophy
- **Realistic games:** Hitmarkers break immersion (mil-sim, survival horror)
- **Puzzle games:** No combat confirmation needed
- **Stealth games:** Audio feedback reveals player
- **Screen clutter:** Already busy UI, don't add more
- **Accessibility:** Provide option to disable for photosensitivity

**Examples from Shipped Games:**

1. **Doom (2016/Eternal):** Aggressive hitmarkers (white X), distinct sound per weapon, heavy screen shake. Glory kills have unique "DOOM GUY" text. Kill confirmation shows demon portrait. Extremely satisfying multi-sensory feedback.

2. **Overwatch:** Character-specific hitmarkers, headshot marker (red X), elimination notification with portrait. Killstreak announcements. Audio cue for final blow. Clear, readable feedback even in chaos.

3. **Destiny 2:** Yellow numbers (normal), orange (crit), white (immune). Precision kill creates "ghost" effect. Distinct headshot sound (ding). Controller rumble synced to damage. Industry-standard implementation.

4. **Borderlands series:** Exaggerated damage numbers (30-100pt font), color-coded by element. Critical hits show "CRITICAL" text. Numbers fly off enemies with physics. Playful, over-the-top feedback.

5. **Enter the Gungeon:** White flash on hit, distinct sound per gun, shells eject. Boss damage shows health bar chunk. Critical hits spark differently. Minimalist but extremely clear.

**Platform Considerations:**
- **Console:** Full rumble support, essential for feel
- **PC:** Rumble if controller present, visual/audio primary
- **Mobile:** Haptic feedback on supported devices, larger hitmarkers
- **Switch:** HD Rumble allows nuanced vibration patterns
- **VR:** Audio/haptic only, avoid screen-space markers
- **Accessibility:** Toggle individual feedback channels

**Godot-Specific Notes:**
- `CanvasLayer` for hitmarkers (UI space, not world space)
- `AudioStreamPlayer` pool for simultaneous sounds
- `Input.start_joy_vibration()` for rumble
- World to screen: `get_viewport().get_camera_2d().get_screen_center_position()`
- `Tween` for hitmarker animations
- Signal architecture: `damage_dealt.connect(_on_damage_dealt)`
- Godot 4.x: `RichTextLabel` for styled damage numbers

**Synergies:**
- **Hit Pause (Technique 95):** Pause enhances visual confirmation
- **Screen Shake (Technique 97):** Shake reinforces impact
- **Particle Effects (Technique 98):** Particles at hit location
- **Damage Scaling (Technique 102):** All feedback scales together
- **Chromatic Aberration (Technique 99):** Flash on critical hit
- **Sound Design:** Layered impact sounds create richness

**Measurement/Profiling:**
- **Readability testing:** Can players see hits in busy combat?
- **Performance:** Monitor hitmarker count, cap at 20-30
- **Audio mixing:** Hit sounds shouldn't drown out important audio
- **A/B testing:** Compare hit satisfaction with/without confirmation
- **Accessibility feedback:** Ensure colorblind modes work
- **Network testing:** Latency compensation for online games
- **Playtests:** "Did you feel your attacks connecting?" yes/no

**Advanced Techniques:**
```gdscript
# Stacking damage numbers (combine rapid hits)
var damage_accumulator: Dictionary = {}  # enemy_id: accumulated_damage
var accumulator_timers: Dictionary = {}

func confirm_hit_stacked(damage: float, enemy_id: int, position: Vector2):
	if enemy_id not in damage_accumulator:
		damage_accumulator[enemy_id] = 0.0
		accumulator_timers[enemy_id] = 0.0

	damage_accumulator[enemy_id] += damage
	accumulator_timers[enemy_id] = 0.1  # 100ms window

func _process(delta):
	for enemy_id in accumulator_timers.keys():
		accumulator_timers[enemy_id] -= delta

		if accumulator_timers[enemy_id] <= 0:
			# Display accumulated damage
			var total_damage = damage_accumulator[enemy_id]
			_spawn_damage_number(total_damage, enemy_position, false)

			damage_accumulator.erase(enemy_id)
			accumulator_timers.erase(enemy_id)

# Floating combat text (MMO-style)
class_name FloatingCombatText extends Label

enum TextType { DAMAGE, HEAL, BUFF, DEBUFF, MISS }

func configure(value: float, type: TextType):
	match type:
		TextType.DAMAGE:
			text = str(int(value))
			add_theme_color_override("font_color", Color.RED)
		TextType.HEAL:
			text = "+" + str(int(value))
			add_theme_color_override("font_color", Color.GREEN)
		TextType.BUFF:
			text = "BUFF!"
			add_theme_color_override("font_color", Color.CYAN)
		TextType.MISS:
			text = "MISS"
			add_theme_color_override("font_color", Color.GRAY)

# Hitmarker variation (headshot, bodyshot, weak point)
const HITMARKER_TEXTURES = {
	"normal": preload("res://ui/hitmarker_normal.png"),
	"headshot": preload("res://ui/hitmarker_headshot.png"),
	"weak_point": preload("res://ui/hitmarker_weakpoint.png"),
}

func _spawn_hitmarker(position: Vector2, intensity: float, is_critical: bool, hit_type: String):
	var hitmarker = HITMARKER_SCENE.instantiate()
	hitmarker_container.add_child(hitmarker)

	# Set texture based on type
	hitmarker.sprite.texture = HITMARKER_TEXTURES.get(hit_type, HITMARKER_TEXTURES["normal"])

	hitmarker.global_position = position
	hitmarker.configure(intensity, is_critical, hit_type)

# Procedural rumble patterns
func create_rumble_pattern(pattern_type: String):
	match pattern_type:
		"pulse":
			# Quick pulses
			for i in range(3):
				Input.start_joy_vibration(0, 0.5, 0.5, 0.05)
				await get_tree().create_timer(0.1).timeout

		"crescendo":
			# Build up vibration
			for i in range(5):
				var intensity = float(i) / 5.0
				Input.start_joy_vibration(0, intensity, intensity, 0.1)
				await get_tree().create_timer(0.08).timeout
```
