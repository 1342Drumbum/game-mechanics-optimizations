### 76. Aim Assist / Bullet Magnetism

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Controller aiming imprecision and player frustration with "almost hit" shots. Analog sticks provide ~100x less precision than mouse (stick: ±0.5° accuracy, mouse: ±0.005°). Without assistance, hit rates drop from 60-80% (mouse) to 15-35% (controller) in fast-paced games. Results in player frustration, perceived "bad hitboxes," and unbalanced PC vs. console gameplay. Critical for accessibility—enables players with motor impairments to compete.

**Technical Explanation:**
Aim assist comprises multiple techniques working together:

1. **Adhesion/Friction:** Slow down crosshair movement when over targets (20-40% speed reduction). Creates "sticky" feeling without removing control.

2. **Magnetism:** Subtle crosshair pull toward nearby targets (1-3° per frame). Applied when within activation radius (typically 5-15° visual angle).

3. **Bullet Magnetism:** Projectiles curve toward targets within cone (3-8° deviation). Happens post-firing, invisible to player but ensures "felt like a hit" shots connect.

4. **Hitbox Expansion:** Increase target collision size by 10-30% for projectiles. Easier to balance than magnetism—simply larger targets.

5. **Auto-Rotation:** Automatic crosshair tracking when aiming near target (controversial, used in aggressive aim assist). Typically disabled for multiplayer fairness.

Implementation uses raycasts or sphere casts to find targets within assist radius. Apply velocity modifiers to crosshair rotation (adhesion) or projectile trajectories (magnetism). Strength scales with distance—stronger at medium range, weaker at close/far range to preserve skill ceiling.

**Algorithmic Complexity:**
- Target detection: O(n) where n = nearby enemies (typically 3-10)
- Angle calculation: O(1) per target
- Assist application: O(1) velocity modification

