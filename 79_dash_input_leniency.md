### 79. Dash Input Leniency

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players failing directional dash inputs due to imprecise stick angles or timing issues. Without leniency, dashes require exact 8-way or 4-way directional input (±15° tolerance), causing 30-50% of intended dashes to fire in wrong direction or not at all. Double-tap dashes suffer from strict timing windows (100-150ms), failing when player is 20-30ms off. Results in frustration, deaths from mis-dashed into hazards, and perception of "broken dash controls."

**Technical Explanation:**
Dash input leniency provides forgiveness for both directional precision and timing requirements:

1. **Directional Snapping:** Expand angle tolerance for cardinal/diagonal directions from ±15° to ±25-35°. Input at 23° (near-right) snaps to 0° (pure right). Prevents "diagonal when I wanted horizontal" failures.

2. **Dead Zone Adjustment:** Reduce deadzone for dash input detection (0.5-0.7 threshold vs. 0.8+ for movement). Allows lighter stick pressure to register dash.

3. **Double-Tap Timing:** Increase window from 150ms to 200-250ms. Store timestamp of last direction press, accept second press within window. Handles human timing variance.

4. **Input Buffering:** Buffer dash input 100-150ms before becoming valid (during recovery, cooldown). Execute immediately when able.

5. **Priority System:** Dash input overrides movement input when detected. Prevents "wanted to dash right but just ran right" scenarios.

Implementation uses angle quantization (round to nearest 45° for 8-way, 90° for 4-way), timestamp tracking for double-tap, and state machine for cooldown/buffer handling.

**Algorithmic Complexity:**
- Angle calculation: O(1) - atan2 and snap to nearest
- Double-tap detection: O(1) - timestamp comparison
- Buffer check: O(1) - single entry per direction

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Dash configuration
const DASH_SPEED = 600.0
const DASH_DURATION = 0.2  # 200ms / 12 frames
const DASH_COOLDOWN = 0.5  # 500ms between dashes

# Leniency parameters
@export_group("Dash Leniency")
@export var directional_snap_angle: float = 30.0  # Degrees tolerance
@export var dash_input_threshold: float = 0.6  # Lower than movement (0.8)
@export var double_tap_window: float = 0.25  # 250ms for double-tap
@export var dash_buffer_window: float = 0.15  # 150ms buffer

# Dash state
var is_dashing: bool = false
var dash_direction: Vector2 = Vector2.ZERO
var dash_time_remaining: float = 0.0
var dash_cooldown_remaining: float = 0.0

# Double-tap tracking
var last_direction_press: Dictionary = {
    "right": -1.0,
    "left": -1.0,
    "up": -1.0,
    "down": -1.0
}

# Buffer
var buffered_dash_direction: Vector2 = Vector2.ZERO
var buffered_dash_time: float = -1.0

func _physics_process(delta: float) -> void:
    # Update dash timers
    if is_dashing:
        dash_time_remaining -= delta
        if dash_time_remaining <= 0:
            end_dash()
    elif dash_cooldown_remaining > 0:
        dash_cooldown_remaining -= delta

    # Check for dash input
    var dash_input = get_dash_input()

    if dash_input != Vector2.ZERO:
        attempt_dash(dash_input)

    # Apply dash movement
    if is_dashing:
        velocity = dash_direction * DASH_SPEED
    else:
        # Normal movement
        apply_normal_movement(delta)

    move_and_slide()

    # Update buffer
    update_dash_buffer()

func get_dash_input() -> Vector2:
    # Method 1: Button-based dash (simple, always works)
    if Input.is_action_just_pressed("dash"):
        return get_directional_input_with_snap()

    # Method 2: Double-tap detection
    var double_tap_dir = detect_double_tap()
    if double_tap_dir != Vector2.ZERO:
        return double_tap_dir

    return Vector2.ZERO

