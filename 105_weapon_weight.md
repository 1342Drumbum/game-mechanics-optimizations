### 105. Weapon Weight Simulation

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** All weapons feel identical despite different sizes/types. Dagger feels same as greatsword. Players can't distinguish weapon classes by feel alone. Instant attack startup/recovery makes weapons feel weightless. Without weight simulation, weapon choice becomes pure stat comparison lacking tactile differentiation.

**Technical Explanation:**
Vary timing parameters (anticipation, action, recovery), movement penalties, camera effects, and animation curves per weapon weight class. Heavy weapons have longer anticipation (0.2-0.4s), reduced movement speed during swing (0.4-0.7x), stronger hit pause (0.15-0.25s), and exaggerated follow-through. Light weapons have minimal anticipation (0.05-0.1s), maintained movement speed (0.9-1.0x), light hit pause (0.05s), snappy recovery. Uses animation curves (ease-in for heavy, linear for light) and procedural effects (camera shake, player velocity modification, arm recoil).

**Algorithmic Complexity:**
- O(1) per weapon: Parameter lookup from weapon data
- O(1) per frame: Apply movement/animation modifiers
- CPU: <0.01ms per weapon update
- Memory: ~200 bytes per weapon (timing + modifier parameters)

**Implementation Pattern:**
```gdscript
# Godot implementation - Weapon Weight System
class_name WeaponWeight
extends Node

enum WeightClass { LIGHT, MEDIUM, HEAVY, ULTRA_HEAVY }

# Weapon data resource
class_name WeaponData extends Resource

@export var weapon_name: String = "Sword"
@export var weight_class: WeightClass = WeightClass.MEDIUM
@export var base_damage: float = 25.0

# Timing parameters (seconds)
@export var anticipation_time: float = 0.15
@export var action_time: float = 0.25
@export var recovery_time: float = 0.2

# Movement modifiers (multipliers)
@export var movement_penalty: float = 0.7    # Speed during swing
@export var turn_rate_penalty: float = 0.5   # Rotation during swing

# Feel parameters
@export var hit_pause_duration: float = 0.1
@export var camera_shake_intensity: float = 0.3
@export var recoil_strength: float = 10.0    # Knockback on swing

# Animation curves
@export var swing_curve: Curve  # Attack acceleration curve

# Weapon controller
class_name WeaponController extends Node

@export var character: CharacterBody2D
@export var weapon_sprite: Sprite2D
@export var animation_player: AnimationPlayer

var current_weapon: WeaponData
var is_attacking: bool = false
var attack_phase: String = "idle"  # idle, anticipation, action, recovery
var phase_timer: float = 0.0

var base_move_speed: float = 200.0
var base_turn_rate: float = 5.0

func equip_weapon(weapon: WeaponData):
	current_weapon = weapon
	weapon_sprite.texture = load(weapon.sprite_path)

func start_attack():
	if is_attacking or not current_weapon:
		return

	is_attacking = true
	attack_phase = "anticipation"
	phase_timer = current_weapon.anticipation_time

	# Play anticipation animation
	_play_phase_animation("anticipation")

	# Apply anticipation effects
	_apply_anticipation_effects()

func _physics_process(delta):
	if is_attacking:
		_update_attack(delta)

	# Apply weight-based movement modifiers
	if is_attacking and attack_phase in ["anticipation", "action"]:
		_apply_movement_modifiers()

func _update_attack(delta):
	phase_timer -= delta

	if phase_timer <= 0.0:
		_advance_attack_phase()

func _advance_attack_phase():
	match attack_phase:
		"anticipation":
			attack_phase = "action"
			phase_timer = current_weapon.action_time
			_execute_attack()

		"action":
			attack_phase = "recovery"
			phase_timer = current_weapon.recovery_time
			_play_phase_animation("recovery")

		"recovery":
			attack_phase = "idle"
			is_attacking = false

func _execute_attack():
	# Play action animation
	_play_phase_animation("action")

	# Spawn hitbox
	_spawn_hitbox()

	# Apply recoil
	var recoil_dir = Vector2.from_angle(character.rotation)
	character.velocity -= recoil_dir * current_weapon.recoil_strength

	# Camera shake
	get_node("/root/Camera").add_trauma(current_weapon.camera_shake_intensity)

func _apply_movement_modifiers():
	# Reduce movement speed during attack
	var speed_mult = current_weapon.movement_penalty

	if Input.is_action_pressed("move"):
		character.velocity *= speed_mult

	# Reduce turn rate
	var turn_mult = current_weapon.turn_rate_penalty
	var turn_input = Input.get_axis("turn_left", "turn_right")
	character.rotation += turn_input * base_turn_rate * turn_mult * get_physics_process_delta_time()

func _apply_anticipation_effects():
	# Heavy weapons: Pull back
	match current_weapon.weight_class:
		WeightClass.HEAVY, WeightClass.ULTRA_HEAVY:
			# Squash character
			character.scale = Vector2(0.9, 1.1)

			# Pull weapon back
			weapon_sprite.offset.x = -15

			# Slight camera zoom
			get_node("/root/Camera").zoom *= 0.95

func _play_phase_animation(phase: String):
	var anim_name = current_weapon.weapon_name.to_lower() + "_" + phase
	if animation_player.has_animation(anim_name):
		animation_player.play(anim_name)

# Preset weapon configurations
class WeaponPresets:
	static func create_dagger() -> WeaponData:
		var weapon = WeaponData.new()
		weapon.weapon_name = "Dagger"
		weapon.weight_class = WeightClass.LIGHT

		# Fast, snappy
		weapon.anticipation_time = 0.05
		weapon.action_time = 0.15
		weapon.recovery_time = 0.1

		# Minimal penalties
		weapon.movement_penalty = 0.95
		weapon.turn_rate_penalty = 0.9

		# Light feedback
		weapon.hit_pause_duration = 0.05
		weapon.camera_shake_intensity = 0.15
		weapon.recoil_strength = 5.0

		return weapon

	static func create_greatsword() -> WeaponData:
		var weapon = WeaponData.new()
		weapon.weapon_name = "Greatsword"
		weapon.weight_class = WeightClass.HEAVY

		# Slow, deliberate
		weapon.anticipation_time = 0.3
		weapon.action_time = 0.4
		weapon.recovery_time = 0.35

		# Heavy penalties
		weapon.movement_penalty = 0.5
		weapon.turn_rate_penalty = 0.3

		# Strong feedback
		weapon.hit_pause_duration = 0.2
		weapon.camera_shake_intensity = 0.7
		weapon.recoil_strength = 30.0

		return weapon

	static func create_spear() -> WeaponData:
		var weapon = WeaponData.new()
		weapon.weapon_name = "Spear"
		weapon.weight_class = WeightClass.MEDIUM

		# Medium timing
		weapon.anticipation_time = 0.12
		weapon.action_time = 0.25
		weapon.recovery_time = 0.18

		# Moderate penalties (thrust has less turn penalty)
		weapon.movement_penalty = 0.8
		weapon.turn_rate_penalty = 0.7

		# Medium feedback
		weapon.hit_pause_duration = 0.1
		weapon.camera_shake_intensity = 0.3
		weapon.recoil_strength = 15.0

		return weapon

# Advanced: Momentum system
class_name WeaponMomentum extends WeaponController

var swing_momentum: float = 0.0
var max_momentum: float = 1.0

func _execute_attack():
	super._execute_attack()

	# Build momentum based on weight
	var momentum_gain = 0.1
	match current_weapon.weight_class:
		WeightClass.LIGHT:
			momentum_gain = 0.05
		WeightClass.MEDIUM:
			momentum_gain = 0.15
		WeightClass.HEAVY:
			momentum_gain = 0.25
		WeightClass.ULTRA_HEAVY:
			momentum_gain = 0.4

	swing_momentum = min(swing_momentum + momentum_gain, max_momentum)

func _update_attack(delta):
	super._update_attack(delta)

	# Decay momentum when not attacking
	if not is_attacking:
		swing_momentum = max(swing_momentum - delta * 0.5, 0.0)

	# Heavy weapons gain damage/knockback from momentum
	if attack_phase == "action":
		var momentum_bonus = 1.0 + swing_momentum * 0.5
		# Apply to damage calculation

# Charge attack system
class_name ChargeAttack extends WeaponController

var charge_time: float = 0.0
var max_charge_time: float = 2.0
var is_charging: bool = false

func start_charge():
	is_charging = true
	charge_time = 0.0

func _process(delta):
	if is_charging:
		charge_time = min(charge_time + delta, max_charge_time)
		_update_charge_visuals()

func release_charge():
	is_charging = false

	# Scale attack based on charge
	var charge_percent = charge_time / max_charge_time

	# Heavy weapons benefit more from charge
	var damage_mult = 1.0
	match current_weapon.weight_class:
		WeightClass.HEAVY, WeightClass.ULTRA_HEAVY:
			damage_mult = 1.0 + charge_percent * 2.0  # Up to 3x damage
		WeightClass.MEDIUM:
			damage_mult = 1.0 + charge_percent * 1.0  # Up to 2x damage
		WeightClass.LIGHT:
			damage_mult = 1.0 + charge_percent * 0.5  # Up to 1.5x damage

	# Longer anticipation for charged attacks
	if charge_percent > 0.5:
		current_weapon.anticipation_time *= 1.5

	start_attack()

func _update_charge_visuals():
	var charge_percent = charge_time / max_charge_time

	# Weapon glows/shakes
	weapon_sprite.modulate = Color.WHITE.lerp(Color.YELLOW, charge_percent)

	# Character strains
	character.scale = Vector2.ONE.lerp(Vector2(0.85, 1.15), charge_percent)
```

