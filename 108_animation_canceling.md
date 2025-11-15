### 108. Animation Canceling

**Category:** Game Feel - Combat/Animation

**Problem It Solves:** Animation lock creates unresponsive combat where players are forced to watch full animations complete, leading to deaths from inability to react. Fighting game studies show 200-300ms animation locks reduce player satisfaction by 40-50%. Creates disconnect between input and action, breaking game feel and perceived responsiveness.

**Technical Explanation:**
Animation canceling allows interrupting current animation to start higher-priority action immediately. Implementation uses state machine with priority values, cancel windows (specific frames where cancel is allowed), and blend trees for smooth transitions. System checks if new action priority exceeds current, if within cancel window, then interrupts and blends to new animation over 0.05-0.15s.

Key technique: Store "animation commitment points" - frames before which cancel is allowed, after which animation must complete (e.g., attack windup cancelable, but not recovery after hit connects). Uses AnimationTree blend spaces and transitions with custom logic.

**Algorithmic Complexity:**
- Priority check: O(1) comparison
- State transition: O(1) state machine update
- Animation blend: O(n) where n = bones (typically 20-80)
- Total: O(n) per cancel, negligible overhead

**Implementation Pattern:**
```gdscript
# Godot animation canceling system
class_name AnimationCancelSystem
extends Node

enum ActionPriority {
	IDLE = 0,
	MOVE = 1,
	DODGE = 2,
	LIGHT_ATTACK = 3,
	HEAVY_ATTACK = 4,
	ABILITY = 5,
	HURT = 6,
	DEATH = 7
}

# Animation state with cancel rules
class AnimationState:
	var name: String
	var priority: ActionPriority
	var duration: float
	var cancel_window_start: float  # Frame where canceling becomes possible
	var cancel_window_end: float    # Frame where canceling ends (committed)
	var cancelable_by: Array[ActionPriority] = []  # Which actions can cancel this
	var blend_time: float = 0.1

	func can_cancel(new_priority: ActionPriority, current_time: float) -> bool:
		# Always allow higher priority
		if new_priority > priority:
			return true

		# Check if in cancel window
		if current_time < cancel_window_start or current_time > cancel_window_end:
			return false

		# Check if action is in allowed list
		return new_priority in cancelable_by

@export var animation_tree: AnimationTree
@export var animation_player: AnimationPlayer

var current_state: AnimationState
var animation_time: float = 0.0
var state_machine: AnimationNodeStateMachinePlayback

# Define animation states
var states: Dictionary = {}

func _ready():
	_setup_states()
	state_machine = animation_tree.get("parameters/playback")
	set_physics_process(true)

func _setup_states():
	# Light attack - can be canceled early, committed after frame 10
	var light_attack = AnimationState.new()
	light_attack.name = "light_attack"
	light_attack.priority = ActionPriority.LIGHT_ATTACK
	light_attack.duration = 0.6
	light_attack.cancel_window_start = 0.15  # Can cancel after windup
	light_attack.cancel_window_end = 0.35    # Committed during active frames
	light_attack.cancelable_by = [
		ActionPriority.DODGE,
		ActionPriority.HEAVY_ATTACK,
		ActionPriority.ABILITY,
		ActionPriority.HURT
	]
	light_attack.blend_time = 0.08
	states["light_attack"] = light_attack

	# Heavy attack - longer commitment
	var heavy_attack = AnimationState.new()
	heavy_attack.name = "heavy_attack"
	heavy_attack.priority = ActionPriority.HEAVY_ATTACK
	heavy_attack.duration = 1.2
	heavy_attack.cancel_window_start = 0.3
	heavy_attack.cancel_window_end = 0.8   # Long commitment
	heavy_attack.cancelable_by = [
		ActionPriority.DODGE,     # Can dodge cancel
		ActionPriority.ABILITY,   # Can ability cancel
		ActionPriority.HURT
	]
	heavy_attack.blend_time = 0.12
	states["heavy_attack"] = heavy_attack

	# Dodge - very cancelable
	var dodge = AnimationState.new()
	dodge.name = "dodge"
	dodge.priority = ActionPriority.DODGE
	dodge.duration = 0.5
	dodge.cancel_window_start = 0.2
	dodge.cancel_window_end = 0.5
	dodge.cancelable_by = [
		ActionPriority.LIGHT_ATTACK,
		ActionPriority.HEAVY_ATTACK,
		ActionPriority.ABILITY,
		ActionPriority.HURT
	]
	dodge.blend_time = 0.05  # Fast blend
	states["dodge"] = dodge

func _physics_process(delta: float):
	animation_time += delta

func try_action(action_name: String) -> bool:
	if action_name not in states:
		return false

	var new_state = states[action_name]

	# Check if can cancel current animation
	if current_state and not current_state.can_cancel(new_state.priority, animation_time):
		return false  # Cancel rejected

	# Perform cancel
	_transition_to(new_state)
	return true

func _transition_to(new_state: AnimationState):
	var blend_time = new_state.blend_time

	# Faster blend if canceling (more responsive feel)
	if current_state and animation_time < current_state.duration:
		blend_time *= 0.6  # 60% of normal blend time

	# Update state machine
	state_machine.travel(new_state.name)

	# Set blend time
	animation_tree.set("parameters/transition_blend_time", blend_time)

	current_state = new_state
	animation_time = 0.0

	# Emit signal for game logic (damage windows, etc.)
	animation_canceled.emit(new_state.name)

signal animation_canceled(new_action: String)

# Advanced: Frame-perfect cancel checking
func is_in_cancel_window() -> bool:
	if not current_state:
		return true
	return (animation_time >= current_state.cancel_window_start and
	        animation_time <= current_state.cancel_window_end)

# Advanced: Cancel with momentum preservation
func cancel_with_momentum(new_action: String, preserve_velocity: float = 0.7):
	if try_action(new_action):
		# Preserve movement momentum through cancel
		var velocity = get_parent().velocity
		get_parent().velocity = velocity * preserve_velocity

# Visual feedback for cancel windows (development tool)
func debug_draw_cancel_window():
	if not current_state:
		return

	var window_start = current_state.cancel_window_start
	var window_end = current_state.cancel_window_end
	var total_duration = current_state.duration

	# Draw timeline with cancel window highlighted
	# (Use CanvasItem or debug overlay)
	var in_window = is_in_cancel_window()
	print("Cancel window: ", "OPEN" if in_window else "CLOSED")

# Example usage in character controller
class_name CharacterCombat
extends CharacterBody3D

@export var cancel_system: AnimationCancelSystem

func _input(event: InputEvent):
	if event.is_action_pressed("light_attack"):
		cancel_system.try_action("light_attack")

	elif event.is_action_pressed("heavy_attack"):
		cancel_system.try_action("heavy_attack")

	elif event.is_action_pressed("dodge"):
		# Dodge has high priority, can cancel most things
		if cancel_system.try_action("dodge"):
			apply_dodge_velocity()

func apply_dodge_velocity():
	var direction = get_input_direction()
	velocity = direction * 12.0  # Fast dodge

# Advanced: Attack chains with cancel windows
class AttackChain:
	var attacks: Array[String] = ["light_1", "light_2", "light_3"]
	var current_index: int = 0
	var chain_window: float = 0.3  # Time window to continue chain
	var last_attack_time: float = 0.0

	func get_next_attack() -> String:
		var time_now = Time.get_ticks_msec() / 1000.0

		# Check if chain timed out
		if time_now - last_attack_time > chain_window:
			current_index = 0  # Reset chain

		var attack = attacks[current_index]
		current_index = (current_index + 1) % attacks.size()
		last_attack_time = time_now

		return attack
```