**Implementation Pattern:**
```gdscript
# Godot implementation
extends Node3D

# Aim assist configuration
@export_group("Aim Assist")
@export var enable_aim_assist: bool = true
@export var assist_strength: float = 0.6  # 0.0 = none, 1.0 = maximum
@export var adhesion_strength: float = 0.35  # Slowdown when over target
@export var magnetism_strength: float = 0.25  # Pull toward target
@export var bullet_magnetism_angle: float = 5.0  # Degrees
@export var assist_radius: float = 15.0  # Degrees from crosshair

# Platform detection
var is_using_controller: bool = false
var assist_multiplier: float = 1.0

# Target tracking
var current_target: Node3D = null
var targets_in_range: Array[Node3D] = []

func _ready() -> void:
    # Detect input method
    Input.joy_connection_changed.connect(_on_controller_changed)
    is_using_controller = Input.get_connected_joypads().size() > 0

    # Disable aim assist for mouse/keyboard
    if not is_using_controller:
        assist_multiplier = 0.0

func _on_controller_changed(device: int, connected: bool) -> void:
    is_using_controller = Input.get_connected_joypads().size() > 0
    assist_multiplier = 1.0 if is_using_controller else 0.0

func _physics_process(delta: float) -> void:
    if not enable_aim_assist or assist_multiplier == 0.0:
        return

    # Find targets in assist range
    update_targets_in_range()

    # Apply aim assist to camera rotation
    if current_target:
        apply_aim_adhesion(delta)
        apply_aim_magnetism(delta)

func update_targets_in_range() -> void:
    targets_in_range.clear()
    current_target = null

    var camera = $Camera3D
    var camera_forward = -camera.global_transform.basis.z

    # Find all potential targets
    var enemies = get_tree().get_nodes_in_group("enemies")

    for enemy in enemies:
        if not is_valid_target(enemy):
            continue

        # Check if within assist radius
        var to_target = (enemy.global_position - camera.global_position).normalized()
        var angle = rad_to_deg(camera_forward.angle_to(to_target))

        if angle <= assist_radius:
            targets_in_range.append(enemy)

    # Sort by angle (closest to crosshair = highest priority)
    targets_in_range.sort_custom(func(a, b):
        var angle_a = get_angle_to_target(a)
        var angle_b = get_angle_to_target(b)
        return angle_a < angle_b
    )

    # Set closest as current target
    if targets_in_range.size() > 0:
        current_target = targets_in_range[0]

func is_valid_target(target: Node3D) -> bool:
    # Check if target is visible and alive
    if not is_instance_valid(target):
        return false

    if target.has_method("is_alive") and not target.is_alive():
        return false

    # Line of sight check
    var camera = $Camera3D
    var space_state = get_world_3d().direct_space_state

    var query = PhysicsRayQueryParameters3D.create(
        camera.global_position,
        target.global_position
    )
    query.exclude = [self]

    var result = space_state.intersect_ray(query)

    return result and result.collider == target

func get_angle_to_target(target: Node3D) -> float:
    var camera = $Camera3D
    var camera_forward = -camera.global_transform.basis.z
    var to_target = (target.global_position - camera.global_position).normalized()
    return rad_to_deg(camera_forward.angle_to(to_target))

func apply_aim_adhesion(delta: float) -> void:
    # Slow down aim sensitivity when over target
    var angle = get_angle_to_target(current_target)

    # Adhesion strength scales with proximity to crosshair center
    var proximity = 1.0 - (angle / assist_radius)
    var slowdown = adhesion_strength * proximity * assist_multiplier

    # Apply to input sensitivity (this would be in player controller)
    var base_sensitivity = 1.0
    var adjusted_sensitivity = base_sensitivity * (1.0 - slowdown)

    # Store for use in look input processing
    set_meta("aim_sensitivity_multiplier", adjusted_sensitivity)

func apply_aim_magnetism(delta: float) -> void:
    # Gently pull crosshair toward target
    var camera = $Camera3D
    var to_target = current_target.global_position - camera.global_position
    to_target = to_target.normalized()

    var camera_forward = -camera.global_transform.basis.z
    var pull_direction = camera_forward.lerp(to_target, magnetism_strength * assist_multiplier * delta * 5.0)

    # Apply rotation toward target
    var target_rotation = camera.global_transform.looking_at(
        camera.global_position + pull_direction,
        Vector3.UP
    )

    camera.global_transform = camera.global_transform.interpolate_with(
        target_rotation,
        magnetism_strength * assist_multiplier * delta * 3.0
    )

# Bullet magnetism (applied to projectiles)
func apply_bullet_magnetism(projectile: Node3D, initial_direction: Vector3) -> Vector3:
    if not enable_aim_assist or assist_multiplier == 0.0:
        return initial_direction

    var camera = $Camera3D

    # Find target within bullet magnetism cone
    var best_target: Node3D = null
    var best_angle: float = bullet_magnetism_angle

    for enemy in get_tree().get_nodes_in_group("enemies"):
        if not is_valid_target(enemy):
            continue

        var to_target = (enemy.global_position - projectile.global_position).normalized()
        var angle = rad_to_deg(initial_direction.angle_to(to_target))

        if angle < best_angle:
            best_angle = angle
            best_target = enemy

    # Apply magnetism if target found
    if best_target:
        var to_target = (best_target.global_position - projectile.global_position).normalized()
        var magnetism_factor = 1.0 - (best_angle / bullet_magnetism_angle)
        magnetism_factor *= assist_multiplier

        # Blend toward target direction
        return initial_direction.lerp(to_target, magnetism_factor * 0.4)

    return initial_direction
```

**Advanced Context-Aware Assist:**
```gdscript
class_name ContextualAimAssist

enum AssistLevel { NONE, LOW, MEDIUM, HIGH, MAXIMUM }

# Dynamic assist tuning
var current_assist_level: AssistLevel = AssistLevel.MEDIUM
var base_assist_strength: float = 0.6

# Situational modifiers
var distance_curve: Curve  # Stronger at medium range
var velocity_multiplier: float = 1.0  # Reduce for fast-moving players
var precision_bonus: float = 1.0  # Reward skilled players with less assist

func calculate_assist_strength(target: Node3D, player_velocity: Vector3) -> float:
    var base = base_assist_strength

    # Distance scaling
    var distance = global_position.distance_to(target.global_position)
    var distance_factor = evaluate_distance_curve(distance)

    # Velocity reduction (less assist when moving fast = skill-based)
    var speed = player_velocity.length()
    velocity_multiplier = clamp(1.0 - (speed / 10.0) * 0.3, 0.7, 1.0)

    # Player accuracy history (reduce assist for skilled players)
    precision_bonus = calculate_precision_bonus()

    return base * distance_factor * velocity_multiplier * precision_bonus

func evaluate_distance_curve(distance: float) -> float:
    # Strongest at medium range (10-30m)
    # Weaker at close range (<5m) to preserve skill
    # Weaker at long range (>50m) for fairness
    if distance_curve:
        return distance_curve.sample(distance / 100.0)

    # Default curve
    if distance < 5.0:
        return 0.5  # Close range
    elif distance < 30.0:
        return 1.0  # Sweet spot
    elif distance < 60.0:
        return lerp(1.0, 0.3, (distance - 30.0) / 30.0)
    else:
        return 0.3  # Long range

func calculate_precision_bonus() -> float:
    # Track player's hit accuracy over last 20 shots
    var recent_accuracy = get_meta("recent_hit_accuracy", 0.5)

    # Reduce assist for players with >70% accuracy
    if recent_accuracy > 0.7:
        return 0.7  # Less assistance for skilled players
    elif recent_accuracy < 0.3:
        return 1.3  # More assistance for struggling players

    return 1.0
```