**Key Parameters:**

**Light Weapons (Dagger, Rapier):**
- Anticipation: 0.03-0.08s (2-5 frames)
- Action: 0.1-0.2s (6-12 frames)
- Recovery: 0.08-0.15s (5-9 frames)
- Movement penalty: 0.9-1.0x (minimal)
- Hit pause: 0.03-0.05s
- Shake: 0.1-0.2 trauma

**Medium Weapons (Sword, Axe):**
- Anticipation: 0.1-0.15s (6-9 frames)
- Action: 0.2-0.3s (12-18 frames)
- Recovery: 0.15-0.25s (9-15 frames)
- Movement penalty: 0.7-0.8x
- Hit pause: 0.08-0.12s
- Shake: 0.3-0.4 trauma

**Heavy Weapons (Greatsword, Hammer):**
- Anticipation: 0.25-0.4s (15-24 frames)
- Action: 0.35-0.5s (21-30 frames)
- Recovery: 0.3-0.5s (18-30 frames)
- Movement penalty: 0.4-0.6x
- Hit pause: 0.15-0.25s
- Shake: 0.6-0.8 trauma

**Edge Cases:**
- **Weapon switching:** Reset attack state, blend to new idle
- **Interrupted attack:** Stun during anticipation cancels, during action stagger
- **Dual wielding:** Each hand independent timing or synchronized
- **Underwater/slowed:** Exaggerate weight penalties further
- **Strength buffs:** Reduce weight penalties, not timing
- **Animation canceling:** Light weapons allow, heavy weapons prevent
- **Fall damage while attacking:** Ultra-heavy weapons cancel fall

