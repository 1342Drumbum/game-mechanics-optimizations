### 91. Sprint Delay/Acceleration

**Category:** Game Feel - Movement

**Problem It Solves:** Instant sprint activation removes stamina/weight feeling and enables twitch exploitation. Sprint-strafing breaks tactical gameplay balance. Players can abuse instant speed changes for combat advantage. No commitment to movement direction reduces decision-making depth. Character feels weightless and ungrounded.

**Technical Explanation:**
Sprint has multi-phase activation: input detection → windup delay (0.1-0.3s) → acceleration phase (0.2-0.5s) → full sprint. Windup requires continuous input without direction change >45°. Acceleration applies ease-out curve from walk to sprint speed. Implementation: `sprint_timer += delta` while conditions met, `speed = lerp(walk_speed, sprint_speed, curve.sample(timer / accel_time))`. Direction change >threshold resets timer, creating commitment. Optional: FOV increase during acceleration (65° → 75° over 0.3s) enhances speed perception. Stamina integration: sprint drains resource, empty stamina prevents sprint start. Creates tactical decision: when to commit to sprint vs maintain agility. Common pattern: 0.2s windup + 0.3s acceleration = 0.5s total to max sprint speed.

**Algorithmic Complexity:**
- Per-frame: O(1) - timer update and curve sample
- Direction check: O(1) - dot product calculation
- No iterations or complex calculations
- Memory: ~24 bytes (timer, direction, state)
- CPU: <0.005ms per character

