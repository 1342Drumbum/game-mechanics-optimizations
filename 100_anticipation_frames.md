### 100. Anticipation Frames (Windup)

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Actions feel instant and robotic, lacking weight and telegraphing. Players can't read enemy attacks or prepare for their own actions. Immediate execution makes powerful actions feel weak. Without anticipation, animations lack natural rhythm and player reactions feel unearned. Enemy attacks hit without warning, creating frustration instead of challenge.

**Technical Explanation:**
One of Disney's 12 Principles of Animation. Insert preparatory frames before main action - character winds back before swinging forward, squashes before jumping, leans before running. Typically 3-12 frames (0.05-0.2s at 60fps) of opposite motion before the primary action. Creates visual rhythm: anticipation → action → follow-through. Implementation stores animation states and enforces timing delays between input and execution. The "coyote time" of attack animations - player commits to action before seeing full result.

**Algorithmic Complexity:**
- O(1) state management: Simple timer + state enum
- O(1) animation lookup: Hash table or array indexing
- CPU: <0.01ms per character
- Memory: ~100 bytes per character (timers, state)

**Implementation Pattern:**
```gdscript
# Godot implementation - Anticipation Animation System
class_name AnticipationController
extends Node

enum ActionState { IDLE, ANTICIPATION, ACTION, RECOVERY }

@export var character: CharacterBody2D
@export var animation_player: AnimationPlayer
@export var sprite: Sprite2D

var current_state: ActionState = ActionState.IDLE
var action_queue: Array[String] = []

# Timing parameters (in seconds)
var anticipation_times: Dictionary = {
	"light_attack": 0.083,  # 5 frames at 60fps
	"heavy_attack": 0.200,  # 12 frames
	"jump": 0.066,          # 4 frames
	"dash": 0.050,          # 3 frames
	"special": 0.250,       # 15 frames
}

var action_times: Dictionary = {
	"light_attack": 0.150,  # 9 frames
	"heavy_attack": 0.350,  # 21 frames
	"jump": 0.100,          # 6 frames
	"dash": 0.200,          # 12 frames
	"special": 0.500,       # 30 frames
}

var recovery_times: Dictionary = {
	"light_attack": 0.100,  # 6 frames
	"heavy_attack": 0.200,  # 12 frames
	"jump": 0.050,          # 3 frames
	"dash": 0.133,          # 8 frames
	"special": 0.300,       # 18 frames
}

var current_action: String = ""
var state_timer: float = 0.0

func _process(delta):
	if current_state == ActionState.IDLE:
		_check_for_input()
	else:
		_update_action(delta)

func _check_for_input():
	# Example input handling
	if Input.is_action_just_pressed("light_attack"):
		start_action("light_attack")
	elif Input.is_action_just_pressed("heavy_attack"):
		start_action("heavy_attack")
	elif Input.is_action_just_pressed("jump"):
		start_action("jump")

func start_action(action_name: String):
	if current_state != ActionState.IDLE:
		# Optional: Queue action or ignore
		return

	current_action = action_name
	current_state = ActionState.ANTICIPATION
	state_timer = anticipation_times[action_name]

	# Play anticipation animation
	animation_player.play(action_name + "_anticipation")

	# Visual feedback during anticipation
	_apply_anticipation_visuals(action_name)

	# Emit signal for other systems
	action_started.emit(action_name)

func _update_action(delta):
	state_timer -= delta

	if state_timer <= 0.0:
		match current_state:
			ActionState.ANTICIPATION:
				_transition_to_action()
			ActionState.ACTION:
				_transition_to_recovery()
			ActionState.RECOVERY:
				_transition_to_idle()

func _transition_to_action():
	current_state = ActionState.ACTION
	state_timer = action_times[current_action]

	# Play main action animation
	animation_player.play(current_action + "_action")

	# Execute gameplay effect (damage, movement, etc.)
	_execute_action_effect(current_action)

	# Visual/audio feedback
	action_executed.emit(current_action)

func _transition_to_recovery():
	current_state = ActionState.RECOVERY
	state_timer = recovery_times[current_action]

	# Play recovery animation
	animation_player.play(current_action + "_recovery")

func _transition_to_idle():
	current_state = ActionState.IDLE
	current_action = ""
	animation_player.play("idle")

	action_completed.emit()

func _apply_anticipation_visuals(action_name: String):
	match action_name:
		"heavy_attack":
			# Pull weapon back
			sprite.offset.x = -10
			# Squash slightly
			sprite.scale = Vector2(0.9, 1.1)
		"jump":
			# Crouch down
			sprite.scale = Vector2(1.2, 0.8)
			sprite.offset.y = 5
		"dash":
			# Lean in dash direction
			sprite.skew = 0.1

func _execute_action_effect(action_name: String):
	match action_name:
		"light_attack", "heavy_attack":
			_spawn_hitbox(action_name)
		"jump":
			character.velocity.y = -500
		"dash":
			var dash_dir = Vector2(Input.get_axis("left", "right"), 0)
			character.velocity = dash_dir * 800

# Enemy anticipation (telegraph attacks)
class_name EnemyAnticipation extends Node

@export var telegraph_duration: float = 0.5  # Longer for player reaction
@export var telegraph_sprite: Sprite2D
@export var telegraph_color: Color = Color.RED

var is_telegraphing: bool = false

func telegraph_attack(attack_name: String):
	is_telegraphing = true

	# Visual telegraph
	telegraph_sprite.visible = true
	telegraph_sprite.modulate = telegraph_color

	# Flash effect
	var tween = create_tween()
	tween.tween_property(telegraph_sprite, "modulate:a", 0.0, 0.2)
	tween.tween_property(telegraph_sprite, "modulate:a", 1.0, 0.2)
	tween.set_loops(int(telegraph_duration / 0.4))

	# Wait for anticipation to complete
	await get_tree().create_timer(telegraph_duration).timeout

	is_telegraphing = false
	telegraph_sprite.visible = false

	# Execute attack
	execute_attack(attack_name)
```