**Key Parameters:**
- **Adhesion strength:** 0.2-0.5 (typical: 0.35)
- **Magnetism strength:** 0.15-0.35 (typical: 0.25)
- **Bullet magnetism angle:** 3-8° (typical: 5°)
- **Assist radius:** 10-20° visual angle
- **Distance curve:** Peak at 15-30m, falloff at edges

**Game genre variations:**
- **Competitive FPS:** Low (0.2-0.3), skill-based
- **Casual FPS:** Medium (0.4-0.6), accessibility focus
- **Single-player action:** High (0.5-0.8), power fantasy
- **Battle royale:** Medium (0.3-0.5), balanced
- **Horror games:** Low/none (tension from difficulty)

**Edge Cases:**
- **Multiple targets:** Prioritize closest to crosshair, not closest distance
- **Target switching:** Add hysteresis to prevent rapid switching
- **ADS vs hip-fire:** Stronger assist when aiming down sights
- **Different weapons:** Snipers get less assist, shotguns more
- **Moving targets:** Lead target prediction for better assist
- **Friendly fire:** Disable assist on friendlies
- **Destructibles:** Don't assist to barrels/objects
- **Through walls:** Only assist on visible targets
- **Input device switching:** Instantly adjust assist when switching mouse↔controller

**When NOT to Use:**
- Mouse/keyboard input (provides unfair advantage)
- Competitive esports (controller players already at disadvantage)
- Hardcore/realistic games where difficulty is core
- Horror games where missing shots creates tension
- When community explicitly rejects it
- Crossplay games (unless input-based matchmaking)

**Examples from Shipped Games:**

1. **Halo series (2001-present):** Gold standard for aim assist. Red reticle range (RRR) indicates assist active. Magnetism: ~4-6°, adhesion: 40% slowdown. Bungie documented "if the player felt like they hit, they should hit." Different assist per weapon—sniper has minimal, assault rifle has strong.

2. **Call of Duty: Modern Warfare (2019):** Controversial strong aim assist. Adhesion: ~35%, bullet magnetism: 5-7°, hitbox expansion: ~15%. Crossplay separated by input method after community backlash. Reduced assist at long range to balance sniping.

3. **Apex Legends (2019):** Different assist for PC (0.4x) vs console (0.6x). Adhesion-based, minimal magnetism. Distance scaling: strong at 10-40m, weak beyond 60m. Controversial in competitive scene—some players use controller on PC for assist advantage.

4. **Destiny 2 (2017):** Adaptive assist based on weapon archetype. Hand cannons: low assist, high skill ceiling. Auto rifles: higher assist, lower skill ceiling. Aim assist stat visible on weapons (25-100 range). Community accepts it as core mechanic, not controversial.

5. **The Last of Us Part II (2020):** Accessibility-focused with granular control. Aim assist strength: 0-100% adjustable. Separate settings for adhesion, magnetism, and aim lock. Enables players with disabilities to experience game. Up to 80% assist in accessibility mode.

**Platform Considerations:**
- **Console (controller only):** Standard assist (0.5-0.7)
- **PC (mixed input):** Disable for mouse, enable for controller
- **Crossplay:** Input-based matchmaking or adjust assist levels
- **Mobile (touch):** Higher assist (0.6-0.8) due to occlusion issues
- **VR:** Minimal/none (1:1 hand tracking expects precision)

