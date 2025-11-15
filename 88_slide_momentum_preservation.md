### 88. Slide Momentum Preservation

**Category:** Game Feel - Movement

**Problem It Solves:** Sliding that instantly stops or ignores current speed feels disconnected and unrewarding. Players can't maintain momentum through combat encounters. No incentive to optimize speed before sliding. Slide feels like separate "mode" rather than natural movement extension. Movement lacks flow state and chain combos.

**Technical Explanation:**
Slide velocity inherits from pre-slide speed plus slide-specific acceleration. System captures horizontal velocity magnitude on slide initiation, applies slide friction curve over duration. Implementation: `slide_velocity = max(min_slide_speed, current_speed * preservation_factor)`, then `slide_velocity -= friction * delta`. Slide typically 0.5-1.5s duration with exponential decay. Exit velocity transfers to normal movement: `velocity = slide_direction * slide_velocity * exit_multiplier`. Key design: higher entry speed = longer slide distance and duration. Physics uses separate collision shape (capsule becomes box/cylinder). Slope detection multiplies slide speed on downhill (1.2-2.0x), reduces on uphill (0.5-0.8x). Creates skill expression through speed optimization before sliding.

**Algorithmic Complexity:**
- Velocity capture: O(1) - single vector magnitude
- Per-frame friction: O(1) - multiplication and subtraction
- Slope calculation: O(1) - dot product with ground normal
- Memory: ~32 bytes (slide state, velocity, timer)
- CPU: <0.01ms per character