**Key Parameters:**
- **Light attack anticipation:** 3-5 frames (0.05-0.083s)
- **Heavy attack anticipation:** 8-15 frames (0.133-0.25s)
- **Jump anticipation:** 2-5 frames (0.033-0.083s)
- **Ultimate/special anticipation:** 15-30 frames (0.25-0.5s)
- **Enemy telegraph:** 0.3-1.0s (depends on difficulty)
- **Boss attack telegraph:** 0.5-2.0s (more reaction time)
- **Anticipation movement distance:** 5-20 pixels pullback

**Edge Cases:**
- **Canceling anticipation:** Allow cancel during anticipation, not action
- **Interrupted anticipation:** Enemy hit during windup cancels action
- **Animation blending:** Smooth transition between anticipation and action
- **Rapid inputs:** Buffer next action during recovery, not anticipation
- **Network lag:** Client predicts anticipation, server validates
- **Stun during anticipation:** Cancel immediately, play stun animation
- **Death during windup:** Abort action, play death immediately

**When NOT to Use:**
- **Ultra-responsive games:** Fighting games where frame-perfect input matters
- **Twitch shooters:** Gun firing must be instant
- **Rhythm games:** Destroys timing precision
- **Quick actions:** Picking up items, opening doors
- **Menu interactions:** Frustrating UI delay
- **Already slow actions:** Don't add 0.2s anticipation to 2s cast time

**Examples from Shipped Games:**

1. **Hollow Knight:** Nail swing has 3-frame anticipation (arm pulls back), 5-frame action, 4-frame recovery. Shade Soul spell has 8-frame anticipation (charge-up glow). Anticipation is subtle but creates rhythm to combat.

2. **Cuphead:** Boss attacks have 0.5-2.0s telegraph animations (pink = parryable). Player parry has 2-frame anticipation. Heavy on telegraphing for pattern-learning gameplay. Exaggerated anticipation matches 1930s animation style.

3. **Monster Hunter:** Greatsword charge has 1-2 second anticipation (weapon raises), followed by powerful swing. Commitment to slow weapons is core mechanic. Anticipation creates risk/reward - can be interrupted.

4. **Dark Souls:** Heavy attacks have 10-20 frame anticipation, light attacks 5-8 frames. Enemy bosses telegraph attacks 0.5-1.5s before execution. Anticipation timing is learnable and consistent (skill-based).

5. **Celeste:** Madeline squats 3 frames before jump, leans 2 frames before dash. Minimal anticipation preserves responsiveness but adds personality. Dash anticipation is barely noticeable but improves feel.

**Platform Considerations:**
- **60fps target:** Design in frame counts (3, 5, 8, 12 frames)
- **30fps games:** Reduce frame counts by ~40% (not 50%, shorter feels better)
- **Variable refresh:** Use time-based (seconds), not frame counts
- **Input lag:** Account for display lag + anticipation (total <200ms)
- **Mobile touch:** Reduce anticipation 20-30% (touch feels less precise)
- **Competitive games:** Keep anticipation minimal (<100ms total)

