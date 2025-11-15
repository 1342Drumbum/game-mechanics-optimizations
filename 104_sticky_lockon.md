### 104. Sticky/Soft Lock-On Targeting

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Aiming in fast-paced action games feels imprecise and frustrating. Players miss attacks due to slight input errors. Manual aiming creates constant micro-adjustments breaking flow. Without assistance, enemies moving perpendicular to player make melee combat tedious. Players spend more time fighting controls than enemies.

**Technical Explanation:**
Subtly adjust player facing direction toward nearby enemies within "sticky zone" (typically 30-60 degree cone, 50-200 unit radius). When enemy enters zone, gradually rotate player toward enemy over 0.1-0.3s using exponential interpolation. Unlike hard lock-on (Dark Souls), rotation is invisible and cancelable by player input. Uses weighted scoring: closer enemies and those in crosshair center score higher. Formula: `rotation_strength = distance_weight * angle_weight * (1 - player_input_magnitude)`. When player actively moves stick, disable or reduce sticky behavior.

**Algorithmic Complexity:**
- O(n) where n = enemies in range
- O(n log n) with spatial partitioning (query nearby enemies)
- O(1) per frame: Rotation interpolation
- CPU: 0.01-0.1ms per frame (depends on enemy count)
- Memory: ~50 bytes per tracked target

