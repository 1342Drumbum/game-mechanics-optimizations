### 94. Separate Air/Ground Acceleration

**Category:** Game Feel - Movement

**Problem It Solves:** Identical air and ground control feels unnatural and unrealistic. Players have too much or too little air control. Platforming lacks weight and commitment. Movement feels either too "floaty" or too restricted. Can't balance responsive ground movement with meaningful jump arcs.

**Technical Explanation:**
Different acceleration values and friction coefficients for grounded vs airborne states. Ground acceleration typically 2-5x higher than air (ground = 30-50 u/s², air = 10-20 u/s²). Implementation tracks `is_on_floor()` state, applies appropriate acceleration curve. Ground provides instant feedback (0.15-0.3s to max speed), air provides control without full authority (0.5-1.0s to redirect). Air friction usually 30-50% of ground friction. Creates natural physics feel: easy to change direction on ground, committed trajectory in air. Advanced implementation: separate forward/strafe acceleration in air (forward = 15 u/s², strafe = 20 u/s²). Enables air strafing while maintaining forward momentum. Critical for platformer feel - wrong ratio makes game feel broken.

**Algorithmic Complexity:**
- State check: O(1) - boolean floor detection
- Acceleration application: O(1) - vector addition
- Per-frame: O(1) - simple branching
- Memory: ~48 bytes (ground/air state, acceleration values)
- CPU: <0.005ms per character

