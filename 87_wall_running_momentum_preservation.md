### 87. Wall Running Momentum Preservation

**Category:** Game Feel - Movement

**Problem It Solves:** Wall running that kills momentum feels unsatisfying and breaks flow. Instant speed changes when entering/exiting walls make movement unpredictable. Players can't chain wall runs for speed optimization. Flat speed during wall run makes it feel "sticky" rather than dynamic. Momentum loss discourages creative routing and reduces skill ceiling.

**Technical Explanation:**
Preserve and modify player velocity when transitioning to/from wall running state. System captures pre-wall-run velocity magnitude, applies it as forward speed along wall surface. Implementation projects 3D velocity onto wall plane: `wall_velocity = velocity - velocity.project(wall_normal)`. Optional multipliers enhance momentum (1.1-1.3x speed boost entering wall, 1.2-1.5x exiting). Gravity reduction during wall run (30-50% normal) maintains height without full negation. Exit velocity calculation: combine wall tangent velocity + jump impulse + angle-based bonus. Critical math: `exit_velocity = wall_forward * wall_speed * exit_multiplier + wall_normal * jump_force`. Momentum curves can gradually increase speed during wall run (0.1-0.3s acceleration period).

**Algorithmic Complexity:**
- Wall detection: O(1) - raycast or collision callback
- Velocity projection: O(1) - vector dot/cross products
- Per-frame: O(1) - simple vector math
- Memory: ~48 bytes (wall normal, tangent, cached velocity)
- CPU: <0.02ms per character