func get_directional_input_with_snap() -> Vector2:
    # Get raw input
    var input_dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")

    # Check if input magnitude meets threshold
    if input_dir.length() < dash_input_threshold:
        return Vector2.ZERO

    # Calculate angle
    var angle = rad_to_deg(atan2(input_dir.y, input_dir.x))

    # Snap to nearest cardinal/diagonal (8-way)
    var snapped_angle = snap_to_8_way(angle)

    # Convert back to direction vector
    return Vector2(cos(deg_to_rad(snapped_angle)), sin(deg_to_rad(snapped_angle)))

func snap_to_8_way(angle: float) -> float:
    # 8 directions: 0°, 45°, 90°, 135°, 180°, -135°, -90°, -45°
    var directions = [0.0, 45.0, 90.0, 135.0, 180.0, -135.0, -90.0, -45.0]
    var closest_angle = 0.0
    var min_diff = 360.0

    for dir_angle in directions:
        var diff = abs(angle_difference(angle, dir_angle))
        if diff < min_diff:
            min_diff = diff
            closest_angle = dir_angle

    # Only snap if within tolerance
    if min_diff <= directional_snap_angle:
        return closest_angle

    # Outside tolerance, use raw angle
    return angle

func angle_difference(a: float, b: float) -> float:
    # Calculate shortest angle between two angles
    var diff = fmod(b - a + 180.0, 360.0) - 180.0
    return diff if diff >= -180.0 else diff + 360.0

func detect_double_tap() -> Vector2:
    var current_time = Time.get_ticks_msec() / 1000.0

    # Check each direction
    var directions = {
        "right": Vector2.RIGHT,
        "left": Vector2.LEFT,
        "up": Vector2.UP,
        "down": Vector2.DOWN
    }

    for dir_name in directions:
        var action_name = "move_" + dir_name

        if Input.is_action_just_pressed(action_name):
            # Check if this is a double-tap
            var last_press_time = last_direction_press[dir_name]

            if last_press_time > 0:
                var time_since_last = current_time - last_press_time

                if time_since_last <= double_tap_window:
                    # Double-tap detected!
                    last_direction_press[dir_name] = -1.0  # Reset
                    return directions[dir_name]

            # Update last press time
            last_direction_press[dir_name] = current_time

    return Vector2.ZERO

func attempt_dash(direction: Vector2) -> void:
    # Can't dash if already dashing or on cooldown
    if is_dashing or dash_cooldown_remaining > 0:
        # Buffer the dash input
        buffer_dash(direction)
        return

    # Execute dash
    execute_dash(direction)

func buffer_dash(direction: Vector2) -> void:
    buffered_dash_direction = direction
    buffered_dash_time = Time.get_ticks_msec() / 1000.0
    print("Dash buffered for direction: %v" % direction)

func update_dash_buffer() -> void:
    if buffered_dash_direction == Vector2.ZERO:
        return

    # Check if buffer expired
    var current_time = Time.get_ticks_msec() / 1000.0
    var buffer_age = current_time - buffered_dash_time

    if buffer_age > dash_buffer_window:
        buffered_dash_direction = Vector2.ZERO
        return

    # Check if we can now execute buffered dash
    if not is_dashing and dash_cooldown_remaining <= 0:
        execute_dash(buffered_dash_direction)
        buffered_dash_direction = Vector2.ZERO

func execute_dash(direction: Vector2) -> void:
    is_dashing = true
    dash_direction = direction.normalized()
    dash_time_remaining = DASH_DURATION

    # Visual/audio feedback
    $AnimationPlayer.play("dash")
    $DashParticles.restart()
    $DashSound.play()

    # Optional: Invincibility frames
    set_collision_layer_value(1, false)  # Disable player collision

    print("Dash executed: %v" % direction)

func end_dash() -> void:
    is_dashing = false
    dash_cooldown_remaining = DASH_COOLDOWN

    # Restore collision
    set_collision_layer_value(1, true)

    # Optional: Momentum preservation
    velocity = dash_direction * (DASH_SPEED * 0.3)  # 30% of dash speed carried