**Implementation Pattern:**
```gdscript
class_name DualAccelerationController extends CharacterBody3D

# Ground movement parameters
@export var ground_max_speed: float = 10.0
@export var ground_acceleration: float = 40.0  # Fast ground response
@export var ground_deceleration: float = 50.0  # Even faster stopping
@export var ground_friction: float = 0.8

# Air movement parameters
@export var air_max_speed: float = 8.0  # Slightly slower than ground
@export var air_acceleration: float = 15.0  # ~38% of ground (reduced control)
@export var air_deceleration: float = 5.0  # Much slower air braking
@export var air_friction: float = 0.2  # Minimal air resistance

# Advanced: Directional air control
@export var air_forward_acceleration: float = 10.0
@export var air_strafe_acceleration: float = 20.0  # Easier to strafe than go forward

# State tracking
var was_on_floor: bool = true
var air_time: float = 0.0

func _physics_process(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")
    var is_grounded = is_on_floor()

    # Track state transitions
    if is_grounded and not was_on_floor:
        on_land()
    elif not is_grounded and was_on_floor:
        on_leave_ground()

    # Update air time
    if not is_grounded:
        air_time += delta
    else:
        air_time = 0.0

    # Apply appropriate movement
    if is_grounded:
        apply_ground_movement(input_dir, delta)
    else:
        apply_air_movement(input_dir, delta)

    # Apply friction
    apply_friction(is_grounded, delta)

    move_and_slide()
    was_on_floor = is_grounded

func apply_ground_movement(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    # Calculate movement direction
    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Determine if accelerating or decelerating
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    var accel = ground_acceleration
    var target_velocity = direction * ground_max_speed

    # Apply acceleration
    velocity.x = move_toward(velocity.x, target_velocity.x, accel * delta)
    velocity.z = move_toward(velocity.z, target_velocity.z, accel * delta)

func apply_air_movement(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Much slower air acceleration
    var accel = air_acceleration
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    # Don't exceed air speed cap
    var target_velocity = direction * air_max_speed

    # Softer acceleration in air
    velocity.x += (target_velocity.x - velocity.x) * (accel / ground_acceleration) * delta
    velocity.z += (target_velocity.z - velocity.z) * (accel / ground_acceleration) * delta

    # Clamp to air max speed
    horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    if horizontal_velocity.length() > air_max_speed:
        horizontal_velocity = horizontal_velocity.normalized() * air_max_speed
        velocity.x = horizontal_velocity.x
        velocity.z = horizontal_velocity.z

func apply_friction(is_grounded: bool, delta: float) -> void:
    var friction = ground_friction if is_grounded else air_friction
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    if current_speed < 0.1:
        return

    # Apply friction force
    var friction_force = current_speed * friction * delta
    var new_speed = max(0.0, current_speed - friction_force)

    # Preserve direction
    if current_speed > 0:
        velocity.x = (velocity.x / current_speed) * new_speed
        velocity.z = (velocity.z / current_speed) * new_speed

func on_land() -> void:
    # Optional: Landing effects, velocity adjustments
    # Can add landing recovery time here
    pass

func on_leave_ground() -> void:
    # Optional: Preserve some ground momentum
    # Or apply initial air velocity modifier
    pass

# Advanced: Directional air control (forward vs strafe)
func apply_directional_air_control(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var forward = -camera_basis.z
    var right = camera_basis.x

    # Project input onto forward and right vectors
    var forward_input = input_dir.y  # Negative = forward
    var strafe_input = input_dir.x   # Positive = right

    # Different acceleration for forward vs strafe
    var forward_accel = air_forward_acceleration * delta
    var strafe_accel = air_strafe_acceleration * delta

    # Apply directional forces
    velocity.x += forward.x * forward_input * forward_accel
    velocity.z += forward.z * forward_input * forward_accel

    velocity.x += right.x * strafe_input * strafe_accel
    velocity.z += right.z * strafe_input * strafe_accel

# Alternative: Momentum-based air control
@export var air_control_multiplier: float = 0.3  # 30% of ground control

func apply_momentum_air_control(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Air control scales with ground acceleration
    var air_accel = ground_acceleration * air_control_multiplier * delta

    velocity.x += direction.x * air_accel
    velocity.z += direction.z * air_accel

# Advanced: Gradual air control (more control over time)
func apply_gradual_air_control(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    # Air control increases the longer you're airborne
    var control_factor = min(air_time / 0.5, 1.0)  # Full control after 0.5s
    var effective_accel = air_acceleration * (0.3 + 0.7 * control_factor)  # 30-100%

    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    var accel_force = direction * effective_accel * delta
    velocity.x += accel_force.x
    velocity.z += accel_force.z

# Quake-style air acceleration (direction-based)
func apply_quake_air_acceleration(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var wish_dir = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)

    # Only accelerate if wish direction differs from current velocity
    var projection = horizontal_velocity.dot(wish_dir)
    var add_speed = air_max_speed - projection

    if add_speed <= 0:
        return

    var accel_speed = air_acceleration * delta
    accel_speed = min(accel_speed, add_speed)

    velocity.x += wish_dir.x * accel_speed
    velocity.z += wish_dir.z * accel_speed

# Helper: Calculate stopping distance
func calculate_ground_stopping_distance(speed: float) -> float:
    # Distance = speed² / (2 * deceleration)
    return (speed * speed) / (2.0 * ground_deceleration)

func calculate_air_stopping_distance(speed: float) -> float:
    # Much longer due to lower deceleration
    return (speed * speed) / (2.0 * air_deceleration)

# Debug visualization
func _draw_movement_debug() -> void:
    var is_grounded = is_on_floor()
    var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()
    var accel = ground_acceleration if is_grounded else air_acceleration
    var max_speed = ground_max_speed if is_grounded else air_max_speed

    print("State: %s | Speed: %.1f/%.1f | Accel: %.1f | Air Time: %.2fs" % [
        "GROUND" if is_grounded else "AIR",
        horizontal_speed,
        max_speed,
        accel,
        air_time
    ])

# Compare acceleration profiles
func get_acceleration_ratio() -> float:
    return air_acceleration / ground_acceleration

func get_time_to_max_speed_ground() -> float:
    return ground_max_speed / ground_acceleration

func get_time_to_max_speed_air() -> float:
    return air_max_speed / air_acceleration
```

**Key Parameters:**
- **ground_acceleration:** 30-60 u/s² (40 responsive, 50 snappy)
- **air_acceleration:** 10-25 u/s² (15 = 38% of ground, balanced)
- **acceleration_ratio:** 0.25-0.5 (air/ground, 0.38 typical)
- **ground_friction:** 0.6-1.0 (0.8 moderate)
- **air_friction:** 0.1-0.4 (0.2 minimal resistance)
- **air_max_speed:** 80-100% of ground speed (8 vs 10 = 80%)
- **ground_deceleration:** 1.2-1.5x acceleration (faster stopping)

**Edge Cases:**
- **Slope transitions:** Treat steep slopes as "air" (reduced traction)
- **Water surfaces:** Third acceleration profile (slower than both)
- **Moving platforms:** Add platform velocity to calculations
- **Wall slide state:** Separate acceleration profile (vertical control)
- **Damage knockback:** Override acceleration temporarily
- **Ice/slippery surfaces:** Reduce ground acceleration 50-70%
- **Coyote time:** Maintain ground acceleration 0.1s after leaving edge

