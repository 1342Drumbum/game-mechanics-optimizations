### 84. Acceleration Curves (Ease In/Out)

**Category:** Game Feel - Movement

**Problem It Solves:** Instant acceleration/deceleration feels robotic and unnatural. Digital input (keyboard/button) creates jarring 0-to-max speed transitions, making character feel "floaty" or "ice skating." Players lose precision in platforming because they can't anticipate stopping distance. Movement lacks weight and responsiveness feedback.

**Technical Explanation:**
Instead of linear velocity changes, apply easing functions (cubic, sine, exponential) to create smooth acceleration/deceleration. Mathematical interpolation between current and target velocity over time. Common patterns: ease-out (fast start, slow end) for acceleration; ease-in (slow start, fast end) for deceleration. Implementation uses `lerp()` with non-linear time factors or explicit curve sampling. Physics formula: `velocity = lerp(current, target, ease_factor(t))` where `ease_factor()` returns 0-1 but follows a curve instead of linear. The curve shape determines "feel" - aggressive curves (cubic) feel snappy, gentle curves (sine) feel smooth. Critical for bridging digital input to analog feel.

**Algorithmic Complexity:**
- Per-frame: O(1) - single lerp + curve evaluation
- Curve lookup: O(1) if using pre-baked curve table
- Memory: ~32 bytes (current velocity + target + curve parameters)
- CPU: ~0.005ms per character (math operations only)

**Implementation Pattern:**
```gdscript
class_name AccelerationController extends CharacterBody3D

@export var max_speed: float = 10.0
@export var acceleration_time: float = 0.2  # Time to reach max speed
@export var deceleration_time: float = 0.15  # Time to stop
@export var acceleration_curve: Curve  # Visual curve editor in Godot
@export var deceleration_curve: Curve

# Advanced curve types
enum CurveType { LINEAR, EASE_IN, EASE_OUT, EASE_IN_OUT, CUBIC, EXPONENTIAL }
@export var accel_curve_type: CurveType = CurveType.EASE_OUT
@export var decel_curve_type: CurveType = CurveType.EASE_IN

var current_speed: float = 0.0
var target_speed: float = 0.0
var accel_timer: float = 0.0

func _physics_process(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")
    var input_length = input_dir.length()

    # Determine target speed from input
    target_speed = input_length * max_speed

    # Apply acceleration curve
    apply_curved_acceleration(delta)

    # Convert to velocity vector
    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()
    velocity.x = direction.x * current_speed
    velocity.z = direction.z * current_speed

    move_and_slide()

func apply_curved_acceleration(delta: float) -> void:
    var speed_diff = target_speed - current_speed

    if abs(speed_diff) < 0.01:
        current_speed = target_speed
        accel_timer = 0.0
        return

    # Determine if accelerating or decelerating
    var is_accelerating = speed_diff > 0
    var curve_time = acceleration_time if is_accelerating else deceleration_time

    accel_timer += delta
    var t = clamp(accel_timer / curve_time, 0.0, 1.0)

    # Sample appropriate curve
    var curve_value: float
    if is_accelerating:
        curve_value = sample_curve(t, accel_curve_type, acceleration_curve)
    else:
        curve_value = sample_curve(t, decel_curve_type, deceleration_curve)

    # Apply curved interpolation
    current_speed = lerp(current_speed, target_speed, curve_value)

    # Reset timer when target changes
    if sign(speed_diff) != sign(target_speed - current_speed):
        accel_timer = 0.0

func sample_curve(t: float, type: CurveType, custom_curve: Curve = null) -> float:
    # If custom curve provided, use it
    if custom_curve != null:
        return custom_curve.sample(t)

    # Otherwise use preset curve types
    match type:
        CurveType.LINEAR:
            return t

        CurveType.EASE_IN:
            return t * t  # Quadratic

        CurveType.EASE_OUT:
            return t * (2.0 - t)  # Inverse quadratic

        CurveType.EASE_IN_OUT:
            if t < 0.5:
                return 2.0 * t * t
            else:
                return -1.0 + (4.0 - 2.0 * t) * t

        CurveType.CUBIC:
            return t * t * t  # Cubic for aggressive feel

        CurveType.EXPONENTIAL:
            return 1.0 - pow(2.0, -10.0 * t)  # Exponential ease-out

    return t  # Fallback

# Alternative: Physics-based acceleration (feels more natural)
func apply_physics_acceleration(delta: float) -> void:
    var speed_diff = target_speed - current_speed

    if abs(speed_diff) < 0.01:
        current_speed = target_speed
        return

    # Calculate acceleration force
    var is_accelerating = speed_diff > 0
    var accel_rate = max_speed / (acceleration_time if is_accelerating else deceleration_time)

    # Apply with curve multiplier
    var t = abs(current_speed / max_speed)
    var curve_multiplier: float

    if is_accelerating:
        # Ease-out: fast start, slow finish
        curve_multiplier = 1.0 - (t * t)  # Decreases as speed increases
    else:
        # Ease-in: slow start, fast finish
        curve_multiplier = 1.0 - ((1.0 - t) * (1.0 - t))

    var acceleration = sign(speed_diff) * accel_rate * curve_multiplier * delta
    current_speed += acceleration
    current_speed = clamp(current_speed, 0.0, max_speed)

# Separate horizontal axes (for more control)
class_name DirectionalAcceleration:
    var velocity_x: float = 0.0
    var velocity_z: float = 0.0
    var accel_curve: Curve

    func update(target_x: float, target_z: float, delta: float, accel_time: float):
        velocity_x = apply_curve(velocity_x, target_x, delta, accel_time)
        velocity_z = apply_curve(velocity_z, target_z, delta, accel_time)

    func apply_curve(current: float, target: float, delta: float, accel_time: float) -> float:
        var diff = target - current
        if abs(diff) < 0.01:
            return target

        var step = (max_speed / accel_time) * delta
        var t = abs(current / max_speed)
        var curve_mult = accel_curve.sample(t) if accel_curve else (1.0 - t * t)

        return current + sign(diff) * step * curve_mult
```