func apply_normal_movement(delta: float):
    # Regular movement logic
    var input = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    var speed = 200.0
    velocity = input * speed
```

**Advanced Contextual Dash System:**
```gdscript
class_name ContextualDashSystem

# Different dash behaviors based on context
enum DashType { GROUND, AIR, WALL, COMBAT }

var current_dash_type: DashType = DashType.GROUND
var dash_charges: int = 1  # Air dashes separate from ground
var air_dash_charges_max: int = 1

# Advanced snapping
var snap_mode: String = "8way"  # "4way", "8way", "free"
var enable_auto_aim_dash: bool = true  # Dash toward nearest enemy
var auto_aim_range: float = 300.0

func get_contextual_dash_direction(raw_input: Vector2) -> Vector2:
    var snapped = snap_direction(raw_input)

    # Auto-aim toward enemies if enabled and no clear direction
    if enable_auto_aim_dash and raw_input.length() < 0.4:
        var nearest_enemy = find_nearest_enemy_in_range(auto_aim_range)
        if nearest_enemy:
            return (nearest_enemy.global_position - global_position).normalized()

    return snapped

func snap_direction(input: Vector2) -> Vector2:
    if input.length() < dash_input_threshold:
        return Vector2.ZERO

    match snap_mode:
        "4way":
            return snap_to_4_way(input)
        "8way":
            return snap_to_8_way_vector(input)
        "free":
            return input.normalized()

    return input.normalized()

func snap_to_4_way(input: Vector2) -> Vector2:
    # Snap to cardinal directions only
    var angle = rad_to_deg(atan2(input.y, input.x))

    # Determine closest cardinal
    if abs(angle) < 45.0:
        return Vector2.RIGHT
    elif abs(angle) > 135.0:
        return Vector2.LEFT
    elif angle > 0:
        return Vector2.DOWN
    else:
        return Vector2.UP

func snap_to_8_way_vector(input: Vector2) -> Vector2:
    var angle = rad_to_deg(atan2(input.y, input.x))
    var snapped = snap_to_8_way(angle)
    return Vector2(cos(deg_to_rad(snapped)), sin(deg_to_rad(snapped)))

# Adaptive leniency based on player skill
var player_dash_accuracy: float = 0.8  # 80% correct directions
var adaptive_snap_angle: float = 30.0

func update_adaptive_leniency(was_correct_direction: bool):
    # Track accuracy over time
    var weight = 0.1  # Exponential moving average
    if was_correct_direction:
        player_dash_accuracy = lerp(player_dash_accuracy, 1.0, weight)
    else:
        player_dash_accuracy = lerp(player_dash_accuracy, 0.0, weight)

    # Reduce assistance for skilled players
    if player_dash_accuracy > 0.9:
        adaptive_snap_angle = 20.0  # Tighter for pros
    elif player_dash_accuracy < 0.6:
        adaptive_snap_angle = 40.0  # More forgiving for struggling players
    else:
        adaptive_snap_angle = 30.0  # Default

func find_nearest_enemy_in_range(range: float) -> Node2D:
    var enemies = get_tree().get_nodes_in_group("enemies")
    var nearest: Node2D = null
    var nearest_dist = range

    for enemy in enemies:
        var dist = global_position.distance_to(enemy.global_position)
        if dist < nearest_dist:
            nearest = enemy
            nearest_dist = dist

    return nearest
