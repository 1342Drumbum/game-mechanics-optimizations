### 86. Landing Recovery Animation

**Category:** Game Feel - Movement

**Problem It Solves:** Instant post-landing control feels disconnected from physics and removes weight from character. No landing feedback makes falls indistinguishable from steps. Players lose spatial awareness of height fallen. Movement lacks consequences and impact. Combat becomes "float spam" without landing vulnerability windows.

**Technical Explanation:**
Landing recovery introduces brief control restriction (0.1-0.4s) based on fall height/velocity. System calculates impact velocity `|velocity.y|` on ground contact, maps to recovery duration via linear or exponential curve. During recovery: reduce movement speed (50-80%), disable jump, play animation, optional screen shake. Implementation tracks `last_air_time` and `impact_velocity`, uses state machine (Airborne → Landing → Grounded). Recovery duration formula: `recovery_time = min(max_recovery, impact_velocity * time_scale)`. Animation blends from fall → land → idle using blend tree. Camera/screenshake intensity scales with impact. Critical for communicating weight and danger of falls.

**Algorithmic Complexity:**
- Impact calculation: O(1) - single velocity magnitude check
- State transition: O(1) - boolean flags
- Animation blending: O(1) - linear interpolation
- Memory: ~64 bytes (state, timers, cached velocity)
- CPU: <0.01ms per character