**Implementation Pattern:**
```gdscript
class_name SprintController extends CharacterBody3D

# Sprint parameters
@export var walk_speed: float = 5.0
@export var sprint_speed: float = 9.0
@export var sprint_windup_time: float = 0.2  # Delay before acceleration starts
@export var sprint_acceleration_time: float = 0.3  # Time to reach max sprint
@export var sprint_fov_increase: float = 10.0  # FOV change (degrees)

# Direction commitment
@export var max_direction_change: float = 45.0  # Degrees allowed without reset
@export var require_forward_input: bool = true  # No sprint while strafing

# Stamina integration (optional)
@export var enable_stamina: bool = true
@export var max_stamina: float = 100.0
@export var stamina_drain_rate: float = 20.0  # Per second while sprinting
@export var stamina_regen_rate: float = 15.0  # Per second while not sprinting
@export var min_stamina_to_sprint: float = 10.0

# State tracking
var is_sprinting: bool = false
var sprint_windup_timer: float = 0.0
var sprint_acceleration_timer: float = 0.0
var current_sprint_direction: Vector3 = Vector3.ZERO
var current_stamina: float = 100.0

# Animation and effects
@export var sprint_acceleration_curve: Curve
@onready var camera: Camera3D = $Camera3D
var base_fov: float = 75.0

func _ready() -> void:
    base_fov = camera.fov if camera else 75.0
    current_stamina = max_stamina

func _physics_process(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")

    # Update stamina
    update_stamina(delta)

    # Check sprint conditions
    if should_attempt_sprint(input_dir):
        update_sprint_state(input_dir, delta)
    else:
        cancel_sprint()

    # Apply movement
    var current_speed = calculate_current_speed()
    apply_movement(input_dir, current_speed, delta)

    move_and_slide()

func should_attempt_sprint(input_dir: Vector2) -> bool:
    # Must be holding sprint button
    if not Input.is_action_pressed("sprint"):
        return false

    # Must be on ground
    if not is_on_floor():
        return false

    # Must have input
    if input_dir.length() < 0.5:
        return false

    # Optional: Require forward input (no strafe sprinting)
    if require_forward_input and input_dir.y > -0.5:
        return false

    # Check stamina
    if enable_stamina and current_stamina < min_stamina_to_sprint:
        return false

    return true

func update_sprint_state(input_dir: Vector2, delta: float) -> void:
    var camera_basis = camera.global_transform.basis if camera else global_transform.basis
    var intended_direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Windup phase
    if sprint_windup_timer < sprint_windup_time:
        # Check if player changed direction significantly
        if current_sprint_direction != Vector3.ZERO:
            var direction_change = rad_to_deg(current_sprint_direction.angle_to(intended_direction))
            if direction_change > max_direction_change:
                # Direction changed too much, reset windup
                sprint_windup_timer = 0.0
                current_sprint_direction = intended_direction
                return

        sprint_windup_timer += delta
        current_sprint_direction = intended_direction
        return

    # Acceleration phase
    if sprint_acceleration_timer < sprint_acceleration_time:
        sprint_acceleration_timer += delta
        is_sprinting = true

        # Update FOV during acceleration
        update_sprint_fov()
        return

    # Full sprint reached
    is_sprinting = true
    current_sprint_direction = intended_direction

func cancel_sprint() -> void:
    if not is_sprinting and sprint_windup_timer == 0.0:
        return

    is_sprinting = false
    sprint_windup_timer = 0.0
    sprint_acceleration_timer = 0.0
    current_sprint_direction = Vector3.ZERO

    # Reset FOV
    reset_fov()

func calculate_current_speed() -> float:
    if not is_sprinting and sprint_windup_timer < sprint_windup_time:
        # During windup: stay at walk speed
        return walk_speed

    if sprint_acceleration_timer < sprint_acceleration_time:
        # During acceleration: lerp between walk and sprint
        var t = sprint_acceleration_timer / sprint_acceleration_time

        # Apply curve if available
        var curve_value = sprint_acceleration_curve.sample(t) if sprint_acceleration_curve else t

        return lerp(walk_speed, sprint_speed, curve_value)

    # Full sprint speed
    return sprint_speed

func apply_movement(input_dir: Vector2, speed: float, delta: float) -> void:
    if input_dir.length() < 0.1:
        # Apply deceleration
        velocity.x = move_toward(velocity.x, 0, speed * 2.0 * delta)
        velocity.z = move_toward(velocity.z, 0, speed * 2.0 * delta)
        return

    # Calculate movement direction
    var camera_basis = camera.global_transform.basis if camera else global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Apply acceleration
    var acceleration = 30.0
    velocity.x = move_toward(velocity.x, direction.x * speed, acceleration * delta)
    velocity.z = move_toward(velocity.z, direction.z * speed, acceleration * delta)

func update_stamina(delta: float) -> void:
    if not enable_stamina:
        return

    if is_sprinting and sprint_acceleration_timer >= sprint_acceleration_time:
        # Drain stamina while sprinting
        current_stamina -= stamina_drain_rate * delta
        current_stamina = max(0.0, current_stamina)

        # Stop sprinting if stamina depleted
        if current_stamina <= 0:
            cancel_sprint()
    else:
        # Regenerate stamina
        current_stamina += stamina_regen_rate * delta
        current_stamina = min(max_stamina, current_stamina)

func update_sprint_fov() -> void:
    if not camera:
        return

    var t = sprint_acceleration_timer / sprint_acceleration_time
    var target_fov = base_fov + sprint_fov_increase * t
    camera.fov = lerp(camera.fov, target_fov, 0.1)

func reset_fov() -> void:
    if not camera:
        return

    var tween = create_tween()
    tween.tween_property(camera, "fov", base_fov, 0.2).set_trans(Tween.TRANS_CUBIC).set_ease(Tween.EASE_OUT)

# Advanced: Sprint momentum preservation
var sprint_momentum_bonus: float = 0.0

func calculate_sprint_momentum(delta: float) -> void:
    # Build momentum the longer you sprint
    if is_sprinting and sprint_acceleration_timer >= sprint_acceleration_time:
        sprint_momentum_bonus += 0.5 * delta  # +0.5 m/s per second
        sprint_momentum_bonus = min(sprint_momentum_bonus, 3.0)  # Cap at +3 m/s
    else:
        # Decay momentum when not sprinting
        sprint_momentum_bonus -= 2.0 * delta
        sprint_momentum_bonus = max(0.0, sprint_momentum_bonus)

func get_effective_sprint_speed() -> float:
    return sprint_speed + sprint_momentum_bonus

# Alternative: Slope-based sprint speed
func calculate_slope_sprint_speed() -> float:
    var floor_normal = get_floor_normal()
    if floor_normal == Vector3.ZERO:
        return sprint_speed

    # Calculate slope angle
    var slope_angle = acos(floor_normal.dot(Vector3.UP))
    var is_downhill = floor_normal.dot(current_sprint_direction) < 0

    if is_downhill:
        # Faster downhill (up to +30%)
        var angle_factor = min(slope_angle / (PI * 0.25), 1.0)  # Normalize to 45°
        return sprint_speed * (1.0 + 0.3 * angle_factor)
    else:
        # Slower uphill (down to -40%)
        var angle_factor = min(slope_angle / (PI * 0.25), 1.0)
        return sprint_speed * (1.0 - 0.4 * angle_factor)

# UI feedback (optional)
func _draw_sprint_ui() -> void:
    # Debug: Show sprint state
    if is_sprinting:
        print("SPRINTING - Speed: %.1f | Stamina: %.0f%%" % [
            calculate_current_speed(),
            (current_stamina / max_stamina) * 100.0
        ])
    elif sprint_windup_timer > 0:
        print("SPRINT WINDUP: %.1f%%" % [(sprint_windup_timer / sprint_windup_time) * 100.0])
```