**Implementation Pattern:**
```gdscript
# Godot implementation - Sticky Lock-On System
class_name StickyLockOn
extends Node

@export var character: CharacterBody2D
@export var detection_radius: float = 150.0
@export var detection_angle: float = 60.0  # Degrees (total cone width)
@export var rotation_speed: float = 10.0   # How fast to rotate toward target
@export var rotation_strength: float = 0.7 # 0-1, how much to assist
@export var min_input_to_disable: float = 0.3  # Input magnitude to cancel assist

var current_target: Node2D = null
var potential_targets: Array[Node2D] = []

# Detection area
@onready var detection_area: Area2D = $DetectionArea

func _ready():
	_setup_detection_area()

func _setup_detection_area():
	# Create circular detection area
	var circle_shape = CircleShape2D.new()
	circle_shape.radius = detection_radius

	var collision_shape = CollisionShape2D.new()
	collision_shape.shape = circle_shape

	detection_area.add_child(collision_shape)
	detection_area.body_entered.connect(_on_target_entered)
	detection_area.body_exited.connect(_on_target_exited)

func _physics_process(delta):
	# Check for player input
	var input_vector = Input.get_vector("move_left", "move_right", "move_up", "move_down")
	var input_magnitude = input_vector.length()

	# Disable sticky if player is actively controlling direction
	if input_magnitude > min_input_to_disable:
		current_target = null
		return

	# Find best target
	_update_target_selection()

	# Apply sticky rotation
	if current_target:
		_apply_sticky_rotation(delta, input_magnitude)

func _update_target_selection():
	if potential_targets.is_empty():
		current_target = null
		return

	# Score each target
	var best_target: Node2D = null
	var best_score: float = -INF

	for target in potential_targets:
		var score = _calculate_target_score(target)
		if score > best_score:
			best_score = score
			best_target = target

	current_target = best_target

func _calculate_target_score(target: Node2D) -> float:
	var to_target = target.global_position - character.global_position
	var distance = to_target.length()
	var angle_to_target = rad_to_deg(abs(character.rotation - to_target.angle()))

	# Normalize angle to 0-180
	if angle_to_target > 180:
		angle_to_target = 360 - angle_to_target

	# Check if target is within detection cone
	if angle_to_target > detection_angle / 2:
		return -INF  # Outside cone

	# Distance score (closer = better)
	var distance_score = 1.0 - clamp(distance / detection_radius, 0.0, 1.0)

	# Angle score (more centered = better)
	var angle_score = 1.0 - (angle_to_target / (detection_angle / 2))

	# Combined score (weighted)
	var score = distance_score * 0.4 + angle_score * 0.6

	return score

func _apply_sticky_rotation(delta: float, input_magnitude: float):
	if not current_target:
		return

	# Calculate desired rotation
	var to_target = current_target.global_position - character.global_position
	var target_angle = to_target.angle()

	# Interpolate rotation
	var current_angle = character.rotation
	var angle_diff = _angle_difference(current_angle, target_angle)

	# Reduce strength based on player input
	var effective_strength = rotation_strength * (1.0 - input_magnitude / min_input_to_disable)

	# Apply rotation
	var rotation_amount = angle_diff * rotation_speed * effective_strength * delta
	character.rotation += rotation_amount

func _angle_difference(from_angle: float, to_angle: float) -> float:
	# Calculate shortest rotation direction
	var diff = to_angle - from_angle
	diff = fmod(diff + PI, TAU) - PI
	return diff

func _on_target_entered(body: Node2D):
	if body.is_in_group("enemies"):
		potential_targets.append(body)

func _on_target_exited(body: Node2D):
	potential_targets.erase(body)
	if current_target == body:
		current_target = null

# Advanced: Attack-specific lock-on
class_name AttackLockOn extends StickyLockOn

var is_attacking: bool = false
var attack_target: Node2D = null

func start_attack():
	is_attacking = true

	# Lock onto current target for duration of attack
	if current_target:
		attack_target = current_target

func end_attack():
	is_attacking = false
	attack_target = null

func _physics_process(delta):
	if is_attacking and attack_target:
		# Stronger lock during attack
		_apply_attack_lock(delta)
	else:
		# Normal sticky behavior
		super._physics_process(delta)

func _apply_attack_lock(delta):
	# Rotate strongly toward locked target during attack
	var to_target = attack_target.global_position - character.global_position
	var target_angle = to_target.angle()

	# Fast, strong rotation during attack
	character.rotation = lerp_angle(character.rotation, target_angle, 15.0 * delta)

	# Optional: Lunge toward target
	var lunge_strength = 200.0
	character.velocity += to_target.normalized() * lunge_strength * delta

# Melee auto-aim (Home toward enemies)
class_name MeleeAutoAim extends Node

@export var character: CharacterBody2D
@export var homing_strength: float = 500.0
@export var homing_radius: float = 100.0

var is_attacking: bool = false
var attack_target: Node2D = null

func perform_attack():
	is_attacking = true

	# Find closest enemy
	attack_target = _find_closest_enemy(homing_radius)

func _physics_process(delta):
	if is_attacking and attack_target:
		_apply_homing(delta)

func _apply_homing(delta):
	# Move character toward target during attack
	var to_target = attack_target.global_position - character.global_position
	var distance = to_target.length()

	if distance > 10.0:  # Minimum distance
		var homing_velocity = to_target.normalized() * homing_strength
		character.velocity += homing_velocity * delta

func _find_closest_enemy(radius: float) -> Node2D:
	var enemies = get_tree().get_nodes_in_group("enemies")
	var closest: Node2D = null
	var closest_distance: float = INF

	for enemy in enemies:
		var distance = character.global_position.distance_to(enemy.global_position)
		if distance < radius and distance < closest_distance:
			closest = enemy
			closest_distance = distance

	return closest

# Projectile auto-aim (Bullet magnetism)
class_name ProjectileAutoAim extends Node

@export var detection_radius: float = 50.0
@export var homing_strength: float = 0.3  # 0-1, how much to curve

func spawn_projectile(projectile: Node2D, direction: Vector2):
	# Find nearby enemies
	var target = _find_best_target(projectile.global_position, direction)

	if target:
		# Adjust projectile direction slightly toward target
		var to_target = target.global_position - projectile.global_position
		var adjusted_direction = direction.lerp(to_target.normalized(), homing_strength)
		projectile.direction = adjusted_direction
	else:
		projectile.direction = direction

func _find_best_target(from_position: Vector2, intended_direction: Vector2) -> Node2D:
	# Find enemies near intended trajectory
	var enemies = get_tree().get_nodes_in_group("enemies")
	var best_target: Node2D = null
	var best_score: float = -INF

	for enemy in enemies:
		var to_enemy = enemy.global_position - from_position
		var distance = to_enemy.length()

		if distance > detection_radius:
			continue

		# Score based on alignment with intended direction
		var alignment = intended_direction.dot(to_enemy.normalized())

		if alignment < 0.7:  # Must be roughly in firing direction
			continue

		var score = alignment * (1.0 - distance / detection_radius)

		if score > best_score:
			best_score = score
			best_target = enemy

	return best_target
```