**Implementation Pattern:**
```gdscript
class_name LandingRecoveryController extends CharacterBody3D

# Landing parameters
@export var min_fall_height_for_recovery: float = 3.0  # Meters
@export var recovery_time_per_unit_velocity: float = 0.02  # Seconds per m/s
@export var max_recovery_time: float = 0.5  # Hard cap
@export var min_recovery_time: float = 0.1  # Minimum noticeable recovery

# Movement penalties during recovery
@export var recovery_speed_multiplier: float = 0.3  # 30% speed
@export var disable_jump_during_recovery: bool = true
@export var recovery_turn_speed_mult: float = 0.5  # Slower turning

# Impact effects
@export var camera_shake_intensity_scale: float = 0.02
@export var dust_particle_velocity_threshold: float = 15.0
@export var hard_landing_velocity_threshold: float = 25.0  # Play heavy sound

# State tracking
enum LandingState { GROUNDED, AIRBORNE, RECOVERING }
var landing_state: LandingState = LandingState.GROUNDED
var recovery_timer: float = 0.0
var impact_velocity: float = 0.0
var air_time: float = 0.0
var was_on_floor: bool = true

# References
@onready var animation_tree: AnimationTree = $AnimationTree
@onready var camera: Camera3D = $Camera3D
@onready var dust_particles: GPUParticles3D = $DustParticles

func _physics_process(delta: float) -> void:
    var is_grounded = is_on_floor()

    # State transitions
    update_landing_state(is_grounded, delta)

    # Apply movement with recovery penalty
    var movement_multiplier = 1.0
    if landing_state == LandingState.RECOVERING:
        movement_multiplier = recovery_speed_multiplier
        update_recovery(delta)

    apply_movement(delta, movement_multiplier)
    move_and_slide()

    # Update state tracking
    was_on_floor = is_grounded

func update_landing_state(is_grounded: bool, delta: float) -> void:
    match landing_state:
        LandingState.GROUNDED:
            if not is_grounded:
                # Just left ground
                landing_state = LandingState.AIRBORNE
                air_time = 0.0

        LandingState.AIRBORNE:
            air_time += delta
            if is_grounded and not was_on_floor:
                # Just landed
                trigger_landing()

        LandingState.RECOVERING:
            if recovery_timer <= 0.0:
                landing_state = LandingState.GROUNDED

func trigger_landing() -> void:
    # Calculate impact velocity (negative Y velocity)
    impact_velocity = abs(velocity.y)

    # Determine if recovery is needed
    var fall_height = calculate_approximate_fall_height()

    if fall_height < min_fall_height_for_recovery:
        # Soft landing, no recovery
        landing_state = LandingState.GROUNDED
        play_soft_landing_effects()
        return

    # Calculate recovery duration based on impact
    recovery_timer = calculate_recovery_duration(impact_velocity)

    if recovery_timer >= min_recovery_time:
        landing_state = LandingState.RECOVERING
        play_hard_landing_effects()
    else:
        landing_state = LandingState.GROUNDED
        play_soft_landing_effects()

    # Reset air time
    air_time = 0.0

func calculate_recovery_duration(impact_vel: float) -> float:
    # Linear scaling with velocity
    var duration = impact_vel * recovery_time_per_unit_velocity
    # Clamp to reasonable range
    return clamp(duration, min_recovery_time, max_recovery_time)

func calculate_approximate_fall_height() -> float:
    # Rough approximation: v² = 2gh → h = v²/(2g)
    var gravity = ProjectSettings.get_setting("physics/3d/default_gravity", 9.8)
    return (impact_velocity * impact_velocity) / (2.0 * gravity)

func update_recovery(delta: float) -> void:
    recovery_timer -= delta

    # Update animation blend
    if animation_tree:
        var recovery_progress = 1.0 - (recovery_timer / max_recovery_time)
        animation_tree.set("parameters/landing_blend/blend_amount", recovery_progress)

func apply_movement(delta: float, multiplier: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")

    # Apply recovery penalty
    input_dir *= multiplier

    # Prevent jumping during recovery
    if Input.is_action_just_pressed("jump"):
        if landing_state != LandingState.RECOVERING or not disable_jump_during_recovery:
            perform_jump()

    # ... rest of movement code

func play_hard_landing_effects() -> void:
    # Camera shake scaled by impact
    var shake_intensity = impact_velocity * camera_shake_intensity_scale
    camera_shake(shake_intensity, 0.15)

    # Play sound effect
    if impact_velocity > hard_landing_velocity_threshold:
        $HardLandingSound.play()
    else:
        $LandingSound.play()

    # Spawn dust particles
    if impact_velocity > dust_particle_velocity_threshold:
        dust_particles.emitting = true
        dust_particles.amount_ratio = min(impact_velocity / 30.0, 1.0)

    # Animation
    if animation_tree:
        animation_tree.set("parameters/landing_type/transition_request", "hard_landing")

func play_soft_landing_effects() -> void:
    # Minimal feedback for small falls
    if animation_tree:
        animation_tree.set("parameters/landing_type/transition_request", "soft_landing")

    # Subtle sound
    $SoftLandingSound.play()

func camera_shake(intensity: float, duration: float) -> void:
    # Simple camera shake implementation
    var shake_tween = create_tween()
    var original_pos = camera.position
    var shake_time = 0.0

    while shake_time < duration:
        var shake_offset = Vector3(
            randf_range(-intensity, intensity),
            randf_range(-intensity, intensity),
            0
        )
        shake_tween.tween_property(camera, "position", original_pos + shake_offset, 0.05)
        shake_time += 0.05

    shake_tween.tween_property(camera, "position", original_pos, 0.1)

# Advanced: Directional landing
func calculate_landing_direction() -> Vector2:
    # Determine forward/backward/side landing for animation
    var horizontal_vel = Vector2(velocity.x, velocity.z)
    var forward = Vector2(transform.basis.z.x, transform.basis.z.z)
    var right = Vector2(transform.basis.x.x, transform.basis.x.z)

    var forward_component = horizontal_vel.dot(forward)
    var right_component = horizontal_vel.dot(right)

    return Vector2(right_component, forward_component).normalized()

# Alternative: Exponential recovery curve
func calculate_recovery_duration_exponential(impact_vel: float) -> float:
    # Exponential scaling for more dramatic high falls
    var normalized_velocity = impact_vel / hard_landing_velocity_threshold
    var curve_value = pow(normalized_velocity, 1.5)  # Power curve
    return clamp(curve_value * max_recovery_time, min_recovery_time, max_recovery_time)
```

**Key Parameters:**
- **min_fall_height_for_recovery:** 2-5m (3m = ~0.6s fall time)
- **max_recovery_time:** 0.3-0.6s (0.5s max, longer feels unresponsive)
- **recovery_speed_multiplier:** 0.2-0.5 (0.3 = 70% speed reduction)
- **min_recovery_time:** 0.08-0.15s (0.1s minimum to be noticeable)
- **time_scale:** 0.01-0.03s per m/s impact velocity
- **camera_shake_intensity:** 0.01-0.05 (scaling factor)