**Key Parameters:**
- **acceleration_time:** 0.1-0.4s (0.2s feels responsive, 0.3s weighty)
- **deceleration_time:** 0.1-0.25s (usually faster than acceleration)
- **curve_type:** Ease-out for acceleration (snappy), ease-in for deceleration (controllable)
- **max_speed:** 8-15 units/s for platformers, 20-40 for action games
- **curve_strength:** 2.0 (quadratic) to 4.0 (quartic) - higher = more aggressive
- **deadzone:** 0.15-0.25 for analog sticks (ignore small inputs)

**Edge Cases:**
- **Direction changes:** Reset acceleration timer or blend between curves
- **Zero-to-zero:** Skip acceleration entirely if both current and target are zero
- **Framerate variance:** Use delta time, but very low framerates (<20fps) may feel choppy
- **Input buffering:** Store input during curve for responsive reversal
- **Steep slopes:** Apply separate vertical acceleration curve
- **Collision:** Immediate stop vs curved deceleration (design choice)

**When NOT to Use:**
- **Instant response games:** Fighting games, rhythm games need frame-perfect input
- **Realistic racing sims:** Use actual tire friction/engine curves instead
- **High-speed arcade games:** Sonic, racing games want instant top speed
- **Precise pixel-perfect platformers:** Traditional Mario-style needs predictable physics
- **Flight simulators:** Use realistic thrust/drag models
- **Top-down shooters:** Often want instant 8-directional movement

**Examples from Shipped Games:**

1. **Celeste:** 0.08s ground acceleration, 0.21s air deceleration. Ease-out curve on acceleration makes jumps feel responsive while maintaining control. Ground deceleration faster than air for tight platforming. Curve values heavily playtested - 0.01s changes noticeable.

2. **Ori and the Blind Forest:** Exponential ease-out on acceleration (fast initial boost, gradual top speed). 0.15s to 80% speed, 0.3s to 100%. Creates "light" feeling while maintaining control. Deceleration uses ease-in for precise stopping.

3. **Doom Eternal:** Minimal acceleration curves (0.05-0.1s) for responsive combat. Ease-out on sprint acceleration (0.2s to max). Instant strafe direction changes. Prioritizes combat responsiveness over "realistic" movement.

4. **The Last of Us Part II:** 0.3-0.5s acceleration for weighty, realistic feel. Cubic ease-in-out curves. Different curves for walking/running/sprinting. Longer curves during stealth (0.6s) for careful movement. Animation-driven blending.

5. **Super Meat Boy:** Almost no acceleration (0.02-0.05s) for instant control. Slight ease-out on direction changes to prevent "ice skating." Deceleration instant on input release. Precision platforming demands immediate response.

**Platform Considerations:**
- **PC (Keyboard):** Longer curves (0.2-0.3s) smooth out digital input
- **Console (Controller):** Shorter curves (0.1-0.2s), analog stick provides natural easing
- **Mobile (Touch):** Virtual joystick needs aggressive curves (0.3-0.4s) for smooth feel
- **High refresh rate (120Hz+):** Curves feel smoother, can use more aggressive shapes
- **Low-end hardware:** Pre-calculate curve tables, avoid real-time pow() calls

**Godot-Specific Notes:**
- Use `Curve` resource for visual editing in inspector (drag points, see preview)
- `Curve.sample(t)` is optimized, ~0.001ms per call
- Export curves with `@export var curve: Curve` for per-character tuning
- Animation Tree can blend acceleration animations with curves
- Consider `Tween` for simple cases but manual control better for gameplay
- Godot 4.x: `Curve2D/3D` for path-based movement, `Curve` for value interpolation
- Physics mode: Use `PhysicsBody3D` with custom `_integrate_forces()` for curves

**Synergies:**
- **Sprint Delay/Acceleration (#91):** Sprint uses separate, longer acceleration curve
- **Separate Air/Ground Acceleration (#94):** Different curves for grounded vs airborne
- **Air Strafing (#83):** Ground uses curves, air uses different acceleration model
- **Momentum Curves (#89):** Acceleration curves feed into momentum preservation
- **Jump Height Variability (#85):** Jump velocity affected by current acceleration state
- **Animation blending:** Curve progress drives animation blend weights

**Measurement/Profiling:**
- **Stopping distance:** Measure units traveled during deceleration (should be <2 character widths)
- **Time to max speed:** A/B test 0.15s vs 0.25s vs 0.35s (sweet spot usually 0.2s)
- **Direction change speed:** Time to reverse from max speed (target <0.3s total)
- **Player feedback:** Survey "Does movement feel responsive?" (target >80% positive)
- **Precision metrics:** Success rate on tight platforming challenges
- **Frame time:** Curve sampling <0.01ms, negligible impact
- **Input latency:** Total time from input to visible movement (target <100ms)
- **Playtesting:** Watch for "ice skating" or "stuck in mud" complaints