**Implementation Pattern:**
```gdscript
class_name SlideController extends CharacterBody3D

# Slide parameters
@export var min_slide_speed: float = 8.0  # Minimum speed to initiate slide
@export var max_slide_speed: float = 20.0  # Speed cap during slide
@export var slide_duration: float = 1.0  # Base slide duration
@export var momentum_preservation: float = 0.95  # Keep 95% of entry speed

# Friction and acceleration
@export var slide_friction: float = 5.0  # Deceleration per second
@export var slide_initial_boost: float = 1.2  # 20% speed boost on entry
@export var slope_speed_multiplier: float = 1.5  # Downhill bonus
@export var uphill_speed_penalty: float = 0.6  # Uphill slowdown

# Exit parameters
@export var exit_speed_multiplier: float = 0.9  # Speed retention on exit
@export var slide_jump_bonus: float = 1.3  # Jump from slide = 30% boost
@export var can_cancel_slide: bool = true  # Allow early exit

# State tracking
var is_sliding: bool = false
var slide_timer: float = 0.0
var slide_velocity: float = 0.0
var slide_direction: Vector3 = Vector3.ZERO
var entry_speed: float = 0.0

# Collision shape switching
@onready var standing_collision: CollisionShape3D = $StandingCollision
@onready var sliding_collision: CollisionShape3D = $SlidingCollision
@onready var ceiling_check: RayCast3D = $CeilingCheck

func _physics_process(delta: float) -> void:
    if is_sliding:
        update_slide(delta)
    else:
        check_slide_input()

    move_and_slide()

func check_slide_input() -> void:
    # Check slide input (crouch while moving fast)
    if Input.is_action_just_pressed("crouch") and is_on_floor():
        var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()

        if horizontal_speed >= min_slide_speed:
            start_slide()

func start_slide() -> void:
    is_sliding = true
    slide_timer = 0.0

    # Capture entry velocity
    var horizontal_velocity = Vector3(velocity.x, 0, velocity.z)
    entry_speed = horizontal_velocity.length()
    slide_direction = horizontal_velocity.normalized()

    # Calculate initial slide velocity with preservation and boost
    slide_velocity = entry_speed * momentum_preservation * slide_initial_boost
    slide_velocity = min(slide_velocity, max_slide_speed)

    # Switch collision shape
    standing_collision.disabled = true
    sliding_collision.disabled = false

    # Trigger effects
    play_slide_start_effects()

func update_slide(delta: float) -> void:
    slide_timer += delta

    # Check exit conditions
    if should_exit_slide():
        exit_slide(false)
        return

    # Apply slide physics
    apply_slide_movement(delta)

    # Check for jump input (slide jump)
    if Input.is_action_just_pressed("jump"):
        exit_slide(true)

func should_exit_slide() -> bool:
    # Auto-exit after duration
    if slide_timer >= slide_duration:
        return true

    # Exit if speed too low
    if slide_velocity < min_slide_speed * 0.5:
        return true

    # Exit if not on floor
    if not is_on_floor():
        return true

    # Manual cancel
    if can_cancel_slide and Input.is_action_just_released("crouch"):
        # Check if ceiling blocks standing up
        if not ceiling_check.is_colliding():
            return true

    return false

func apply_slide_movement(delta: float) -> void:
    # Calculate slope influence
    var slope_multiplier = calculate_slope_multiplier()

    # Apply friction with slope consideration
    var effective_friction = slide_friction * (1.0 / slope_multiplier)
    slide_velocity -= effective_friction * delta
    slide_velocity = max(0.0, slide_velocity)

    # Optional: Slight speed boost on downhill
    if slope_multiplier > 1.1:
        slide_velocity += (slope_multiplier - 1.0) * 10.0 * delta
        slide_velocity = min(slide_velocity, max_slide_speed * slope_multiplier)

    # Update slide direction based on slope
    update_slide_direction()

    # Apply velocity
    velocity.x = slide_direction.x * slide_velocity
    velocity.z = slide_direction.z * slide_velocity

    # Maintain ground contact
    velocity.y = -2.0  # Small downward force

func calculate_slope_multiplier() -> float:
    # Get floor normal
    var floor_normal = get_floor_normal()
    if floor_normal == Vector3.ZERO:
        return 1.0

    # Calculate slope angle (dot with up vector)
    var slope_dot = floor_normal.dot(Vector3.UP)
    var slope_angle = acos(slope_dot)

    # Determine if uphill or downhill
    var movement_up_slope = slide_direction.dot(Vector3.UP)
    var is_downhill = floor_normal.dot(slide_direction) < 0

    if abs(slope_angle) < 0.1:  # Flat ground
        return 1.0
    elif is_downhill:
        # Steeper downhill = more speed
        var angle_factor = slope_angle / (PI * 0.25)  # Normalize to 45°
        return 1.0 + (slope_speed_multiplier - 1.0) * angle_factor
    else:
        # Uphill = speed penalty
        var angle_factor = slope_angle / (PI * 0.25)
        return 1.0 - (1.0 - uphill_speed_penalty) * angle_factor

func update_slide_direction() -> void:
    # Align slide direction with slope
    var floor_normal = get_floor_normal()
    if floor_normal != Vector3.ZERO:
        # Project slide direction onto slope
        slide_direction = slide_direction - slide_direction.project(floor_normal)
        slide_direction = slide_direction.normalized()

func exit_slide(jumped: bool) -> void:
    is_sliding = false

    # Restore collision shape
    standing_collision.disabled = false
    sliding_collision.disabled = true

    if jumped:
        # Slide jump with momentum boost
        apply_slide_jump()
    else:
        # Normal exit - preserve velocity
        var exit_velocity = slide_velocity * exit_speed_multiplier
        velocity.x = slide_direction.x * exit_velocity
        velocity.z = slide_direction.z * exit_velocity

    play_slide_end_effects()

func apply_slide_jump() -> void:
    # Combine slide momentum with jump
    var jump_velocity = slide_velocity * slide_jump_bonus

    velocity.x = slide_direction.x * jump_velocity
    velocity.z = slide_direction.z * jump_velocity
    velocity.y = 15.0  # Jump impulse

    # Optional: Extra boost for well-timed jumps
    if slide_timer < 0.3:  # Early jump = more boost
        velocity.x *= 1.1
        velocity.z *= 1.1

func play_slide_start_effects() -> void:
    $SlideParticles.emitting = true
    $SlideSound.play()

func play_slide_end_effects() -> void:
    $SlideParticles.emitting = false

# Advanced: Dynamic slide duration based on speed
func calculate_dynamic_slide_duration(speed: float) -> float:
    # Faster entry = longer slide
    var speed_ratio = speed / max_slide_speed
    return slide_duration * (0.5 + speed_ratio * 0.5)  # 50-100% of base duration

# Alternative: Exponential friction curve
func apply_exponential_friction(delta: float) -> void:
    # Speed decreases faster at end of slide
    var t = slide_timer / slide_duration
    var friction_curve = 1.0 + t * t * 2.0  # Increases exponentially

    slide_velocity -= slide_friction * friction_curve * delta
    slide_velocity = max(0.0, slide_velocity)

# Advanced: Directional control during slide
@export var slide_turn_speed: float = 1.5  # Radians per second

func apply_slide_steering(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")

    if abs(input_dir.x) > 0.1:
        # Rotate slide direction based on input
        var camera_basis = $Camera3D.global_transform.basis
        var turn_amount = input_dir.x * slide_turn_speed * delta

        # Rotate around Y axis
        var rotation_axis = Vector3.UP
        slide_direction = slide_direction.rotated(rotation_axis, turn_amount)

        # Speed penalty for turning
        slide_velocity *= 0.98  # Lose 2% speed per turn input
```

**Key Parameters:**
- **min_slide_speed:** 6-10 units/s (8 units/s typical minimum)
- **momentum_preservation:** 0.85-1.0 (0.95 = keep 95% of speed)
- **slide_initial_boost:** 1.0-1.3 (1.2 = 20% boost on entry)
- **slide_duration:** 0.6-1.5s (1.0s balanced)
- **slide_friction:** 3-8 units/s² (5 = moderate deceleration)
- **slope_multiplier:** 1.3-2.0x for downhill (1.5x typical)
- **slide_jump_bonus:** 1.2-1.5x (1.3 = 30% boost)

