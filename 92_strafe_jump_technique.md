### 92. Strafe Jump Technique

**Category:** Game Feel - Movement

**Problem It Solves:** Ground movement caps at fixed speed with no skill expression. Players can't optimize routes for speed. Movement feels one-dimensional and lacks depth. Competitive play has low mechanical skill ceiling. No reward for mastering advanced movement techniques.

**Technical Explanation:**
Strafe jumping exploits velocity addition by jumping while strafing at specific angles to accumulate speed. Each jump adds velocity vector at angle to current movement, creating curved acceleration pattern. Implementation: when jumping while holding strafe key (A/D), add velocity perpendicular to view direction. Critical angle: 30-45° between current velocity and input direction maximizes acceleration. Physics: `new_velocity = current_velocity + (strafe_direction * strafe_accel * jump_bonus)`. Typically preserves 95-100% of horizontal velocity on landing. Ground friction higher than air friction (ground = 6.0, air = 0.3) incentivizes staying airborne. Success requires rhythm: jump → strafe left 30° → jump → strafe right 30° → repeat. Advanced players chain 10+ jumps to reach 150-300% base speed. Maps to Quake III's "circle jump" start technique.

**Algorithmic Complexity:**
- Per-jump calculation: O(1) - vector addition
- Angle optimization: O(1) - dot product
- No complex iterations
- Memory: ~32 bytes (velocity vectors, angle data)
- CPU: <0.01ms per jump event

