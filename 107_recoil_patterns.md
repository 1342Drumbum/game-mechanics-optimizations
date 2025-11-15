### 107. Recoil Patterns (Predictable vs Random)

**Category:** Game Feel - Combat

**Problem It Solves:** Lack of skill expression and mastery in shooting mechanics. Purely random recoil creates inconsistent gunplay where skilled players can't compensate for weapon behavior, leading to frustration and reduced competitive depth. Affects player retention in competitive shooters by 30-40% according to CSGO/Valorant player surveys.

**Technical Explanation:**
Predictable recoil patterns create a fixed sequence of kick vectors (horizontal/vertical displacement) that repeat identically each time. The weapon follows a predefined path (e.g., rise straight then drift left) that players can learn and counter through muscle memory. Random recoil adds controlled variance around the pattern using noise functions. Implementation stores recoil curve as array of Vector2 offsets, applies rotation/position changes to camera, and optionally adds Perlin noise for feel.

Hybrid approach: 70-80% pattern consistency with 20-30% random variation creates both skill ceiling and organic feel. Pattern resets after firing stops (typically 0.3-0.5s cooldown).

**Algorithmic Complexity:**
- Pattern lookup: O(1) array access
- Random variation: O(1) noise sample
- Smoothing/interpolation: O(1) per frame
- Total: O(1) per shot

**Implementation Pattern:**
```gdscript
# Godot implementation for predictable recoil with random variance
class_name RecoilPattern
extends Node

# Recoil pattern data (normalized -1 to 1)
var pattern: Array[Vector2] = [
	Vector2(0.0, 1.0),    # Shot 1: Straight up
	Vector2(0.0, 0.9),    # Shot 2: Up
	Vector2(-0.2, 0.8),   # Shot 3: Up-left
	Vector2(-0.4, 0.7),   # Shot 4: More left
	Vector2(-0.3, 0.6),   # Shot 5: Left-up
	Vector2(0.1, 0.7),    # Shot 6: Transition right
	Vector2(0.4, 0.8),    # Shot 7: Right-up
	Vector2(0.5, 0.7),    # Shot 8: More right
	Vector2(0.3, 0.6),    # Shot 9: Right-up
	Vector2(0.0, 0.5),    # Shot 10: Center-up
]

@export var recoil_strength: float = 2.0  # Degrees per shot
@export var pattern_randomness: float = 0.2  # 0-1, amount of random variation
@export var recovery_speed: float = 8.0  # How fast recoil recovers
@export var reset_time: float = 0.4  # Seconds before pattern resets

var current_shot: int = 0
var time_since_last_shot: float = 0.0
var accumulated_recoil: Vector2 = Vector2.ZERO
var target_recoil: Vector2 = Vector2.ZERO

func _ready():
	set_physics_process(true)

func _physics_process(delta: float):
	time_since_last_shot += delta

	# Reset pattern if too much time passed
	if time_since_last_shot > reset_time:
		current_shot = 0

	# Smoothly recover from recoil
	accumulated_recoil = accumulated_recoil.lerp(
		Vector2.ZERO,
		recovery_speed * delta
	)

func apply_recoil() -> Vector2:
	# Get pattern vector (loops if exceeds pattern length)
	var pattern_index = current_shot % pattern.size()
	var base_recoil = pattern[pattern_index] * recoil_strength

	# Add random variation using noise
	var random_offset = Vector2(
		randf_range(-pattern_randomness, pattern_randomness),
		randf_range(-pattern_randomness * 0.5, pattern_randomness * 0.5)
	) * recoil_strength

	target_recoil = base_recoil + random_offset
	accumulated_recoil += target_recoil

	current_shot += 1
	time_since_last_shot = 0.0

	return target_recoil

# Camera controller integration
class_name WeaponRecoilController
extends Node3D

@export var camera: Camera3D
@export var recoil_pattern: RecoilPattern
@export var visual_punch: float = 0.3  # Extra punch for feedback

var camera_base_rotation: Vector3 = Vector3.ZERO

func fire_weapon():
	var recoil = recoil_pattern.apply_recoil()

	# Apply to camera rotation (pitch and yaw)
	camera.rotation.x -= deg_to_rad(recoil.y)  # Vertical (pitch)
	camera.rotation.y -= deg_to_rad(recoil.x)  # Horizontal (yaw)

	# Optional: Add visual punch (quick kick that recovers faster)
	_apply_visual_punch(recoil * visual_punch)

func _apply_visual_punch(punch: Vector2):
	var tween = create_tween()
	tween.set_parallel(true)
	tween.tween_property(camera, "rotation:x",
		camera.rotation.x - deg_to_rad(punch.y * 2), 0.05)
	tween.tween_property(camera, "rotation:x",
		camera.rotation.x, 0.2).set_delay(0.05)

# Advanced: Pattern visualization for development
func debug_draw_pattern():
	var start_pos = Vector2(100, 100)
	var scale = 50.0
	var accumulated = Vector2.ZERO

	for i in pattern.size():
		var kick = pattern[i] * scale
		accumulated += kick
		# Draw line to next position
		# (Use CanvasItem or debug overlay)

# Example weapon with different patterns
const RIFLE_PATTERN = [
	Vector2(0, 1.0), Vector2(-0.1, 0.9), Vector2(-0.3, 0.8),
	Vector2(-0.4, 0.7), Vector2(-0.2, 0.7), Vector2(0.1, 0.8),
	Vector2(0.3, 0.8), Vector2(0.4, 0.7), Vector2(0.2, 0.6)
]

const SMG_PATTERN = [
	Vector2(0, 0.6), Vector2(-0.2, 0.6), Vector2(0.2, 0.6),
	Vector2(-0.15, 0.5), Vector2(0.15, 0.5)  # Faster, less predictable
]

const PISTOL_PATTERN = [
	Vector2(0, 1.2), Vector2(0, 1.0), Vector2(-0.1, 0.9)  # Strong, short
]
```

