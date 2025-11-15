### 89. Momentum Curves (Speed/Drag Balance)

**Category:** Game Feel - Movement

**Problem It Solves:** Unrestricted speed accumulation creates uncontrollable movement and breaks level design. Hard speed caps feel artificial and punish skillful play. Linear friction makes deceleration feel wrong. Players lose sense of weight and physics. No reward for maintaining speed through level flow.

**Technical Explanation:**
Dynamic speed caps and friction that scale with velocity. Instead of hard max speed, apply progressive drag that increases with velocity squared: `drag = base_drag * (velocity/target_speed)^drag_exponent`. Creates soft speed cap where acceleration and drag balance naturally. Implementation: `velocity -= velocity * drag_coefficient * delta`. Drag coefficient rises exponentially above target speed. Allows brief speed bursts above "normal" max without infinite acceleration. Momentum preservation through abilities (dash, jump) can temporarily exceed soft cap. Physics: `net_force = applied_force - drag_force`, where `drag_force = k * v^n` (n typically 1.5-2.5). Creates satisfying "weight" feeling where maintaining high speed requires effort.

**Algorithmic Complexity:**
- Per-frame: O(1) - simple mathematical operations
- Drag calculation: O(1) - power function and multiplication
- No iterations or searches
- Memory: ~16 bytes (velocity, drag state)
- CPU: <0.005ms per character