**When NOT to Use:**
- **Purely realistic sims:** Racing, sports (use actual physics models)
- **Zero-gravity games:** Need 3D volumetric movement
- **2D platformers (sometimes):** Often use same values for simplicity
- **Top-down games:** No meaningful ground/air distinction
- **Flight simulators:** Completely different physics model
- **When instant response needed:** Fighting games, rhythm games

**Examples from Shipped Games:**

1. **Super Mario 64:** Ground accel 40 u/s², air 12 u/s² (30% ratio). Ground max 8.0, air max 8.0 (same cap). Air control allows gentle trajectory adjustments. Ground control tight and responsive. Triple jump preserves momentum into air. Defines 3D platformer feel.

2. **Celeste:** Ground accel 100 u/s², air 25 u/s² (25% ratio). Extremely responsive ground, moderate air control. Ground friction high (instant stop on input release). Air friction minimal (preserve momentum). Dash bypasses both systems. Tight control critical for precision platforming.

3. **Quake III Arena:** Ground accel 70 u/s², air 10 u/s² but with special strafe rules. Air acceleration only perpendicular to current velocity (enables strafe jumping). Ground friction 6.0, air friction 0.0. Entire movement meta based on air/ground difference. Pro players minimize ground time.

4. **Titanfall 2:** Ground accel 45 u/s², air 18 u/s² (40% ratio). Slide/wallrun override both systems. Air control allows mid-jump correction. Ground control prioritizes combat responsiveness. Momentum preservation between states. Chaining moves minimizes ground friction.

5. **Doom Eternal (2020):** Ground accel 50 u/s², air 20 u/s² (40% ratio). Meathook and double jump grant temporary increased air control. Ground instant stop for combat precision. Air control allows combat repositioning. Balanced for aggressive forward movement.

**Platform Considerations:**
- **All platforms:** Core mechanic, no platform-specific tuning
- **Controller:** Air acceleration feels better with analog input (gradual direction change)
- **Keyboard:** Ground acceleration more important (instant direction changes)
- **Mobile:** Slightly higher air control (0.45 ratio) for imprecise touch input
- **VR:** May want higher air control (0.5 ratio) for comfort
- **Competitive:** Fixed physics timestep essential for consistency

**Godot-Specific Notes:**
- `CharacterBody3D.is_on_floor()` for state detection
- `get_floor_normal()` to distinguish flat ground from slopes
- Use separate acceleration curves (`@export var Curve`) for each state
- AnimationTree: Blend animations based on grounded state
- `move_and_slide()` parameters: Set `floor_max_angle` for slope detection
- Godot 4.x: More reliable floor detection than 3.x
- Consider `PhysicsServer3D.body_get_direct_state()` for advanced control
- Debug: Draw velocity vector, color-coded by state (green = ground, blue = air)

**Synergies:**
- **Air Strafing (#83):** Relies on separate air acceleration for technique
- **Strafe Jump (#92):** Ground friction incentivizes staying airborne
- **Momentum Curves (#89):** Different drag curves for ground vs air
- **Jump Height Variability (#85):** Air control affects jump arc shape
- **Slide Momentum (#88):** Slide uses third acceleration profile
- **Wall Running (#87):** Wall run = fourth acceleration profile
- **Sprint Acceleration (#91):** Sprint modifies ground acceleration

**Measurement/Profiling:**
- **Time to max speed:** Ground 0.2-0.3s, Air 0.5-1.0s (should be 2-3x difference)
- **Stopping distance:** Ground 1-2m, Air 4-8m (air should be 3-4x longer)
- **Direction change speed:** 180° turn time Ground 0.15s, Air 0.4s
- **Player feedback:** "Does jumping feel committed?" (target >80% yes)
- **Air control comfort:** "Can you adjust jumps mid-air?" (target >75% yes)
- **Platforming success:** Jump precision challenges pass rate >70%
- **Acceleration ratio effectiveness:** A/B test 0.3 vs 0.4 vs 0.5 ratios
- **Performance:** State check + acceleration <0.005ms per character
- **Level design:** Do gaps require both ground setup and air correction? (good design)