**Key Parameters:**
- **Cancel window duration:** 0.1-0.3s typical (3-9 frames at 30fps)
- **Blend time:** 0.05-0.15s (faster = snappier, slower = smoother)
- **Priority levels:** 5-8 distinct priorities sufficient
- **Commitment frames:** 30-50% of animation (windup), 10-20% recovery
- **Buffer window:** 0.05-0.1s to queue next action
- **Chain timing window:** 0.2-0.5s for combo continuation

**Edge Cases:**
- **Cancel spam:** Rate limit cancels (max 10-15 per second)
- **Animation events:** Ensure damage/VFX triggers even if canceled
- **Network sync:** Cancels must be frame-perfect synchronized
- **Interrupting blend:** Cancel while already blending between states
- **Death during animation:** Death priority should always override
- **Stagger/stun:** May need to disable canceling entirely
- **Invincibility frames:** Cancel window shouldn't grant extra i-frames

**When NOT to Use:**
- **Narrative sequences:** Cinematic attacks need to play fully
- **Turn-based combat:** No real-time canceling needed
- **Heavy "weighty" combat:** Dark Souls style wants commitment
- **Simple games:** Added complexity may not justify development time
- **Multiplayer with high latency:** >100ms ping makes frame-perfect cancels feel bad
- **Mobile/casual:** Players may not understand/utilize canceling

**Examples from Shipped Games:**

1. **Devil May Cry 5:** Master class in canceling - 90% of animations cancelable into dodge/jump. "Jump canceling" allows extending air combos infinitely. Attacks can be canceled on frame 1 of active frames (after windup). Creates SSS-rank depth - skilled players never animation locked. Cancel windows are 3-5 frames (0.1s at 30fps) requiring precise timing.