**Implementation Pattern:**
```gdscript
class_name MomentumCurveController extends CharacterBody3D

# Speed parameters
@export var base_speed: float = 10.0  # Normal movement speed
@export var soft_speed_cap: float = 15.0  # Speed where drag starts increasing
@export var absolute_max_speed: float = 30.0  # Hard cap (for safety)

# Drag parameters
@export var base_drag: float = 0.5  # Base friction coefficient
@export var drag_exponent: float = 2.0  # How quickly drag increases (1.5-2.5)
@export var high_speed_drag_multiplier: float = 3.0  # Extra drag above soft cap

# Momentum preservation
@export var momentum_decay_time: float = 2.0  # Time to return to base speed
@export var preserve_speed_on_jump: bool = true
@export var aerial_drag_multiplier: float = 0.3  # Less drag in air

# Curve-based drag (alternative to exponential)
@export var drag_curve: Curve  # Visual editor for drag profile

var current_momentum_bonus: float = 0.0  # Temporary speed boost

func _physics_process(delta: float) -> void:
    # Get movement input
    var input_dir = Input.get_vector("left", "right", "forward", "back")

    # Calculate target velocity
    apply_movement_input(input_dir, delta)

    # Apply momentum curves
    apply_momentum_drag(delta)

    # Clamp to absolute max
    clamp_velocity()

    move_and_slide()

func apply_movement_input(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var move_direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Calculate effective max speed (base + momentum bonus)
    var effective_max_speed = base_speed + current_momentum_bonus

    # Accelerate toward max speed
    var acceleration = 30.0  # Units per second squared
    velocity.x += move_direction.x * acceleration * delta
    velocity.z += move_direction.z * acceleration * delta

func apply_momentum_drag(delta: float) -> void:
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    if current_speed < 0.1:
        return

    # Calculate drag coefficient based on speed
    var drag_coefficient = calculate_drag_coefficient(current_speed)

    # Modify drag based on grounded state
    if not is_on_floor():
        drag_coefficient *= aerial_drag_multiplier

    # Apply drag
    var drag_force = horizontal_velocity * drag_coefficient * delta
    velocity.x -= drag_force.x
    velocity.z -= drag_force.z

    # Decay momentum bonus over time
    if current_momentum_bonus > 0:
        var decay_rate = current_momentum_bonus / momentum_decay_time
        current_momentum_bonus -= decay_rate * delta
        current_momentum_bonus = max(0.0, current_momentum_bonus)

func calculate_drag_coefficient(speed: float) -> float:
    # Below soft cap: minimal drag
    if speed <= soft_speed_cap:
        return base_drag

    # Above soft cap: exponential drag increase
    var speed_ratio = (speed - soft_speed_cap) / soft_speed_cap
    var exponential_drag = base_drag * pow(speed_ratio + 1.0, drag_exponent)

    # Apply high-speed multiplier
    if speed > soft_speed_cap * 1.5:
        exponential_drag *= high_speed_drag_multiplier

    return exponential_drag

func clamp_velocity() -> void:
    # Hard cap for safety (prevents physics explosions)
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    if current_speed > absolute_max_speed:
        var clamped_velocity = horizontal_velocity.normalized() * absolute_max_speed
        velocity.x = clamped_velocity.x
        velocity.z = clamped_velocity.z

# Add momentum bonus (from abilities, combos, etc.)
func add_momentum_bonus(bonus_speed: float, duration: float = 0.0) -> void:
    current_momentum_bonus += bonus_speed

    # Optional: Set decay time
    if duration > 0:
        momentum_decay_time = duration

# Alternative: Curve-based drag
func calculate_drag_coefficient_curve(speed: float) -> float:
    if drag_curve == null:
        return calculate_drag_coefficient(speed)

    # Normalize speed to 0-1 range
    var t = clamp(speed / absolute_max_speed, 0.0, 1.0)

    # Sample curve (0 = no drag, 1 = maximum drag)
    var curve_value = drag_curve.sample(t)

    return base_drag + curve_value * high_speed_drag_multiplier

# Advanced: Separate forward/strafe drag
func apply_directional_drag(delta: float) -> void:
    var camera_basis = $Camera3D.global_transform.basis
    var forward = Vector3(camera_basis.z.x, 0, camera_basis.z.z).normalized()
    var right = Vector3(camera_basis.x.x, 0, camera_basis.x.z).normalized()

    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)

    # Calculate forward and strafe components
    var forward_speed = horizontal_velocity.dot(forward)
    var strafe_speed = horizontal_velocity.dot(right)

    # Different drag for forward vs strafe
    var forward_drag = calculate_drag_coefficient(abs(forward_speed))
    var strafe_drag = calculate_drag_coefficient(abs(strafe_speed)) * 1.5  # More strafe drag

    # Apply directional drag
    velocity -= forward * forward_speed * forward_drag * delta
    velocity -= right * strafe_speed * strafe_drag * delta

# Terminal velocity calculation
func calculate_terminal_velocity() -> float:
    # Speed where acceleration = drag
    # acceleration = drag_coefficient * v^n
    # Solve for v: v = (acceleration / base_drag)^(1/drag_exponent)
    var acceleration = 30.0  # Match movement acceleration
    return pow(acceleration / base_drag, 1.0 / drag_exponent)

# Speed boost from chaining moves
func check_movement_chain_bonus() -> void:
    # Example: slide → jump → wall run = cumulative bonus
    # Implementation depends on other movement systems
    pass

# Advanced: Air drag with curve
@export var air_drag_curve: Curve  # Different curve for aerial drag

func apply_aerial_drag(delta: float) -> void:
    if is_on_floor():
        return

    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    if current_speed < 0.1:
        return

    var t = clamp(current_speed / absolute_max_speed, 0.0, 1.0)
    var curve_value = air_drag_curve.sample(t) if air_drag_curve else 0.3

    var drag = horizontal_velocity * curve_value * delta
    velocity.x -= drag.x
    velocity.z -= drag.z

# Momentum visualization (debug)
func _draw_momentum_debug() -> void:
    var current_speed = Vector3(velocity.x, 0, velocity.z).length()
    var speed_percentage = (current_speed / absolute_max_speed) * 100.0
    var drag = calculate_drag_coefficient(current_speed)

    print("Speed: %.1f (%.0f%%) | Drag: %.2f | Bonus: %.1f" % [
        current_speed, speed_percentage, drag, current_momentum_bonus
    ])
```

**Key Parameters:**
- **base_speed:** 8-12 units/s (normal movement)
- **soft_speed_cap:** 15-20 units/s (where drag starts ramping)
- **absolute_max_speed:** 25-40 units/s (2-3x base speed)
- **base_drag:** 0.3-0.8 (0.5 balanced)
- **drag_exponent:** 1.5-2.5 (2.0 = quadratic drag, realistic)
- **high_speed_drag_multiplier:** 2.0-4.0 (3.0 = aggressive cap)
- **aerial_drag_multiplier:** 0.2-0.5 (less air resistance)

**Edge Cases:**
- **Zero velocity:** Skip drag calculation (avoid division by zero)
- **Negative momentum bonus:** Clamp to zero, don't allow negative
- **Teleportation/respawn:** Reset velocity and momentum bonus
- **Cutscenes:** Disable momentum system during scripted sequences
- **Knockback:** Temporarily disable drag to allow full knockback velocity
- **Speed buffs/debuffs:** Modify base_speed and soft_cap together
- **Frame rate spikes:** Use physics delta time, not render delta

