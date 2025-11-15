### 85. Jump Height Variability (Hold Duration)

**Category:** Game Feel - Movement

**Problem It Solves:** Fixed jump height removes player control and creates frustrating platforming. Players can't make subtle height adjustments for different gaps. Binary jump height (same every time) makes movement feel unresponsive and limits level design options. Skilled players want fine control over jump arcs for optimization.

**Technical Explanation:**
Jump height varies based on button hold duration. Initial jump applies full upward velocity; releasing button early applies "gravity multiplier" or reduces upward velocity. Two common implementations: (1) Reduce gravity while button held, increase when released (Mario-style), (2) Apply constant upward force while held, cap at maximum duration (Celeste-style). Physics: `velocity.y = initial_jump_force`, then while held: `velocity.y += hold_force * delta` or `gravity_scale = hold_gravity_multiplier`. Minimum jump height typically 30-40% of maximum. Critical timing window: 0.1-0.5 seconds between min and max height. Creates skill expression - expert players optimize hold duration for each jump.

**Algorithmic Complexity:**
- Per-frame check: O(1) - boolean check + float multiplication
- No complex calculations or iterations
- Memory: 4 bytes (jump timer) + 1 byte (button state)
- CPU: <0.001ms per character (trivial)

**Implementation Pattern:**
```gdscript
class_name VariableJumpController extends CharacterBody3D

# Jump parameters
@export var jump_initial_velocity: float = 15.0  # Initial impulse
@export var jump_hold_force: float = 35.0  # Additional force while held
@export var max_jump_hold_time: float = 0.3  # Maximum hold duration
@export var min_jump_hold_time: float = 0.08  # Minimum for full control

# Gravity modifiers
@export var gravity: float = 30.0
@export var gravity_multiplier_rising: float = 1.0  # While button held
@export var gravity_multiplier_falling: float = 2.0  # After release or peak
@export var max_fall_speed: float = 40.0

# State tracking
var is_jumping: bool = false
var jump_hold_timer: float = 0.0
var jump_released: bool = false

func _physics_process(delta: float) -> void:
    # Ground detection
    var was_on_floor = is_on_floor()

    # Jump input
    if Input.is_action_just_pressed("jump") and is_on_floor():
        start_jump()

    # Variable jump height
    if is_jumping:
        update_jump(delta)

    # Apply gravity
    apply_gravity(delta)

    move_and_slide()

    # Reset jump state on landing
    if not was_on_floor and is_on_floor():
        is_jumping = false
        jump_released = false
        jump_hold_timer = 0.0

func start_jump() -> void:
    # Apply initial jump impulse
    velocity.y = jump_initial_velocity
    is_jumping = true
    jump_released = false
    jump_hold_timer = 0.0

    # Optional: Play jump sound/animation
    # $AudioStreamPlayer.play()

func update_jump(delta: float) -> void:
    jump_hold_timer += delta

    # Check if player released jump button early
    if Input.is_action_just_released("jump") and not jump_released:
        jump_released = true
        # Optional: Immediately reduce upward velocity for snappy feel
        if velocity.y > 0:
            velocity.y *= 0.5  # Cut velocity in half

    # Apply hold force while button held and below max time
    if not jump_released and jump_hold_timer < max_jump_hold_time:
        if Input.is_action_pressed("jump"):
            # Add upward force while holding
            velocity.y += jump_hold_force * delta
        else:
            jump_released = true

    # Stop applying force after max hold time
    if jump_hold_timer >= max_jump_hold_time:
        is_jumping = false

func apply_gravity(delta: float) -> void:
    var gravity_to_apply = gravity

    # Modify gravity based on jump state
    if velocity.y > 0:  # Rising
        if is_jumping and not jump_released:
            # Reduced gravity while holding jump
            gravity_to_apply *= gravity_multiplier_rising
        else:
            # Normal gravity after release
            gravity_to_apply *= gravity_multiplier_falling
    else:  # Falling
        # Increased gravity while falling for snappy feel
        gravity_to_apply *= gravity_multiplier_falling

    velocity.y -= gravity_to_apply * delta

    # Clamp fall speed
    velocity.y = max(velocity.y, -max_fall_speed)

# Alternative implementation: Discrete jump heights
class_name DiscreteJumpController:
    @export var short_jump_velocity: float = 10.0
    @export var medium_jump_velocity: float = 15.0
    @export var full_jump_velocity: float = 20.0
    @export var short_jump_threshold: float = 0.1  # Release before 0.1s
    @export var medium_jump_threshold: float = 0.25  # Release before 0.25s

    var jump_start_time: float = 0.0
    var jump_committed: bool = false

    func start_jump():
        velocity.y = full_jump_velocity  # Start with full velocity
        jump_start_time = Time.get_ticks_msec() / 1000.0
        jump_committed = false

    func update_jump():
        if jump_committed:
            return

        var hold_duration = (Time.get_ticks_msec() / 1000.0) - jump_start_time

        if Input.is_action_just_released("jump"):
            jump_committed = true
            if hold_duration < short_jump_threshold:
                velocity.y = short_jump_velocity
            elif hold_duration < medium_jump_threshold:
                velocity.y = medium_jump_velocity
            # else: keep full jump velocity

# Advanced: Curve-based jump control
@export var jump_force_curve: Curve  # Curve from 0-1 over hold time

func update_jump_with_curve(delta: float) -> void:
    jump_hold_timer += delta

    if jump_hold_timer > max_jump_hold_time or jump_released:
        is_jumping = false
        return

    # Sample curve for force multiplier
    var t = jump_hold_timer / max_jump_hold_time
    var force_multiplier = jump_force_curve.sample(t) if jump_force_curve else 1.0

    velocity.y += jump_hold_force * force_multiplier * delta

# Helper: Calculate jump height
func calculate_max_jump_height() -> float:
    # Physics calculation for design reference
    var initial_height = (jump_initial_velocity * jump_initial_velocity) / (2.0 * gravity)
    var hold_distance = 0.5 * jump_hold_force * max_jump_hold_time * max_jump_hold_time
    return initial_height + hold_distance

func calculate_min_jump_height() -> float:
    # Assuming immediate release with velocity cut
    var reduced_velocity = jump_initial_velocity * 0.5
    return (reduced_velocity * reduced_velocity) / (2.0 * gravity * gravity_multiplier_falling)
```