2. **Bayonetta:** "Dodge offset" - hold attack button during dodge to preserve combo state. Can cancel any attack into dodge with invincibility frames. Window is 6 frames (0.2s). Canceling is core mechanic that separates good from great players. Maintains perfect combo timing across dodge animation.

3. **God of War (2018):** Selective canceling - light attacks cancel into dodge after frame 10, heavy attacks committed after frame 20. Block can cancel anything at cost of stamina. More deliberate than DMC - punishes button mashing. 0.15s blend times maintain weighty feel while allowing reactions.

4. **Dark Souls 3:** Limited canceling - attacks mostly locked, but can cancel into roll at specific points. Roll canceling costs stamina. Design philosophy: commitment equals risk equals reward. Only recovery frames (last 30% of animation) are cancelable. Forces strategic timing over twitch reactions.

5. **Guilty Gear Strive:** Frame-perfect "Roman Cancels" - spend meter to cancel any action on any frame with custom slow-mo effect. Creates explosive pace changes. Multiple cancel types (Red/Purple/Blue) with different costs and effects. Competitive standard - top players cancel 30-50 times per round. Blend time is 2-3 frames (0.05s) for instant transitions.

**Platform Considerations:**
- **60fps vs 30fps:** 30fps needs longer cancel windows (8-10 frames vs 4-5)
- **Input latency:** Console/TV lag (50-100ms) requires generous buffers
- **Mobile touch:** Visual indicators needed (highlight cancel windows)
- **Network play:** Client prediction with rollback critical for feel
- **Competitive esports:** Frame data must be documented and consistent
- **Accessibility:** Option to disable canceling for simpler play

**Godot-Specific Notes:**
- `AnimationTree` with `AnimationNodeStateMachinePlayback` is ideal
- Use `travel()` for instant transitions vs `start()` for blend
- `Animation` blend modes: Set to `BLEND_MODE_BLEND` for smooth cancels
- Store cancel windows in custom `Resource` classes for data-driven design
- `AnimationPlayer.current_animation_position` for precise timing checks
- Use `queue()` in animation tracks to chain actions with buffering
- Network sync: Send cancel action + timestamp, replay on clients
- Performance: State machine has negligible overhead (<0.01ms)

**Synergies:**
- **Input Buffering (Technique 88):** Buffer inputs during non-cancelable frames
- **Hit Pause/Stop (Technique 75):** Pause animation during hit freeze
- **Animation Blending:** Smooth transitions between canceled animations
- **Combo System:** Canceling enables attack chains and juggles
- **I-Frame Dodging:** Cancel into dodge with invincibility frames
- **Stamina System:** Cancel costs can drain stamina for balance

**Measurement/Profiling:**
- **Cancel success rate:** Track attempted vs successful cancels
- **Average cancels per combat:** Skilled players should be 3-5x higher
- **Frame-perfect metric:** % of cancels within ideal 3-frame window
- **Responsiveness feel:** Input-to-action time should be <100ms
- **Animation blend quality:** No visible pops or snaps (visual QA)
- **Network desync rate:** Cancels must sync within 2 frames (<66ms)
- **Performance cost:** State machine overhead should be <0.05ms per frame

**Advanced Patterns:**
```gdscript
# Cancel buffering - queue next action during non-cancelable frames
var buffered_action: String = ""
var buffer_time: float = 0.0
const BUFFER_WINDOW = 0.1

func try_action_buffered(action_name: String) -> bool:
	if try_action(action_name):
		return true
	else:
		# Buffer the action
		buffered_action = action_name
		buffer_time = BUFFER_WINDOW
		return false

func _process(delta: float):
	if buffer_time > 0:
		buffer_time -= delta
		if try_action(buffered_action):
			buffer_time = 0  # Executed buffered action

	if buffer_time <= 0:
		buffered_action = ""

# Cancel with meter cost (fighting game style)
var meter: float = 100.0

func roman_cancel() -> bool:
	if meter < 50.0:
		return false

	meter -= 50.0
	# Cancel current animation immediately
	animation_player.seek(current_state.duration, true)  # Jump to end
	apply_hitstop(0.15)  # Slow-mo effect
	return true

# Chain canceling - cancel into specific followups only
func get_valid_cancels(current: String) -> Array[String]:
	var chains = {
		"light_1": ["light_2", "dodge"],
		"light_2": ["light_3", "heavy_attack", "dodge"],
		"light_3": ["launcher", "dodge"],
	}
	return chains.get(current, [])
```

---