**Godot-Specific Notes:**
- Use `Input.get_connected_joypads()` to detect controllers
- `Input.joy_connection_changed` signal for hot-swapping
- Physics raycasts for line-of-sight checks
- `Transform3D.looking_at()` for rotation calculations
- Export assist strength for per-platform tuning
- Consider `Curve` resource for distance falloff
- Profile raycast cost; limit to 5-10 targets per frame

**Synergies:**
- **Hitbox expansion:** Stack with bullet magnetism for maximum forgiveness
- **Recoil compensation:** Reduce recoil when aim assist active
- **Auto-aim toggles:** L2/LT to activate strong assist (controversial)
- **Visual feedback:** Crosshair color change when assist active
- **Audio cues:** Subtle sound when magnetism pulls toward target

**Measurement/Profiling:**
- **Hit rate analysis:**
  - Mouse: 60-80% in most shooters
  - Controller no assist: 15-35%
  - Controller with assist: 50-70%
  - Target: Match mouse within 5-10%
- **Assist activation tracking:**
  - % of shots that used assist
  - Average magnetism angle applied
  - Adhesion duration per engagement
- **Player satisfaction surveys:**
  - "Do controls feel responsive?" (target: >85% yes)
  - "Do you feel shots register fairly?" (target: >80% yes)
- **A/B testing:**
  - Vary assist strength 0.3-0.7
  - Measure: K/D ratio, hit %, player retention
- **Competitive balance:**
  - Controller vs mouse win rates
  - Should be within 5% in crossplay

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    var camera = $Camera3D
    var viewport_size = get_viewport().size

    # Draw assist radius cone
    if current_target:
        var target_screen_pos = camera.unproject_position(current_target.global_position)
        var center = viewport_size / 2

        # Draw line to target
        draw_line(center, target_screen_pos, Color.GREEN, 2.0)

        # Draw assist circle
        draw_arc(center, assist_radius * 10, 0, TAU, 32, Color(0, 1, 0, 0.3), 2.0)

        # Draw adhesion strength
        var adhesion_text = "Adhesion: %.0f%%" % (adhesion_strength * 100)
        draw_string(ThemeDB.fallback_font, Vector2(10, 60),
                    adhesion_text, HORIZONTAL_ALIGNMENT_LEFT, -1, 16, Color.YELLOW)

    # Draw all targets in range
    for target in targets_in_range:
        var screen_pos = camera.unproject_position(target.global_position)
        draw_circle(screen_pos, 5, Color.RED)

    # Display assist multiplier
    draw_string(ThemeDB.fallback_font, Vector2(10, 30),
                "Assist: %.0f%% (%s)" % [assist_multiplier * 100,
                "Controller" if is_using_controller else "Mouse"],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 16,
                Color.GREEN if is_using_controller else Color.GRAY)

# Stats tracking
var assist_stats = {
    "total_shots": 0,
    "assisted_shots": 0,
    "assisted_hits": 0,
    "unassisted_hits": 0,
}

func log_shot(used_assist: bool, hit: bool):
    assist_stats.total_shots += 1
    if used_assist:
        assist_stats.assisted_shots += 1
        if hit:
            assist_stats.assisted_hits += 1
    else:
        if hit:
            assist_stats.unassisted_hits += 1

    if assist_stats.total_shots % 50 == 0:
        print_assist_stats()

func print_assist_stats():
    var assist_usage = 100.0 * assist_stats.assisted_shots / assist_stats.total_shots
    var assist_accuracy = 100.0 * assist_stats.assisted_hits / max(1, assist_stats.assisted_shots)
    var no_assist_accuracy = 100.0 * assist_stats.unassisted_hits / max(1, assist_stats.total_shots - assist_stats.assisted_shots)

    print("=== Aim Assist Stats ===")
    print("  Assist usage: %.1f%%" % assist_usage)
    print("  Assisted accuracy: %.1f%%" % assist_accuracy)
    print("  Unassisted accuracy: %.1f%%" % no_assist_accuracy)
```

**Performance Impact:**
- CPU: 0.05-0.2ms per frame (depends on target count)
- Target detection: O(n) but typically n=5-15 enemies
- Raycasts: ~0.01ms each, 3-5 per frame
- Memory: ~200 bytes per target tracking
- Safe for single-player; in multiplayer, consider server-side validation