**Key Parameters:**
- **jump_initial_velocity:** 12-20 units/s (15 typical for platformers)
- **max_jump_hold_time:** 0.2-0.5s (0.3s balanced, 0.4s floaty)
- **min_jump_hold_time:** 0.05-0.15s (minimum for player to register)
- **gravity_multiplier_falling:** 1.5-3.0x (2.0x = snappy, 1.5x = floaty)
- **velocity_cut_on_release:** 0.4-0.6 (0.5 = cut in half)
- **min/max height ratio:** 0.3-0.5 (min should be 30-50% of max)

**Edge Cases:**
- **Ceiling collision:** Immediately set velocity.y = 0, end jump
- **Button held from previous jump:** Track "just_pressed" separately from "is_pressed"
- **Variable framerate:** Delta-time issues can affect height - use fixed physics step
- **Landing during hold:** Reset jump state, don't carry over to next jump
- **Buffered jumps:** Allow jump input 0.1s before landing for responsive feel
- **Double jump:** Reset jump_released flag on second jump activation
- **Dash/ability cancels:** Clear jump state when other abilities interrupt

**When NOT to Use:**
- **Realistic sports games:** Basketball, volleyball need consistent jump physics
- **Auto-platformers:** Runner games with automated jumping
- **Puzzle games with precise physics:** Where jump height must be predictable
- **Limited input platforms:** Single-button mobile games
- **Swimming/flying:** Continuous vertical movement needs different control
- **Cutscenes/scripted sequences:** Use animation-driven jumps

**Examples from Shipped Games:**

1. **Super Mario 64/Odyssey:** 0.4s max hold time, 2.5x gravity when falling. Min jump ~35% of max height. Holds button = floaty 3-4m jump, tap = 1.5m hop. Critical for precise platforming. Variable jump taught in first 30 seconds of gameplay.

2. **Celeste:** 0.2s hold time, 3x falling gravity multiplier. Tight, responsive jump control. Min height 40% of max. Combined with dash for precise movement. Jump buffer window 0.1s (coyote time). Speedrunners optimize hold duration per-jump.

3. **Hollow Knight:** 0.25s hold time, 2x falling gravity. Jump height heavily affects platforming difficulty. Min jump used for navigating spike corridors. Max jump clears larger gaps. Single Wings upgrade increases max height 30%.

4. **Ori and the Will of the Wisps:** 0.35s hold, 1.8x falling multiplier (floatier feel). Bash ability extends mid-air control. Triple jump has separate hold parameters. Generous hold time supports exploratory gameplay.

5. **Dead Cells:** 0.15s hold (quick, snappy). 2.5x falling gravity. Combat-focused movement needs fast jumps. Double jump resets hold timer. Wings override jump mechanics entirely. Tight control for dodging enemy attacks.

**Platform Considerations:**
- **Touch controls:** Visual hold indicator (circle filling up), haptic feedback at max hold
- **Controller:** Analog trigger depth could modify jump (rarely used - too complex)
- **Keyboard:** Works perfectly - binary input benefits from variable height
- **Accessibility:** Option for "auto-hold" (single tap = max jump, hold = variable)
- **Input latency:** <50ms critical - delay breaks jump timing feel
- **Mobile:** Larger hit boxes for jump button (thumb precision issues)

**Godot-Specific Notes:**
- Use `Input.is_action_just_pressed()` for initial jump trigger
- `Input.is_action_pressed()` for hold detection
- Don't use `move_and_slide()` return value for floor detection (framerate dependent)
- Enable "Floor Snap" on CharacterBody3D for reliable ground detection
- Godot 4.x: `is_on_floor()` improved, more reliable than 3.x
- Export curves for jump force profile tuning
- Use `PhysicsServer3D.body_set_max_contacts_reported()` if multiple ground types

**Synergies:**
- **Air Strafing (#83):** Different air control during jump hold vs released
- **Coyote Time:** 0.1s grace period extends effective jump window
- **Jump Buffering:** Accept jump input 0.1s before landing
- **Landing Recovery (#86):** Harder landing from max height vs short hop
- **Crouch Jump Bonus (#93):** Hold crouch during jump for extra height
- **Wall Jump:** Reset jump_released flag when wall jumping
- **Double Jump:** Second jump uses same variable height mechanics

**Measurement/Profiling:**
- **Height distribution:** Track % of jumps at min/mid/max height (should see variety)
- **Average hold time:** 0.15-0.25s typical for skilled players
- **Success rates:** Do players fail jumps due to height misjudgment? (<5% acceptable)
- **Tutorial completion:** % players who understand variable height (target >90%)
- **Speedrun optimization:** Do experts use min-jumps for speed? (good sign)
- **Playtesting:** "Can you control jump height easily?" (target >85% yes)
- **Input timing precision:** Standard deviation of hold times (lower = more precise)
- **Platform difficulty:** Are gaps too punishing for variable height? Adjust level design