**Implementation Pattern:**
```gdscript
class_name WallRunController extends CharacterBody3D

# Wall run parameters
@export var wall_run_speed: float = 12.0  # Base speed along wall
@export var wall_run_acceleration: float = 8.0  # Acceleration to wall speed
@export var momentum_preservation: float = 0.8  # Keep 80% of entry speed
@export var entry_speed_multiplier: float = 1.1  # 10% boost on entry
@export var exit_speed_multiplier: float = 1.3  # 30% boost on exit

# Gravity during wall run
@export var wall_run_gravity: float = 10.0  # Reduced gravity
@export var wall_run_duration_max: float = 3.0  # Max time on wall
@export var min_wall_run_speed: float = 5.0  # Fall off if slower

# Exit parameters
@export var wall_jump_force: float = 15.0  # Upward impulse
@export var wall_jump_outward_force: float = 10.0  # Away from wall
@export var angle_bonus_multiplier: float = 1.2  # Bonus for directional jumps

# State tracking
var is_wall_running: bool = false
var wall_normal: Vector3 = Vector3.ZERO
var wall_tangent: Vector3 = Vector3.ZERO
var wall_run_timer: float = 0.0
var entry_velocity: float = 0.0
var current_wall_speed: float = 0.0

# Detection
@onready var wall_ray_right: RayCast3D = $WallRayRight
@onready var wall_ray_left: RayCast3D = $WallRayLeft

func _physics_process(delta: float) -> void:
    if is_wall_running:
        update_wall_run(delta)
    else:
        check_wall_run_start()

    move_and_slide()

func check_wall_run_start() -> void:
    # Can only wall run if in air and moving fast enough
    if is_on_floor():
        return

    var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()
    if horizontal_speed < min_wall_run_speed:
        return

    # Check for wall collision
    var wall_hit: RayCast3D = null
    if wall_ray_right.is_colliding():
        wall_hit = wall_ray_right
    elif wall_ray_left.is_colliding():
        wall_hit = wall_ray_left

    if wall_hit == null:
        return

    # Valid wall found - start wall run
    start_wall_run(wall_hit)

func start_wall_run(wall_ray: RayCast3D) -> void:
    is_wall_running = true
    wall_run_timer = 0.0

    # Get wall surface normal
    wall_normal = wall_ray.get_collision_normal()

    # Calculate wall tangent (direction along wall)
    var up = Vector3.UP
    wall_tangent = up.cross(wall_normal).normalized()

    # Determine if running left or right along wall
    var velocity_horizontal = Vector3(velocity.x, 0, velocity.z)
    if velocity_horizontal.dot(wall_tangent) < 0:
        wall_tangent = -wall_tangent

    # Preserve entry momentum
    entry_velocity = velocity_horizontal.length()
    current_wall_speed = entry_velocity * momentum_preservation * entry_speed_multiplier

    # Project current velocity onto wall plane
    var wall_velocity = velocity - velocity.project(wall_normal)
    velocity = wall_velocity

    # Optional: Slight upward boost on entry
    velocity.y = max(velocity.y, 5.0)

    # Trigger effects
    play_wall_run_enter_effects()

func update_wall_run(delta: float) -> void:
    wall_run_timer += delta

    # Check exit conditions
    if should_exit_wall_run():
        exit_wall_run(false)
        return

    # Apply wall run movement
    apply_wall_run_movement(delta)

    # Check for jump input
    if Input.is_action_just_pressed("jump"):
        exit_wall_run(true)

func should_exit_wall_run() -> bool:
    # Exit if on ground, timer exceeded, or wall ended
    if is_on_floor():
        return true

    if wall_run_timer > wall_run_duration_max:
        return true

    # Check if still touching wall
    var touching_wall = wall_ray_right.is_colliding() or wall_ray_left.is_colliding()
    if not touching_wall:
        return true

    # Exit if speed too low
    if current_wall_speed < min_wall_run_speed * 0.5:
        return true

    return false

func apply_wall_run_movement(delta: float) -> void:
    # Accelerate toward wall run speed
    var target_speed = max(wall_run_speed, entry_velocity * momentum_preservation)
    current_wall_speed = move_toward(current_wall_speed, target_speed, wall_run_acceleration * delta)

    # Update wall tangent based on current orientation
    var up = Vector3.UP
    wall_tangent = up.cross(wall_normal).normalized()

    # Determine direction along wall based on input
    var input_dir = Input.get_vector("left", "right", "forward", "back")
    if input_dir.y < 0:  # Backward input
        wall_tangent = -wall_tangent

    # Apply movement along wall
    velocity.x = wall_tangent.x * current_wall_speed
    velocity.z = wall_tangent.z * current_wall_speed

    # Apply reduced gravity
    velocity.y -= wall_run_gravity * delta

    # Optional: Slight pull toward wall
    velocity += wall_normal * -2.0 * delta  # Stick to wall

func exit_wall_run(jumped: bool) -> void:
    is_wall_running = false

    if jumped:
        # Wall jump with momentum preservation
        apply_wall_jump()
    else:
        # Preserve momentum when falling off
        var exit_velocity = wall_tangent * current_wall_speed * exit_speed_multiplier
        velocity.x = exit_velocity.x
        velocity.z = exit_velocity.z
        # Keep current Y velocity

    play_wall_run_exit_effects()

func apply_wall_jump() -> void:
    # Calculate jump direction (away from wall + forward)
    var camera_forward = $Camera3D.global_transform.basis.z
    camera_forward.y = 0
    camera_forward = camera_forward.normalized()

    # Combine wall normal (outward) with forward direction
    var jump_direction = (wall_normal + camera_forward).normalized()

    # Apply directional bonus if jumping forward
    var forward_dot = camera_forward.dot(wall_tangent)
    var angle_bonus = 1.0 + abs(forward_dot) * (angle_bonus_multiplier - 1.0)

    # Calculate exit velocity
    var horizontal_velocity = jump_direction * current_wall_speed * exit_speed_multiplier * angle_bonus
    horizontal_velocity += wall_normal * wall_jump_outward_force

    velocity.x = horizontal_velocity.x
    velocity.z = horizontal_velocity.z
    velocity.y = wall_jump_force  # Upward impulse

    # Optional: Extra boost for chaining wall runs
    if wall_run_timer < 1.0:  # Quick exit = bonus
        velocity *= 1.1

func play_wall_run_enter_effects() -> void:
    # Particle trail, sound effect
    $WallRunParticles.emitting = true
    $WallRunSound.play()

func play_wall_run_exit_effects() -> void:
    $WallRunParticles.emitting = false
    $WallJumpSound.play()

# Advanced: Speed curve during wall run
@export var wall_run_speed_curve: Curve

func apply_speed_curve_movement(delta: float) -> void:
    var t = wall_run_timer / wall_run_duration_max
    var curve_multiplier = wall_run_speed_curve.sample(t) if wall_run_speed_curve else 1.0

    var target_speed = wall_run_speed * curve_multiplier
    current_wall_speed = move_toward(current_wall_speed, target_speed, wall_run_acceleration * delta)

    # ... rest of movement

# Alternative: Slope-based momentum
func calculate_slope_speed_multiplier(wall_angle: float) -> float:
    # Downward slopes = faster, upward = slower
    # wall_angle: 0 = horizontal, >0 = up, <0 = down
    var slope_factor = -wall_angle * 0.5  # -30° angle = 15° * 0.5 = 0.15 = +15% speed
    return 1.0 + clamp(slope_factor, -0.3, 0.5)  # -30% to +50%
```

**Key Parameters:**
- **momentum_preservation:** 0.7-1.0 (0.8 = keep 80% of entry speed)
- **entry_speed_multiplier:** 1.0-1.3 (1.1 = 10% boost entering wall)
- **exit_speed_multiplier:** 1.2-1.6 (1.3 = 30% boost leaving wall)
- **wall_run_gravity:** 5-15 (vs normal 30) = reduced gravity feel
- **wall_run_speed:** 10-15 units/s (base speed if entering slowly)
- **wall_jump_outward_force:** 8-12 units/s (push away from wall)