**Godot-Specific Notes:**
- Use `AnimationPlayer` with separate anticipation/action/recovery animations
- `AnimationTree` for blending between states smoothly
- Store timing in resource files (`.tres`) for easy tweaking
- Use `Tween` for procedural anticipation visuals
- Input buffering: Store input during anticipation, execute on transition
- State machine pattern recommended (enum-based or `StateMachine` node)
- Godot 4.x: `AnimationMixer` allows layered animations

**Synergies:**
- **Squash & Stretch (Technique 96):** Squash during anticipation
- **Follow-Through (Technique 101):** Anticipation → Action → Follow-through
- **Hit Pause (Technique 95):** Pause at end of anticipation for extra impact
- **Particle Effects (Technique 98):** Spawn charge particles during anticipation
- **Camera Shake (Technique 97):** Small shake at anticipation end
- **Sound Design:** Whoosh/charge sound during anticipation

**Measurement/Profiling:**
- **Playtest feedback:** "Do actions feel responsive?" vs "Do attacks have weight?"
- **Frame timing:** Verify anticipation duration matches specification
- **Input lag testing:** Total time from button press to action execution
- **Animation timing:** Record at 60fps, count frames in video editor
- **Cancellation testing:** Verify cancel windows work correctly
- **A/B testing:** Same attack with/without anticipation, measure preference
- **Difficulty balance:** Enemy telegraphs should give enough reaction time

**Advanced Techniques:**
```gdscript
# Variable anticipation based on charge time
class_name ChargeAttack extends Node

var charge_time: float = 0.0
var max_charge_time: float = 2.0
var min_anticipation: float = 0.05
var max_anticipation: float = 0.3

func _process(delta):
	if Input.is_action_pressed("charge_attack"):
		charge_time = min(charge_time + delta, max_charge_time)
		_update_charge_visuals()
	elif Input.is_action_just_released("charge_attack"):
		_release_attack()

func _release_attack():
	var charge_percent = charge_time / max_charge_time
	var anticipation_time = lerp(min_anticipation, max_anticipation, charge_percent)

	# More charge = longer anticipation = more damage
	start_action_with_anticipation("charged_attack", anticipation_time)

# Combo system with variable anticipation
class_name ComboSystem extends Node

var combo_count: int = 0
var max_combo: int = 3

func perform_combo_attack():
	combo_count += 1

	# Later combo hits have less anticipation (faster flow)
	var anticipation = 0.1 - (combo_count * 0.02)
	anticipation = max(anticipation, 0.03)  # Minimum 3 frames

	start_action_with_anticipation("combo_" + str(combo_count), anticipation)

	# Reset combo after delay
	await get_tree().create_timer(1.0).timeout
	combo_count = 0

# Context-sensitive anticipation
func get_anticipation_time(action: String) -> float:
	var base_time = anticipation_times[action]

	# Reduce anticipation in air (faster aerial combat)
	if not character.is_on_floor():
		base_time *= 0.7

	# Increase anticipation when low stamina (fatigue)
	if stamina < 20:
		base_time *= 1.3

	return base_time

# Interrupt system
signal action_interrupted(action_name: String)

func interrupt_current_action():
	if current_state == ActionState.ANTICIPATION:
		# Can interrupt during anticipation
		action_interrupted.emit(current_action)
		_transition_to_idle()
		return true
	elif current_state == ActionState.ACTION:
		# Cannot interrupt during action (committed)
		return false
	return true

# Visual telegraph intensity scaling
func _update_telegraph_intensity(time_remaining: float, total_time: float):
	var intensity = 1.0 - (time_remaining / total_time)

	# Flash faster as attack approaches
	var flash_speed = lerp(1.0, 5.0, intensity)

	# Change color as attack approaches (yellow → red)
	telegraph_sprite.modulate = Color.YELLOW.lerp(Color.RED, intensity)
```

**Animation State Machine Example:**
```gdscript
# Complete state machine with anticipation
class_name CombatStateMachine extends Node

var states: Dictionary = {
	"idle": IdleState,
	"anticipation": AnticipationState,
	"action": ActionState,
	"recovery": RecoveryState,
	"hit_stun": HitStunState,
}

var current_state: State
var previous_state: State

func change_state(new_state_name: String):
	if current_state:
		current_state.exit()
		previous_state = current_state

	current_state = states[new_state_name].new()
	current_state.enter(self)

# State classes
class AnticipationState extends State:
	var timer: float

	func enter(sm):
		timer = sm.get_anticipation_time()
		sm.animation_player.play("anticipation")

	func update(delta):
		timer -= delta
		if timer <= 0:
			sm.change_state("action")
		elif sm.is_interrupted():
			sm.change_state("hit_stun")
```