**Key Parameters:**
- **sprint_windup_time:** 0.1-0.3s (0.2s balanced commitment)
- **sprint_acceleration_time:** 0.2-0.5s (0.3s smooth buildup)
- **walk_speed:** 4-6 units/s (5 typical)
- **sprint_speed:** 8-12 units/s (9 = 1.8x walk speed)
- **max_direction_change:** 30-60° (45° allows curves)
- **sprint_fov_increase:** 5-15° (10° enhances speed feel)
- **stamina_drain_rate:** 15-25/s (sprint for 4-6s at max stamina)

**Edge Cases:**
- **Sprint during jump:** Maintain sprint state in air, continue on landing
- **Sprint into wall:** Don't cancel sprint, wait for direction input
- **Slope transitions:** Recalculate speed on slope change
- **Damage while sprinting:** Cancel sprint, apply short cooldown
- **Sprint toggle vs hold:** Support both input modes (preference option)
- **Controller deadzone:** 0.2-0.3 for analog sticks (prevent accidental cancel)
- **Stair climbing:** Maintain sprint speed on stairs

**When NOT to Use:**
- **Fast arcade games:** Sonic, racing - want instant max speed
- **Fighting games:** Frame-perfect inputs required
- **Stealth games (sometimes):** Depends on design (MGSV uses delay, Dishonored doesn't)
- **Top-down shooters:** Often want instant sprint for bullet dodging
- **Puzzle platformers:** Sprint timing shouldn't be execution barrier
- **Horror games:** Depends on design (delay adds tension vs frustration)

**Examples from Shipped Games:**

1. **The Last of Us Part II:** 0.3s windup + 0.4s acceleration = 0.7s total. Requires continuous forward input. Direction change >40° resets windup. Sprint animation telegraphs commitment. FOV increases 8° during sprint. Cannot sprint below 20% stamina. Creates deliberate, weighty feel.

2. **Red Dead Redemption 2:** 0.25s windup + 0.5s acceleration. Horse sprinting uses stamina (drains in 8-10s). Sprint speed increases with stamina core level. Direction change allowed during windup but slows acceleration. Realistic momentum buildup.

3. **Ghost of Tsushima:** 0.15s windup + 0.25s acceleration (fast for action game). No stamina system (exploration focus). Can chain sprint with dodge. FOV increase 12° (pronounced). Wind effects enhance speed perception. Allows sprint-strafe (90° direction change allowed).

4. **Hunt: Showdown:** 0.3s windup, very restrictive. Sprinting creates loud footsteps (tactical cost). Stamina depletes in 5s, slow regen (12s to full). Cannot sprint with weapon raised. Heavy commitment required - competitive balance.

5. **Fortnite:** Minimal delay (0.05s windup, 0.1s accel). Sprint freely while maintaining shooting ability (tactical sprint separate). Sliding from sprint preserves momentum. Fast-paced action prioritizes responsiveness over realism.

**Platform Considerations:**
- **All platforms:** Input-agnostic, works with any control scheme
- **Controller:** Analog stick allows partial sprint (70% forward = partial windup)
- **Keyboard:** Binary input benefits from clear windup feedback
- **Mobile:** Touch sprint button with hold duration indicator
- **Accessibility:** Option to reduce/remove windup time
- **Competitive:** Lock to fixed physics timestep (60Hz+)

**Godot-Specific Notes:**
- Use `Input.is_action_pressed()` for continuous sprint check
- `get_floor_normal()` for slope-based speed adjustments
- Export `Curve` for visual acceleration tuning
- Camera FOV: `Camera3D.fov` property, animate with `Tween`
- AnimationTree: Blend sprint animation based on `sprint_acceleration_timer`
- Godot 4.x: Better input handling, supports simultaneous actions
- UI: Use `ProgressBar` to show sprint stamina/windup progress

**Synergies:**
- **Slide Momentum (#88):** Sprint → slide preserves speed
- **Acceleration Curves (#84):** Sprint uses separate acceleration curve
- **Momentum Curves (#89):** Sprint adds to global momentum bonus
- **Wall Running (#87):** Can initiate wall run from sprint
- **Jump Variability (#85):** Sprint jump has different height/distance
- **Stamina System:** Sprint drains shared resource with dodge/melee
- **Combat States:** Cannot sprint while aiming/attacking

**Measurement/Profiling:**
- **Average windup time:** How long do players hold sprint before moving? (0.1-0.2s typical)
- **Sprint cancel rate:** % of sprints cancelled during windup (15-25% = good commitment)
- **Direction change frequency:** How often players change direction while sprinting (metric for exploitation)
- **Sprint duration:** Average continuous sprint time (3-6s typical)
- **Stamina depletion:** % of sprints that fully deplete stamina (should be <30%)
- **Player feedback:** "Does sprint feel responsive yet weighty?" (target >75% positive)
- **Competitive balance:** Do players abuse sprint in PvP? (monitor for exploits)
- **Performance:** Timer/curve sampling <0.005ms (negligible overhead)