**Edge Cases:**
- **Slope transitions:** Recalculate multiplier when floor normal changes
- **Edge/cliff:** Maintain slide velocity when becoming airborne
- **Stairs:** Treat as slope (don't break slide on steps)
- **Ceiling during slide:** Force slide continuation, block standing
- **Water/hazard:** End slide immediately, apply hazard effect
- **Collision during slide:** Reduce velocity 50%, continue if above min speed
- **Moving platforms:** Add platform velocity to slide velocity

**When NOT to Use:**
- **Realistic tactical shooters:** Slide feels too arcadey
- **Top-down games:** Perspective doesn't support slide visual
- **Turn-based games:** No real-time movement
- **Slow-paced stealth:** Slide is too fast/loud for stealth focus
- **Platformers without combat:** May not fit movement theme
- **Racing games:** Use dedicated drift mechanics instead

**Examples from Shipped Games:**

1. **Apex Legends:** Slide preserves 100% momentum, lasts 0.8-1.2s based on entry speed. Downhill slides can reach 12 m/s (vs 7.3 m/s sprint). Slide jump = 1.25x speed multiplier. Chaining slide-jump-slide (bunny hop) maintains speed. Pro players optimize pre-slide speed. No cooldown enables constant sliding.

2. **Titanfall 2:** 110% momentum preservation + 1.3x initial boost. Slide duration 1.0-1.8s, speed-dependent. Can slide-hop to preserve speed indefinitely (advanced tech). Exit jump = 1.4x multiplier. Slide can be chained with wall run for 20+ m/s. Critical for competitive movement.

3. **Doom Eternal (2020):** Slide preserves 90% momentum, 0.7s fixed duration. Slight initial boost (1.1x). Primary use: repositioning during combat. Slide-jump launches player into air for aerial combat. Faster than sprinting for short distances. Tutorial teaches slide-jump within first level.

4. **Warframe:** 100% preservation + continuous acceleration during slide. Slide speed increases up to 200% over 2 seconds. No duration limit (can slide indefinitely). Bullet jump from slide = 1.6x multiplier. Downhill slides reach 25+ m/s. Core to "parkour 2.0" movement system.

5. **Ghostrunner:** 95% preservation, 0.6s duration. Slide-jump mandatory for most long gaps. Entry speed directly affects jump distance. Speedrunners optimize pre-slide routes. Combined with dash for 2x multiplier stack. Downhill slides can skip enemy encounters.

**Platform Considerations:**
- **PC (Keyboard):** Crouch toggle vs hold affects slide feel (hold better)
- **Console (Controller):** Button mapping critical (stick click vs bumper)
- **Mobile:** Swipe down to slide, automatic exit (no hold required)
- **VR:** Reduce slide speed 30% (motion sickness), increase camera stabilization
- **Competitive:** Ensure slide mechanics identical across platforms (frame-rate independent)
- **Low-end hardware:** Simplify particle trail, maintain core physics

**Godot-Specific Notes:**
- Use separate `CollisionShape3D` nodes, toggle `.disabled` property
- `get_floor_normal()` for slope detection (requires floor contact)
- `CharacterBody3D.floor_max_angle` to control slideable slopes
- `move_and_slide()` handles slope sliding automatically
- Ceiling check: `RayCast3D` pointing upward (1.5m range)
- Godot 4.x: Improved floor detection, better slope handling
- Animation: Use AnimationTree blend for slide-to-stand transition

**Synergies:**
- **Wall Running (#87):** Slide → jump → wall run preserves momentum
- **Air Strafing (#83):** Slide jump into air strafe for continued speed
- **Sprint Acceleration (#91):** Sprint → slide chains naturally
- **Momentum Curves (#89):** Slide feeds into global momentum system
- **Jump Height Variability (#85):** Slide jump can use bonus jump height
- **Dash/Boost:** Dash during slide stacks with slide speed
- **Crouch Jump (#93):** Slide exit into crouch jump combines bonuses

**Measurement/Profiling:**
- **Slide usage rate:** Slides per minute during gameplay (10-20 = healthy)
- **Average slide duration:** 0.7-1.1s typical (shorter = aggressive play)
- **Speed retention:** Exit speed vs entry speed (target 85-95%)
- **Slide-jump frequency:** % of slides ending in jump (skill metric, 40-60% good)
- **Distance traveled:** Average slide distance 8-15m at max speed
- **Slope exploitation:** Do players seek downhill routes? (good level design indicator)
- **Player feedback:** "Does slide feel powerful/useful?" (target >80% positive)
- **Performance:** Collision shape swap <0.02ms, slope calculation <0.01ms