**When NOT to Use:**
- **Guns/ranged:** Different feel system (recoil, accuracy)
- **Magic/spells:** Use cast time instead of weight
- **Fast-paced shooters:** All weapons must feel responsive
- **Puzzle games:** No combat mechanics
- **Racing games:** Not applicable
- **Intentionally weightless aesthetic:** Floaty, dreamlike games

**Examples from Shipped Games:**

1. **Dark Souls series:** Definitive weapon weight system. Dagger: 0.3s total, Greatsword: 1.2s total, Ultra Greatsword: 1.8s total. Movement speed while attacking: dagger 90%, ultra 30%. Recovery time prevents panic rolling. Weight is core to game balance.

2. **Monster Hunter series:** Greatsword charge attacks take 2-3 seconds, root player in place. Dual blades allow dodging during combos. Each weapon class has unique movement profile. Weight feel is identity of weapon.

3. **God of War (2018):** Leviathan Axe recall has momentum - catches feels weighty. Heavy axe throws have 0.3s windup, rapid throws 0.1s. Camera shake scales with charge time. Extremely satisfying weight.

4. **Hades:** Fists: 0.05s startup, free movement. Bow: 0.2s charge, movement 0.5x. Adamant Rail: continuous fire, movement 0.8x. Shield: 0.15s startup, can block during recovery. Clear weapon differentiation.

5. **Dead Cells:** Daggers: 0.05s anticipation, combo-friendly. Broadswords: 0.15s anticipation, roots player briefly. Hammers: 0.3s anticipation, charge system. Weight communicates damage output.

**Platform Considerations:**
- **60fps:** Frame-perfect timing critical, design in frames
- **30fps:** Heavier weapons feel worse, reduce timing 20%
- **Variable framerate:** Use delta time, not frame counts
- **Input lag:** Account for display lag + weight = total delay
- **Mobile touch:** Reduce heavy weapon penalties (less precision)
- **Controller vibration:** Sync rumble intensity with weight

**Godot-Specific Notes:**
- Store weapon data in `.tres` Resource files
- Use `@export` for easy designer tweaking
- `AnimationPlayer` with different curves per weapon
- `CharacterBody2D.velocity` modifications for recoil
- Weapon switching: `AnimationTree` blend between weapon idle states
- State machine for attack phases (anticipation/action/recovery)
- Godot 4.x: Physics material friction for weapon drag