**Implementation Pattern:**
```gdscript
class_name StrafeJumpController extends CharacterBody3D

# Base movement parameters
@export var ground_speed: float = 7.0
@export var air_speed: float = 8.0
@export var jump_velocity: float = 8.0

# Strafe jump specific
@export var strafe_jump_acceleration: float = 1.5  # Bonus accel when angled correctly
@export var optimal_strafe_angle: float = 30.0  # Degrees (sweet spot)
@export var angle_tolerance: float = 15.0  # Degrees (acceptable range)
@export var max_strafe_jump_speed: float = 25.0  # Speed cap (soft)
@export var ground_friction: float = 6.0
@export var air_friction: float = 0.3

# Velocity preservation
@export var velocity_retention_on_land: float = 0.98  # Keep 98% of speed
@export var perfect_bhop_window: float = 0.05  # Seconds for perfect timing bonus

# State tracking
var last_jump_time: float = 0.0
var consecutive_bhops: int = 0
var current_speed_bonus: float = 0.0

func _physics_process(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")

    if is_on_floor():
        apply_ground_movement(input_dir, delta)

        # Check for jump input
        if Input.is_action_just_pressed("jump"):
            perform_strafe_jump(input_dir)
    else:
        apply_air_movement(input_dir, delta)

    # Apply appropriate friction
    apply_friction(delta)

    move_and_slide()

func apply_ground_movement(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Ground acceleration
    var accel = 30.0
    velocity.x += direction.x * accel * delta
    velocity.z += direction.z * accel * delta

    # Clamp to ground speed (no boost on ground)
    var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()
    if horizontal_speed > ground_speed:
        var clamped = Vector3(velocity.x, 0, velocity.z).normalized() * ground_speed
        velocity.x = clamped.x
        velocity.z = clamped.z

func perform_strafe_jump(input_dir: Vector2) -> void:
    var current_time = Time.get_ticks_msec() / 1000.0
    var time_since_last_jump = current_time - last_jump_time

    # Check for perfect bunny hop timing
    var is_perfect_bhop = time_since_last_jump < perfect_bhop_window
    if is_perfect_bhop:
        consecutive_bhops += 1
    else:
        consecutive_bhops = 0

    # Apply jump velocity
    velocity.y = jump_velocity

    # Calculate strafe jump bonus
    if input_dir.length() > 0.1:
        var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
        var current_speed = horizontal_velocity.length()

        if current_speed > 0.1:
            var camera_basis = $Camera3D.global_transform.basis
            var input_direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

            # Calculate angle between current velocity and input
            var angle = rad_to_deg(horizontal_velocity.normalized().angle_to(input_direction))

            # Check if within optimal strafe angle range
            if abs(angle - optimal_strafe_angle) < angle_tolerance:
                # Apply strafe jump boost
                var boost = calculate_strafe_boost(angle, is_perfect_bhop)
                velocity.x += input_direction.x * boost
                velocity.z += input_direction.z * boost

                # Update speed bonus for consecutive bhops
                current_speed_bonus += 0.1 * consecutive_bhops
                current_speed_bonus = min(current_speed_bonus, 5.0)  # Cap bonus

    last_jump_time = current_time

func calculate_strafe_boost(angle: float, perfect_timing: bool) -> float:
    # Base boost scaled by how close to optimal angle
    var angle_diff = abs(angle - optimal_strafe_angle)
    var angle_factor = 1.0 - (angle_diff / angle_tolerance)  # 1.0 at perfect, 0.0 at edge

    var boost = strafe_jump_acceleration * angle_factor

    # Bonus for perfect bunny hop timing
    if perfect_timing:
        boost *= 1.3  # 30% bonus for perfect timing

    return boost

func apply_air_movement(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    # Air acceleration (allows strafe control)
    var air_accel = 20.0
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    # Only accelerate if not exceeding speed cap in that direction
    var projection = horizontal_velocity.dot(direction)
    if projection < air_speed or current_speed < max_strafe_jump_speed:
        velocity.x += direction.x * air_accel * delta
        velocity.z += direction.z * air_accel * delta

func apply_friction(delta: float) -> void:
    var friction = air_friction if not is_on_floor() else ground_friction

    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    if current_speed > 0.1:
        # Apply friction
        var friction_force = current_speed * friction * delta
        var new_speed = max(0.0, current_speed - friction_force)

        # Preserve direction
        velocity.x = (velocity.x / current_speed) * new_speed
        velocity.z = (velocity.z / current_speed) * new_speed

    # Decay speed bonus when on ground
    if is_on_floor():
        current_speed_bonus -= 1.0 * delta
        current_speed_bonus = max(0.0, current_speed_bonus)

# Advanced: Circle jump (starting technique)
func perform_circle_jump(input_dir: Vector2) -> void:
    # Special jump from standstill that starts strafe jump chain
    if Vector3(velocity.x, 0, velocity.z).length() < 1.0:
        velocity.y = jump_velocity

        # Apply initial forward + strafe velocity
        var camera_basis = $Camera3D.global_transform.basis
        var forward = -camera_basis.z
        var strafe = camera_basis.x * sign(input_dir.x) if abs(input_dir.x) > 0.1 else Vector3.ZERO

        var circle_direction = (forward + strafe).normalized()
        velocity.x = circle_direction.x * (ground_speed * 1.2)
        velocity.z = circle_direction.z * (ground_speed * 1.2)

# Alternative: Quake-style air acceleration
func apply_quake_air_accel(input_dir: Vector2, delta: float) -> void:
    if input_dir.length() < 0.1:
        return

    var camera_basis = $Camera3D.global_transform.basis
    var wish_dir = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = horizontal_velocity.length()

    # Calculate acceleration
    var add_speed = air_speed - horizontal_velocity.dot(wish_dir)
    if add_speed <= 0:
        return

    var accel_speed = strafe_jump_acceleration * air_speed * delta
    accel_speed = min(accel_speed, add_speed)

    velocity.x += wish_dir.x * accel_speed
    velocity.z += wish_dir.z * accel_speed

# UI feedback for learning
func get_strafe_angle_feedback() -> String:
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    if horizontal_velocity.length() < 1.0:
        return "No movement"

    var input_dir = Input.get_vector("left", "right", "forward", "back")
    if input_dir.length() < 0.1:
        return "No input"

    var camera_basis = $Camera3D.global_transform.basis
    var input_direction = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()
    var angle = rad_to_deg(horizontal_velocity.normalized().angle_to(input_direction))

    if abs(angle - optimal_strafe_angle) < angle_tolerance:
        return "OPTIMAL ANGLE! (+%.1f)" % calculate_strafe_boost(angle, false)
    else:
        return "Angle: %.0f° (optimal: %.0f°)" % [angle, optimal_strafe_angle]

# Speed tracking for skill progression
func get_speed_stats() -> Dictionary:
    var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()
    var speed_percentage = (horizontal_speed / max_strafe_jump_speed) * 100.0

    return {
        "current_speed": horizontal_speed,
        "speed_percentage": speed_percentage,
        "consecutive_bhops": consecutive_bhops,
        "speed_bonus": current_speed_bonus
    }
```

**Key Parameters:**
- **strafe_jump_acceleration:** 1.0-2.5 (1.5 balanced, 2.0+ = Quake-like)
- **optimal_strafe_angle:** 25-45° (30° = Quake standard)
- **angle_tolerance:** 10-20° (15° gives learnable window)
- **max_strafe_jump_speed:** 20-40 units/s (2-4x base speed)
- **ground_friction:** 4-8 (6 = moderate penalty for landing)
- **air_friction:** 0.2-0.5 (0.3 = minimal air drag)
- **velocity_retention:** 0.95-1.0 (0.98 = keep almost all speed)