**Key Parameters:**
- **Detection radius:** 100-200 units (melee), 50-100 units (ranged)
- **Detection angle:** 45-90 degrees (narrow), 90-180 degrees (wide)
- **Rotation speed:** 5-15 (subtle), 15-30 (aggressive)
- **Rotation strength:** 0.3-0.6 (subtle assist), 0.7-0.9 (strong assist)
- **Input threshold:** 0.2-0.4 (how much input disables assist)
- **Attack homing:** 200-800 units/sec lunge speed
- **Projectile magnetism:** 0.1-0.3 direction adjustment

**Edge Cases:**
- **Multiple equidistant targets:** Prioritize most centered target
- **Target behind player:** Don't snap 180°, rotate smoothly
- **Target death:** Immediately clear lock, find new target
- **Platformer vs top-down:** Different angle calculations (2D vs isometric)
- **Flying/jumping enemies:** Predict position if moving fast
- **Invincible enemies:** Exclude from targeting system
- **Friendly fire:** Filter by team/faction, don't lock onto allies

**When NOT to Use:**
- **Competitive shooters:** Players expect pure skill-based aiming
- **Tactical games:** Precise positioning is core gameplay
- **Puzzle games:** No combat targeting needed
- **Realistic military sims:** Auto-aim breaks immersion
- **Mouse aim on PC:** Mouse is already precise, assist conflicts
- **Fighting games:** Manual spacing is fundamental skill

**Examples from Shipped Games:**

1. **Devil May Cry series:** Aggressive sticky lock-on during combos (0.8-0.9 strength). Attack animations home toward enemies. Can cycle targets with right stick. Essential for juggling multiple enemies. One of the best implementations.

2. **God of War (2018):** Subtle sticky behavior (0.5 strength) during normal movement. Strong lock-on (0.9 strength) during Leviathan axe throws. Distance-based scoring prioritizes close threats. Seamless, invisible assist.

3. **Doom (2016):** "Glory Kill" has extreme homing - player lunges 5-10m toward stunned enemies. Shotgun has subtle bullet magnetism (projectiles curve slightly). Feels powerful without being obvious.

4. **Horizon Zero Dawn:** Bow aiming has slight sticky when aiming near weak points. Spear melee attacks home strongly (500-800 unit/sec). Different assist strength per weapon type. Very polished implementation.

5. **Bayonetta:** Strong lock-on during Witch Time (slow-mo). Normal combat has moderate sticky (0.6-0.7). Can manually lock with trigger. Combo attacks have built-in homing. Extremely responsive feel.

**Platform Considerations:**
- **Controller (console/PC):** Strong sticky needed (analog stick imprecise)
- **Mouse (PC):** Minimal or no sticky (mouse is precise)
- **Touch (mobile):** Moderate sticky, wider detection cone
- **Gyro aiming:** Reduce sticky strength (gyro is precise)
- **Accessibility:** Provide stronger assist option for motor impairments
- **Difficulty settings:** Easy mode = stronger sticky, Hard mode = minimal

**Godot-Specific Notes:**
- Use `Area2D` with circle/cone shape for detection
- `body_entered/exited` signals for target tracking
- `lerp_angle()` for smooth rotation interpolation
- `Vector2.angle_to()` for angle calculations
- Group enemies with `add_to_group("enemies")`
- Consider `RayCast2D` for line-of-sight check
- Godot 4.x: `CharacterBody2D.velocity` for homing movement