**Edge Cases:**
- **Slope landings:** Reduce recovery on angled surfaces (<30° = 50% reduction)
- **Water/soft surfaces:** Override with custom recovery values (shorter/none)
- **Damage falls:** Separate "hurt" state with longer recovery
- **Forced landings:** Knockdown attacks bypass normal recovery calculations
- **Ledge grabs:** Cancel recovery if player grabs ledge immediately
- **Roll/dodge input:** Allow cancel of recovery with defensive move
- **Continuous small falls:** Debounce recovery trigger (0.2s minimum between)

**When NOT to Use:**
- **Fast-paced arena shooters:** Quake, Unreal - need constant mobility
- **Platformers prioritizing speed:** Sonic, runners - recovery breaks flow
- **Floaty/low-gravity games:** Space games, water levels
- **Top-down games:** 2D perspective doesn't convey fall impact
- **Puzzle platformers:** Recovery adds frustration to trial-and-error
- **Games with frequent small falls:** Would trigger constantly, annoying

**Examples from Shipped Games:**

1. **Dark Souls / Elden Ring:** 0.4-0.8s recovery based on fall height. Disables all actions (roll, attack, item use). Falls >20m trigger stagger + damage. Recovery animation can be roll-cancelled with precise timing. Critical for combat balance - landing vulnerability.

2. **The Last of Us Part II:** 0.2-0.4s recovery with animation. Speed reduced to 40% during recovery. Camera shake and dust particles scale with height. Different animations for forward/backward/side landings. Enhances stealth (loud landing alerts enemies).

3. **Titanfall 2:** Minimal recovery (0.05-0.1s) for pilots, significant (0.3-0.5s) for Titans. Pilots have powered armor justifying fast recovery. Titans shake screen heavily, momentarily disable dash. Maintains fast pilot gameplay while giving Titans weight.

4. **Monster Hunter World:** 0.5-1.0s recovery from high falls, with animation lock. Can be cancelled with superman dive (if sprinting). Heavy weapons have 20% longer recovery. Creates risk/reward for vertical positioning in combat.

5. **Assassin's Creed:** Variable recovery 0.2-0.6s. "Safe fall" height (6m) has no recovery. Haystack landing bypasses all recovery. Roll on landing (timed input) reduces recovery 50%. Encourages skillful landing technique.

**Platform Considerations:**
- **Console/PC:** Full recovery system with visual effects
- **Mobile:** Reduce recovery times 30-40% (smaller screens, less immersion)
- **VR:** Minimize screen shake (motion sickness), focus on audio feedback
- **Low-end hardware:** Skip particle effects, keep animation and timing
- **High refresh rate:** Smoother animation blending, can use shorter recovery times
- **Competitive multiplayer:** Ensure consistent recovery across all platforms

**Godot-Specific Notes:**
- Use `is_on_floor()` with raycast confirmation for reliable ground detection
- `CharacterBody3D.get_floor_velocity()` for moving platform landings
- AnimationTree blend spaces for directional landing animations
- `AudioStreamPlayer3D` with max_distance for spatial landing sounds
- GPUParticles3D for dust effects (more efficient than CPUParticles3D)
- Godot 4.x: Better floor detection, fewer false positives
- State machine: Use separate script or AnimationTree state machine

**Synergies:**
- **Jump Height Variability (#85):** Higher jumps = longer recovery
- **Wall Running (#87):** Wall dismount landing uses modified recovery
- **Slide Momentum (#88):** Landing during slide cancels recovery
- **Crouch/Roll:** Input during recovery triggers roll (advanced cancel)
- **Double Jump:** Prevents recovery by negating downward velocity
- **Ledge Grab (#90):** Alternative to landing recovery (grab instead of land)
- **Combat System:** Recovery creates attack/dodge windows for enemies

**Measurement/Profiling:**
- **Average recovery time:** Track mean recovery across playthrough (0.15-0.25s typical)
- **Recovery frequency:** Landings per minute triggering recovery (5-15 normal)
- **Player frustration metrics:** Survey if recovery feels fair (target >75% positive)
- **Animation timing:** Does recovery match animation duration exactly? (should sync)
- **Cancel success rate:** If cancelable, track % of successful cancels (skill expression)
- **Impact on combat:** Do enemies successfully punish landing recovery? (balance check)
- **Accessibility:** Playtesting with reduced mobility - is max recovery too long? (0.4s+ problematic)
- **Performance:** Particle systems on landing <0.5ms (GPU), animation <0.1ms (CPU)