**Edge Cases:**
- **Wall collision during bhop:** Preserve vertical velocity, reduce horizontal by 50%
- **Ceiling hit:** Set velocity.y = 0, keep horizontal momentum
- **Landing on slope:** Adjust velocity based on slope angle (preserve energy)
- **Water entry:** Strafe jump mechanics disabled in water
- **Damage during bhop:** Maintain momentum (don't reset on hit)
- **Controller input:** Wider angle tolerance (20°) for analog stick imprecision
- **Frame rate variance:** Use fixed physics step (120Hz ideal for precise timing)

**When NOT to Use:**
- **Realistic games:** Modern military shooters, survival games
- **Casual/mobile games:** Too complex for target audience
- **Story-driven games:** Can break pacing and cinematic sequences
- **Puzzle platformers:** Makes intended solutions trivial
- **Games requiring precise positioning:** Bhop creates too much speed
- **Competitive balance:** If strafe jumping gives unfair advantage (design choice)

**Examples from Shipped Games:**

1. **Quake III Arena / Quake Champions:** Original implementation. Optimal angle 30°, bhop maintains 100% velocity. Pro players reach 800-1000 ups (units per second). Circle jump start technique essential. Entire competitive meta built around movement. Map design accommodates high speeds. Tutorial teaches strafe jumping explicitly.

2. **Team Fortress 2:** Simplified strafe jumping on Soldier/Demo classes (rocket/sticky jumping). Scout has limited ground-based strafe jump. Air strafing 80% effectiveness of Quake. Max speed ~520 ups (vs Scout's 400 base). Critical for competitive play, especially Soldier in 6v6 format.

3. **CS:GO/CS2:** Heavily nerfed strafe jumping (bunny hopping). Speed cap at 300 units/s (vs 250 base). Landing penalty reduces speed 15% each jump. Prevents infinite acceleration. Still used for specific jumps/shortcuts. Balances between skill expression and gun-focused gameplay.

4. **Titanfall 2:** Advanced strafe jumping combined with grapple/slide. Bunny hopping essential for maintaining wallrun speed. Can reach 20+ m/s through chaining. Game actively teaches strafe jumping in gauntlet tutorial. Speedrun community entirely based on movement optimization.

5. **Half-Life / Black Mesa:** Classic Source engine strafe jumping. Longjump module enables advanced techniques. Speedrunners use bhop extensively (50% time save vs walking). Backward bhop even faster than forward. Engine quirk became iconic speedrun tech.

**Platform Considerations:**
- **PC (Mouse + Keyboard):** Optimal platform, precise angle control
- **Console (Controller):** Increase angle_tolerance to 20°, provide aim assist for optimal angle
- **Mobile:** Not recommended - too complex for touch controls
- **VR:** Reduced effectiveness (motion sickness risk from speed)
- **Competitive:** Require 60Hz+ physics tickrate for consistent timing
- **Crossplay:** Ensure controller players have angle assist (balance concern)

**Godot-Specific Notes:**
- Use fixed physics timestep: `Engine.physics_ticks_per_second = 120`
- `CharacterBody3D.velocity` for precise control
- `is_on_floor()` has 1-frame delay, add coyote time buffer
- Input buffering: Store jump input for 0.1s before landing
- Debug draw: Visualize velocity vector and optimal strafe angle
- Godot 4.x: Improved floor detection vs 3.x (more reliable bhop)
- AnimationTree: Blend jump animation based on speed
- Consider `PhysicsServer3D` for frame-perfect collision detection

**Synergies:**
- **Air Strafing (#83):** Core component - strafe jump uses air strafing between hops
- **Momentum Curves (#89):** Strafe jumping feeds into momentum system
- **Slide Momentum (#88):** Bhop → slide → bhop chains for max speed
- **Wall Running (#87):** Exit wall run into strafe jump for speed preservation
- **Jump Height Variability (#85):** Shorter jumps = faster bhop rhythm
- **Separate Air/Ground Accel (#94):** Critical for strafe jump mechanics
- **Speed Visualizer UI:** Show current speed and optimal angle indicator

**Measurement/Profiling:**
- **Top speed achieved:** Track max velocity reached through bhop (20-30 units/s = good design)
- **Average hop angle:** Mean angle of input during jumps (should cluster near optimal 30°)
- **Bhop success rate:** % of jumps that maintain >95% velocity (skill metric)
- **Consecutive hops:** Track longest bhop chain (10+ = skilled player)
- **Tutorial completion:** % players who successfully bhop in tutorial (target >60%)
- **Speed variance:** Standard deviation of speed during bhop chain (lower = more consistent)
- **Player feedback:** "Is strafe jumping learnable?" (target >70% yes after tutorial)
- **Performance:** Per-frame calculation <0.01ms (vector math only)
- **Competitive usage:** % of competitive players using strafe jumping (should be high if intended)