```

**Key Parameters:**
- **Directional snap angle:** 25-40° (typical: 30°)
  - Casual games: 35-45°
  - Skill-based games: 20-30°
- **Input threshold:** 0.5-0.7 (vs 0.8 for movement)
- **Double-tap window:** 200-300ms (typical: 250ms)
- **Buffer window:** 100-150ms
- **Snap modes:**
  - 4-way: 90° snapping (simple platformers)
  - 8-way: 45° snapping (most action games)
  - Free: No snapping (precise control games)

**Edge Cases:**
- **Diagonal vs cardinal conflict:** Prefer cardinal if within tighter tolerance (±20°)
- **Wall collision:** Cancel dash or slide along wall
- **Dash into hazard:** May want warning or prevention
- **Aerial dash:** Separate charges from ground dash
- **Damage during dash:** Cancel or grant invincibility
- **Dash off ledge:** Preserve horizontal momentum
- **Multiple rapid dashes:** Cool down prevents spam
- **Controller drift:** High deadzone for dash detection
- **Input device switch:** Reset double-tap timers

**When NOT to Use:**
- Games requiring precise directional control (shmups, twin-stick shooters)
- When dash direction ambiguity creates unfair situations
- Competitive fighting games (exact input is skill test)
- When free-form dash angle is core mechanic
- VR games with 1:1 motion tracking

**Examples from Shipped Games:**

1. **Celeste (2018):** 8-way dash with 35° snap tolerance. Buffer window: 0.15s. Dash input overrides movement completely during dash. Can dash immediately upon landing (no cooldown), but only 1 air dash that resets on ground. Directional snap crucial for diagonal dashing in tight spaces.

2. **Dead Cells (2018):** Dash snaps to 4-way (cardinals only). 40° tolerance for up/down, 30° for left/right. Dash has 0.3s cooldown, 0.10s buffer. Auto-aims toward enemies if no direction input. Prioritizes forward dash (direction player is facing) on ambiguous input.

3. **Hollow Knight (2017):** Free-angle dash (no snapping), but compensates with long dash distance. 0.5s cooldown, 0.12s buffer. Intentionally precise control—snapping would make shade cloak (invincible dash) too easy. Skill-based design choice.

4. **Hades (2020):** 8-way dash with auto-aim when near enemies. 30° snap angle for cardinal, 25° for diagonal. 0.15s buffer during attack animations. Dash has invincibility frames, making direction crucial for dodging. Boon system can modify dash properties.

5. **Katana ZERO (2019):** Dash snaps to 8-way, but also auto-targets if enemy within 20° and range. 250ms double-tap window. Dash is both movement and attack, so direction forgiveness is generous (35-40°). Slow-motion during planning phase assists with precision.

**Platform Considerations:**
- **PC (keyboard):** 8-way natural (WASD), generous snap (35-40°)
- **PC (mouse aim):** Dash toward cursor (no snap needed)
- **Console (analog stick):** 30-35° snap essential for controller imprecision
- **Mobile (touch):** Swipe gesture for dash, very generous snap (40-45°)
- **VR:** Physical motion, minimal snap (10-15° max)

**Godot-Specific Notes:**
- Use `Input.get_vector()` for raw directional input
- `atan2(y, x)` for angle calculation
- `fmod()` for angle wrapping (-180 to 180)
- `Input.is_action_just_pressed()` for double-tap detection
- Export snap angle as `@export_range(10.0, 50.0)` for designer tuning
- Consider `AnimationTree` for directional dash animations
- Profile: Angle calculation should be <0.01ms per frame

**Synergies:**
- **Input Buffering (#72):** Dash buffer is specific application
- **Roll/Dodge Cancel (#81):** Dash may cancel other animations
- **Aim Assist (#76):** Auto-aim dash toward enemies
- **Camera Lead (#82):** Camera anticipates dash direction
- **Invincibility frames:** Dash often grants brief invulnerability
- **Momentum preservation:** Dash can carry into jump/fall

**Measurement/Profiling:**
- **Direction accuracy:**
  - Track intended vs. actual dash direction
  - Without snap: 60-70% accuracy
  - With snap: 85-95% accuracy
- **Snap activation rate:**
  - % of dashes that used snapping
  - Typical: 40-60% of dashes snapped
- **Failure rate:**
  - Dashes that fired wrong direction: <5% target
  - Missed dashes (no register): <3% target
- **Buffer usage:**
  - % of dashes that used buffer
  - Typical: 10-20% of dashes
- **Player satisfaction:**
  - "Does dash go where you expect?" (target: >90% yes)

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    # Draw directional snap zones
    var center = Vector2.ZERO
    var radius = 60.0

    # Draw 8 snap directions
    var snap_angles = [0.0, 45.0, 90.0, 135.0, 180.0, -135.0, -90.0, -45.0]

    for angle in snap_angles:
        var dir = Vector2(cos(deg_to_rad(angle)), sin(deg_to_rad(angle)))
        var end_pos = dir * radius

        # Draw snap zone (tolerance wedge)
        var tolerance_rad = deg_to_rad(directional_snap_angle)
        draw_arc(center, radius * 0.8, deg_to_rad(angle) - tolerance_rad,
                 deg_to_rad(angle) + tolerance_rad, 16,
                 Color(0, 1, 0, 0.2), 2.0)

        # Draw snap direction line
        draw_line(center, end_pos, Color.GREEN, 2.0)
        draw_circle(end_pos, 4, Color.GREEN)

    # Draw current input
    var input = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    if input.length() > 0.1:
        var input_pos = input.normalized() * radius
        draw_line(center, input_pos, Color.YELLOW, 3.0)
        draw_circle(input_pos, 5, Color.YELLOW)

        # Show snapped direction
        var snapped = get_directional_input_with_snap()
        if snapped != Vector2.ZERO:
            var snapped_pos = snapped * radius
            draw_line(center, snapped_pos, Color.CYAN, 4.0)
            draw_circle(snapped_pos, 6, Color.CYAN)

    # Dash cooldown indicator
    if dash_cooldown_remaining > 0:
        var cooldown_pct = dash_cooldown_remaining / DASH_COOLDOWN
        draw_arc(Vector2(0, -40), 10, 0, TAU * (1.0 - cooldown_pct),
                 16, Color.RED, 3.0)

    # Buffer indicator
    if buffered_dash_direction != Vector2.ZERO:
        var buffer_age = Time.get_ticks_msec() / 1000.0 - buffered_dash_time
        var buffer_remaining = dash_buffer_window - buffer_age
        var buffer_pct = buffer_remaining / dash_buffer_window

        draw_circle(Vector2(0, -50), 8, Color(1, 1, 0, buffer_pct))
        draw_string(ThemeDB.fallback_font, Vector2(-30, -60),
                    "Buffered: %.2fs" % buffer_remaining,
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 10, Color.YELLOW)

# Statistics
var dash_stats = {
    "total_dashes": 0,
    "snapped_dashes": 0,
    "buffered_dashes": 0,
    "direction_errors": 0,
}

func log_dash(used_snap: bool, used_buffer: bool):
    dash_stats.total_dashes += 1
    if used_snap:
        dash_stats.snapped_dashes += 1
    if used_buffer:
        dash_stats.buffered_dashes += 1

    if dash_stats.total_dashes % 25 == 0:
        print_dash_stats()

func print_dash_stats():
    var snap_rate = 100.0 * dash_stats.snapped_dashes / dash_stats.total_dashes
    var buffer_rate = 100.0 * dash_stats.buffered_dashes / dash_stats.total_dashes

    print("=== Dash Statistics ===")
    print("  Total dashes: %d" % dash_stats.total_dashes)
    print("  Snap usage: %.1f%%" % snap_rate)
    print("  Buffer usage: %.1f%%" % buffer_rate)
```

**Performance Impact:**
- CPU: 0.01-0.03ms per frame (angle calculations)
- Memory: ~100 bytes per entity (state tracking)
- Trigonometry: 1-2 atan2 calls per dash input
- No allocation overhead
- Safe for 50+ entities with dash capability