**Synergies:**
- **Camera System:** Camera frames locked target
- **Particle Effects (Technique 98):** Spawn at lock-on acquisition
- **Hit Confirmation (Technique 106):** Stronger feedback when locked
- **Animation System:** Attack animations snap toward target
- **Damage Scaling (Technique 102):** Lock-on hits deal bonus damage
- **Sound Design:** Audio cue when lock-on activates

**Measurement/Profiling:**
- **Hit rate tracking:** Compare with/without sticky, measure improvement
- **Playtest surveys:** "Does aiming feel good?" rating 1-10
- **Performance:** Target detection should be <0.1ms per frame
- **Visibility testing:** Players shouldn't notice assist consciously
- **Input response:** Verify player input overrides instantly
- **Enemy count scaling:** Test with 1, 10, 50 enemies in range
- **A/B testing:** Different strength values, measure player preference

**Advanced Techniques:**
```gdscript
# Predictive targeting (lead moving targets)
class_name PredictiveTarget extends StickyLockOn

func _calculate_target_score(target: Node2D) -> float:
	# Get target velocity if available
	var target_velocity = Vector2.ZERO
	if target is CharacterBody2D:
		target_velocity = target.velocity

	# Predict future position
	var prediction_time = 0.3  # 300ms ahead
	var predicted_position = target.global_position + target_velocity * prediction_time

	# Score based on predicted position instead
	var to_predicted = predicted_position - character.global_position
	var angle_to_predicted = rad_to_deg(abs(character.rotation - to_predicted.angle()))

	# ... rest of scoring

# Priority targeting (certain enemy types prioritized)
const PRIORITY_SCORES = {
	"boss": 2.0,
	"elite": 1.5,
	"ranged": 1.3,
	"melee": 1.0,
}

func _calculate_target_score(target: Node2D) -> float:
	var base_score = super._calculate_target_score(target)

	# Apply priority multiplier
	var priority = 1.0
	for type in PRIORITY_SCORES:
		if target.is_in_group(type):
			priority = PRIORITY_SCORES[type]
			break

	return base_score * priority

# Cone visualization (debug)
func _draw():
	if Engine.is_editor_hint() or OS.is_debug_build():
		# Draw detection cone
		var cone_angle = deg_to_rad(detection_angle / 2)
		var segments = 16

		var points: PackedVector2Array = [Vector2.ZERO]

		for i in range(segments + 1):
			var angle = lerp(-cone_angle, cone_angle, float(i) / segments)
			var dir = Vector2(cos(angle), sin(angle)) * detection_radius
			points.append(dir)

		points.append(Vector2.ZERO)

		draw_polyline(points, Color.YELLOW, 2.0, true)

		# Draw current target indicator
		if current_target:
			var to_target = to_local(current_target.global_position)
			draw_circle(to_target, 10.0, Color.RED)

# Dynamic detection parameters (based on game state)
func update_detection_params(player_state: String):
	match player_state:
		"combat":
			detection_radius = 200.0
			rotation_strength = 0.7
		"exploration":
			detection_radius = 100.0
			rotation_strength = 0.3
		"aiming":
			detection_radius = 300.0
			rotation_strength = 0.9

# Line-of-sight filtering
func _calculate_target_score(target: Node2D) -> float:
	# Only target enemies with clear line of sight
	var space_state = get_world_2d().direct_space_state
	var query = PhysicsRayQueryParameters2D.create(
		character.global_position,
		target.global_position
	)
	query.exclude = [character]

	var result = space_state.intersect_ray(query)

	if result and result.collider != target:
		return -INF  # Blocked by obstacle

	return super._calculate_target_score(target)

# Gradual lock-on (builds up over time)
var lock_strength: float = 0.0
var lock_build_rate: float = 2.0

func _physics_process(delta):
	super._physics_process(delta)

	if current_target:
		# Build lock strength
		lock_strength = min(lock_strength + lock_build_rate * delta, 1.0)
	else:
		# Decay lock strength
		lock_strength = max(lock_strength - lock_build_rate * 2.0 * delta, 0.0)

	# Use lock_strength to scale rotation_strength
	var effective_strength = rotation_strength * lock_strength
```