**Edge Cases:**
- **Corner transitions:** Switch wall_normal when detecting new wall, preserve speed
- **Curved walls:** Recalculate tangent each frame for smooth following
- **Ceiling collision:** End wall run, apply exit momentum
- **Convex vs concave corners:** Different handling for inner/outer corners
- **Moving platforms:** Add platform velocity to wall run speed
- **Damage during wall run:** Preserve some momentum but reduce by 50%
- **Angled walls:** Calculate tangent correctly for non-vertical surfaces

**When NOT to Use:**
- **Realistic games:** Modern military shooters, survival games
- **Slow-paced games:** Stealth, horror (wall running too fast/acrobatic)
- **2D games:** Wall slide/jump is more appropriate
- **Grid-based movement:** Tile-based games can't support free wall running
- **Games without vertical level design:** Flat levels make wall running pointless
- **Puzzle platformers:** Too much speed makes precision difficult

**Examples from Shipped Games:**

1. **Titanfall 2:** 120-150% speed boost chaining wall runs. Momentum fully preserved between walls. Can reach 15-20 m/s through proper chaining. Wall run duration ~2.5s with gradual speed increase. Exit jump preserves 110% of wall speed. Core mechanic defining entire movement meta.

2. **Mirror's Edge Catalyst:** 85% momentum preservation, 1.15x entry multiplier. Wall run accelerates from preserved speed to 12 m/s over 1.5s. Exit jump adds 8 m/s horizontal + 6 m/s vertical. Chaining 3+ walls grants "flow" speed bonus (+20%). Speedrunners optimize wall angles for maximum exit velocity.

3. **Warframe:** 100% momentum preservation + acceleration during run. Wall run speed increases 10% per second (max 150% after 5s). Bullet jump from wall = 1.4x speed multiplier. Parkour velocity caps at 30 m/s. Essential for fast mission completion (speedrun meta).

4. **Ghostrunner:** High momentum preservation (90%) but short duration (1.5s). Exit speed 1.5x multiplier encourages rapid wall-to-wall movement. Wall run maintains entry speed exactly - used for precise speedrun routes. Dash resets wall run timer (infinite potential).

5. **Sunset Overdrive:** 100% momentum + continuous acceleration (no cap). Grinding/wall-running shares momentum pool. Wall run speed can reach 25+ m/s through combos. Style meter increases speed multipliers (up to 2x). Encourages never touching ground.

**Platform Considerations:**
- **PC (Mouse):** Precise camera control enables complex wall-to-wall routing
- **Console (Controller):** Wider auto-assist angles (±15°), more forgiving wall detection
- **Mobile:** Simplified - auto-wall-run when near wall, swipe to jump off
- **VR:** Reduce speed 30-40% (motion sickness risk), increase gravity slightly
- **Low-end hardware:** Simplify particle effects, keep core momentum mechanics
- **High refresh rate:** Smoother transitions, can use more aggressive acceleration

**Godot-Specific Notes:**
- Use `CharacterBody3D` with `move_and_slide()` for wall detection
- Multiple `RayCast3D` nodes for reliable wall detection (left/right/front)
- `get_slide_collision()` provides wall normal during collision
- ShapeCast3D better than RayCast3D for complex wall shapes
- Godot 4.x: `get_wall_normal()` more reliable than 3.x
- Consider `PhysicsServer3D.body_test_motion()` for predictive wall detection
- Animation blending: Use AnimationTree with wall_run_speed as blend parameter

**Synergies:**
- **Air Strafing (#83):** Exit wall run into air strafe for continued speed
- **Slide Momentum (#88):** Wall run → land → slide preserves momentum chain
- **Double Jump:** Can double jump from wall without losing wall run speed
- **Momentum Curves (#89):** Wall run feeds into global momentum system
- **Dash/Boost:** Dash during wall run adds to current speed (stack multipliers)
- **Grapple Hook:** Grapple exit from wall run combines velocities
- **Sprint Acceleration (#91):** Wall run bypasses sprint windup

**Measurement/Profiling:**
- **Speed retention:** Average exit speed vs entry speed (target 110-130%)
- **Chain frequency:** % of wall runs followed by another wall run within 2s (skill metric)
- **Average wall run duration:** 0.8-1.5s typical (too short = frustrating, too long = boring)
- **Peak speeds achieved:** Track max velocity through wall running (balance check)
- **Player usage rate:** % of levels where players use wall running (should be high if available)
- **Speedrun optimization:** Do speedrunners find new routes via wall running? (good design)
- **Tutorial completion:** % players successfully wall run in tutorial (target >85%)
- **Performance:** Wall detection raycasts <0.05ms, movement calculation <0.01ms