**Synergies:**
- **Hit Pause (Technique 95):** Scale pause by weapon weight
- **Screen Shake (Technique 97):** Heavier = more shake
- **Anticipation Frames (Technique 100):** Core to weight feel
- **Follow-Through (Technique 101):** Heavy weapons overshoot more
- **Damage Scaling (Technique 102):** Weight correlates with damage
- **Animation Blending (Technique 103):** Blend weapon equip animations

**Measurement/Profiling:**
- **Playtest surveys:** Can players identify weapon weight blindfolded (by feel)?
- **Timing verification:** Record attacks, measure frame-accurate timings
- **Balance testing:** Heavy weapons should deal 2-3x damage of light
- **Preference polling:** Some players prefer light, others heavy (both valid)
- **Input response:** Total time from input to damage = startup time
- **Performance:** Negligible CPU cost (<0.01ms)
- **A/B testing:** Different timing curves, measure satisfaction

**Advanced Techniques:**
```gdscript
# Dynamic weight (weapon gets heavier with use - fatigue)
class_name WeaponFatigue extends WeaponController

var fatigue: float = 0.0
var max_fatigue: float = 10.0
var fatigue_decay_rate: float = 0.5

func _execute_attack():
	super._execute_attack()

	# Build fatigue
	var fatigue_gain = 0.5
	match current_weapon.weight_class:
		WeightClass.HEAVY, WeightClass.ULTRA_HEAVY:
			fatigue_gain = 1.5

	fatigue = min(fatigue + fatigue_gain, max_fatigue)

func _process(delta):
	# Decay fatigue when not attacking
	if not is_attacking:
		fatigue = max(fatigue - fatigue_decay_rate * delta, 0.0)

	# Apply fatigue penalties
	var fatigue_percent = fatigue / max_fatigue
	current_weapon.movement_penalty *= (1.0 - fatigue_percent * 0.3)
	current_weapon.anticipation_time *= (1.0 + fatigue_percent * 0.5)

# Combo system with variable weight
class_name ComboWeightSystem extends WeaponController

var combo_count: int = 0

func _execute_attack():
	super._execute_attack()
	combo_count += 1

	# Each combo hit gets faster/lighter
	if combo_count <= 3:
		current_weapon.anticipation_time *= 0.8
		current_weapon.action_time *= 0.9

	# Finisher is heavy
	if combo_count >= 3:
		current_weapon.anticipation_time *= 1.5
		current_weapon.camera_shake_intensity *= 2.0

# Directional attacks (overhead vs side swing)
func get_attack_direction() -> String:
	var input_y = Input.get_axis("up", "down")

	if input_y < -0.5:
		return "overhead"  # Slower startup, more damage
	else:
		return "side"  # Faster, less damage

func start_attack():
	var direction = get_attack_direction()

	match direction:
		"overhead":
			current_weapon.anticipation_time *= 1.3
			current_weapon.base_damage *= 1.5
		"side":
			current_weapon.anticipation_time *= 0.8

	super.start_attack()

# Weapon degradation (weight increases as weapon breaks)
var durability: float = 100.0

func _execute_attack():
	super._execute_attack()
	durability -= 1.0

	if durability < 30:
		# Broken weapon is heavier, slower
		current_weapon.movement_penalty *= 0.8
		current_weapon.action_time *= 1.2

# Two-handed vs one-handed
var is_two_handed: bool = false

func toggle_grip():
	is_two_handed = not is_two_handed

	if is_two_handed:
		# Two-handed: slower, more damage
		current_weapon.movement_penalty *= 0.8
		current_weapon.base_damage *= 1.5
		current_weapon.anticipation_time *= 1.2
	else:
		# One-handed: faster, less damage, can use shield
		# (Restore to base values)
		pass
```

**Animation Curve Examples:**
```gdscript
# Create curves for different weapon feels
func create_light_weapon_curve() -> Curve:
	var curve = Curve.new()
	# Fast start, linear progression
	curve.add_point(Vector2(0.0, 0.0))
	curve.add_point(Vector2(0.3, 0.8))
	curve.add_point(Vector2(1.0, 1.0))
	return curve

func create_heavy_weapon_curve() -> Curve:
	var curve = Curve.new()
	# Slow start, explosive finish
	curve.add_point(Vector2(0.0, 0.0))
	curve.add_point(Vector2(0.6, 0.2))  # Long windup
	curve.add_point(Vector2(0.8, 0.9))  # Explosive
	curve.add_point(Vector2(1.0, 1.0))
	return curve

# Apply curve to attack animation
func _update_attack(delta):
	super._update_attack(delta)

	var progress = 1.0 - (phase_timer / current_weapon.action_time)
	var curve_value = current_weapon.swing_curve.sample(progress)

	# Use curve_value to scale weapon rotation speed
	weapon_sprite.rotation = lerp(0.0, PI, curve_value)
```