**Key Parameters:**
- **Pattern length:** 7-15 shots (CSGO uses 30, Valorant 10-15)
- **Recoil strength:** 1.5-3.0 degrees per shot for rifles
- **Randomness:** 15-25% variation for competitive, 40-60% for casual
- **Recovery speed:** 6-12 units/sec (faster = easier to control)
- **Reset time:** 0.3-0.5s for competitive, 0.7-1.0s for casual
- **First shot accuracy:** Usually 100% (no recoil on first shot)

**Edge Cases:**
- **Pattern exhaustion:** Loop pattern or clamp to last values
- **Rapid tap-firing:** Each shot should still follow pattern
- **Movement penalty:** Multiply recoil by 1.5-2.5x when moving
- **Crouching bonus:** Reduce recoil by 20-30%
- **Mid-burst stop:** Preserve pattern position for 0.3-0.5s
- **Ads vs hipfire:** Different patterns or multipliers

**When NOT to Use:**
- **Single-shot weapons** (sniper rifles, shotguns) - use simple kick
- **Horror games** wanting unpredictability for tension
- **Casual mobile shooters** - too complex for touch controls
- **Vehicle weapons** - use physics-based shake instead
- **Prototyping phase** - simple random is faster to implement

**Examples from Shipped Games:**

1. **Counter-Strike: Global Offensive:** Iconic AK-47 pattern - 30 bullets, rises then pulls left then right. Pattern is 100% consistent, no randomness. Top players can spray full mag accurately at 30m. Pattern mastery is core skill, creates 1000+ hour skill ceiling. Community created training maps specifically for pattern practice.

2. **Valorant (Riot Games):** Hybrid system - first 3-4 shots highly accurate, then kicks into pattern with 20% randomness. Vandal rises straight for 4 shots, then left, then right. Reduced pattern length (10-15 shots) vs CSGO to lower skill floor. Intentional design to balance skill expression with accessibility.

3. **Apex Legends:** Patterns exist but with 40-50% randomness for casual feel. R-301 has gentle upward pattern, but horizontal variance is significant. Movement penalties are severe (3x recoil multiplier) to encourage positioning over raw aim. Pattern mastery less important than tracking and positioning.

4. **Rainbow Six Siege:** Realistic patterns with strong horizontal drift. Each gun has unique feel - some pull hard left (C8), others are vertical (416-C). First 5-7 shots are controllable, then becomes chaotic. Encourages burst firing over spraying.

5. **Call of Duty: Modern Warfare 2019:** Minimal patterns, mostly random with upward bias. Focus on fast TTK (time-to-kill) rather than recoil control skill. Attachments dramatically change recoil (compensators, grips). Casual-friendly approach prioritizes accessibility.

**Platform Considerations:**
- **PC (Mouse/KB):** Can support complex 30+ shot patterns, pixel-perfect control possible
- **Console (Controller):** Need 30% less recoil or stronger aim assist to compensate for stick precision
- **Mobile (Touch):** Avoid patterns entirely or use auto-compensation, gyro aiming changes viability
- **Crossplay games:** Must balance PC advantage in recoil control (input-based matchmaking or recoil differences)
- **VR:** Physical arm fatigue matters, shorter patterns (5-7 shots) work better

**Godot-Specific Notes:**
- Use `Camera3D.rotation` for recoil, not `look_at()` (breaks player control)
- `lerp()` for recovery feels better than linear interpolation
- Store patterns as `@export var pattern: Array[Vector2]` for per-weapon customization
- Use `Time.get_ticks_msec()` for precise timing if frame rate varies
- AnimationPlayer can handle complex recoil with recovery curves
- Consider using `Curve2D` resource for visual pattern editing in inspector
- Network sync: Send shot count, not recoil values (predict client-side)

**Synergies:**
- **Camera Shake (Technique 73):** Layer shake on top of recoil for extra punch
- **Procedural Recoil (Technique 116):** Combine pattern with procedural gun kick animation
- **Audio Feedback:** Sync shot sound pitch/volume to recoil intensity
- **Muzzle Flash:** Rotate flash direction based on recoil vector
- **Hit Registration:** Client-side prediction must account for recoil offset
- **Tracer Rounds:** Visual feedback shows recoil pattern to player

**Measurement/Profiling:**
- **Pattern consistency:** Record 100 full sprays, measure spread variance
- **Player mastery curve:** Track accuracy improvement over 10/50/100 hours
- **Skill ceiling metrics:** Compare top 1% vs average player spray control
- **Time to competency:** How long until 50% players can control 10-shot burst
- **A/B testing:** Compare retention/satisfaction for different randomness %
- **Input latency:** Recoil response must be <16ms or feels disconnected
- **Performance:** Recoil calculation should be <0.01ms per shot

**Advanced Techniques:**
```gdscript
# Dynamic pattern based on stance/movement
func get_recoil_multiplier() -> float:
	var mult = 1.0
	if player.is_moving: mult *= 2.0
	if player.is_crouching: mult *= 0.7
	if player.is_aiming: mult *= 0.6
	if player.is_jumping: mult *= 3.5
	return mult

# Decay pattern knowledge over time (realistic forgetting)
func apply_pattern_drift(shots_since_fire: int) -> float:
	return 1.0 + (shots_since_fire * 0.1)  # Pattern becomes less reliable

# First shot accuracy bonus
func get_shot_recoil() -> Vector2:
	if current_shot == 0:
		return Vector2.ZERO  # Perfect first shot
	return apply_recoil()
```

---