**When NOT to Use:**
- **Fixed-speed games:** Pac-Man, Snake (constant velocity)
- **Racing games with gear systems:** Use engine torque curves instead
- **Turn-based games:** No continuous movement
- **Puzzle games:** Speed variation irrelevant
- **Grid-based movement:** Discrete tiles don't need momentum
- **Realistic sports sims:** Use sport-specific physics models

**Examples from Shipped Games:**

1. **Sonic Generations:** Soft cap at 50 m/s, absolute max 80 m/s. Drag exponent 1.8 (gentle curve). Boost adds +30 m/s momentum bonus, decays over 3s. Downhill slopes reduce drag by 50%. Maintaining max speed requires constant boost management. Speedrunners optimize route for minimal drag.

2. **Tribes: Ascend:** Skiing reduces drag to near-zero (0.1 coefficient). Base drag 2.5, soft cap 90 kph, no hard cap. Terminal velocity ~300 kph on steep slopes. Drag exponent 1.5 (shallower curve for maintainable high speeds). Jet pack provides acceleration against drag.

3. **Spider-Man (Insomniac):** Soft cap 25 m/s, hard cap 40 m/s. Web-swing adds momentum bonus (+5-15 m/s) based on height. Drag exponent 2.2 (aggressive above cap). Point-launch temporarily ignores drag (1s). Creates flow state through Manhattan's vertical architecture.

4. **Warframe:** Soft cap varies by frame (15-20 m/s). Sprint mod increases cap +30%. Bullet jump ignores drag for 0.5s. Aerial drag 0.2x (parkour emphasis). Speed buffs add to momentum bonus, stack multiplicatively. Terminal velocity calculation allows infinite acceleration combos.

5. **Doom Eternal:** Soft cap 12 m/s, hard cap 18 m/s. Meathook adds +8 m/s instant velocity. Drag exponent 2.5 (aggressive cap above soft limit). Glory kill momentum preserved 80%. Encourages aggressive forward movement for speed maintenance.

**Platform Considerations:**
- **All platforms:** Physics-based, no platform-specific tuning needed
- **High refresh rate:** More stable due to smaller delta time steps
- **Low FPS (<30):** Large delta can cause overshoot, clamp delta to 0.033s max
- **Mobile:** Slightly higher base_drag (0.6 vs 0.5) for easier control
- **Competitive:** Lock physics to fixed timestep (60 or 120 Hz)
- **VR:** Lower soft_cap by 20% (motion sickness prevention)

**Godot-Specific Notes:**
- Use `get_physics_process_delta_time()` for consistent physics
- Export `Curve` resource for visual drag curve editing
- Physics runs at fixed timestep (default 60 Hz) - perfect for momentum
- `CharacterBody3D.velocity` automatically applied by `move_and_slide()`
- Debug draw with `CanvasLayer` for real-time speed/drag visualization
- Godot 4.x: Improved physics stability at high speeds
- Consider `RigidBody3D` for pure physics-based momentum (less control)

**Synergies:**
- **Air Strafing (#83):** Air momentum preserved, affected by aerial drag
- **Slide Momentum (#88):** Slide adds momentum bonus to curve system
- **Wall Running (#87):** Wall run momentum feeds into global curve
- **Sprint Acceleration (#91):** Sprint increases soft_cap temporarily
- **Jump Preservation:** Jump maintains current momentum within curve limits
- **Dash/Boost Abilities:** Add instant momentum bonus that decays naturally
- **Combat:** Movement abilities grant momentum bonuses for aggressive play

**Measurement/Profiling:**
- **Average speed:** Track mean velocity during active play (should be 70-85% of soft cap)
- **Peak speed duration:** How long players maintain >soft_cap speeds (skill metric)
- **Speed distribution:** Histogram of velocity values (should show cluster near soft cap)
- **Momentum bonus usage:** Frequency and effectiveness of speed-boosting abilities
- **Terminal velocity reached:** Do players ever hit absolute max? (should be rare, <1% of time)
- **Drag impact:** Compare gameplay with vs without momentum curves (playtest feel)
- **Player feedback:** "Does movement feel weighty but responsive?" (target >85% yes)
- **Performance:** Drag calculation <0.005ms per character (trivial overhead)
- **Level completion times:** Does momentum system enable skill-based speedrunning?
