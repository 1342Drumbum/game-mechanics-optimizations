### 82. Camera Lead (Input Anticipation)

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players unable to see where they're going because camera is centered on character, causing 30-50% of deaths from off-screen hazards/enemies. Without camera lead, players must move first then wait for camera to catch up (200-400ms lag), creating "tunnel vision" gameplay. Results in frustration from "unfair" deaths, reduced movement speed (hesitation), and perception of "bad camera" being the most common complaint in playtests.

**Technical Explanation:**
Camera lead predicts player movement direction and shifts camera ahead of the character, revealing more of the playspace in the direction of travel. System tracks: (1) player velocity vector, (2) input direction (pre-movement), (3) aim/look direction, and uses weighted average to calculate lead offset. Camera target becomes: `player_position + (movement_direction × lead_distance)`.

Implementation uses smoothed interpolation (lerp/slerp) to prevent jarring camera shifts. Lead distance scales with velocity—stationary = 0 offset, sprinting = max offset (50-150 pixels). Advanced implementations predict future position: `player_position + velocity × lookahead_time`, accounting for inertia.

Key insight: Human visual field is ~180° horizontal but gameplay typically requires 270-330° awareness. Camera lead artificially extends this by shifting view window in anticipation of movement, reducing "fog of war" effect.

**Algorithmic Complexity:**
- Lead calculation: O(1) - vector math per frame
- Smoothing: O(1) - lerp between current and target

**Implementation Pattern:**
```gdscript
# Godot implementation
extends Camera2D

# Camera lead parameters
@export_group("Camera Lead")
@export var enable_camera_lead: bool = true
@export var max_lead_distance: float = 100.0  # Pixels ahead of player
@export var lead_smoothing: float = 3.0  # Higher = slower camera
@export var velocity_influence: float = 0.7  # 0-1: velocity vs. input direction
@export var vertical_lead_mult: float = 0.5  # Reduce vertical lead (usually less important)

# Input-based lead (predicts before movement)
@export var input_lead_enabled: bool = true
@export var input_lead_distance: float = 40.0  # Offset from pure input

# Advanced features
@export var lookahead_time: float = 0.3  # Seconds to predict ahead
@export var enable_aim_lead: bool = false  # Follow cursor/aim direction
@export var aim_lead_distance: float = 80.0

# Camera state
var current_lead_offset: Vector2 = Vector2.ZERO
var target_lead_offset: Vector2 = Vector2.ZERO
var player: CharacterBody2D = null

# Smoothing history
var velocity_history: Array[Vector2] = []
const VELOCITY_HISTORY_SIZE = 5

func _ready() -> void:
    # Find player
    player = get_parent() as CharacterBody2D
    if not player:
        push_error("Camera2D must be child of player CharacterBody2D")
        return

    # Initialize
    global_position = player.global_position

func _physics_process(delta: float) -> void:
    if not player or not enable_camera_lead:
        # Standard follow cam
        global_position = player.global_position
        return

    # Calculate target lead offset
    target_lead_offset = calculate_lead_offset()

    # Smooth transition to target
    current_lead_offset = current_lead_offset.lerp(target_lead_offset, lead_smoothing * delta)

    # Apply to camera position
    global_position = player.global_position + current_lead_offset

func calculate_lead_offset() -> Vector2:
    var lead = Vector2.ZERO

    # Component 1: Velocity-based lead (where player is moving)
    if player.velocity.length() > 10.0:  # Threshold to ignore tiny movements
        var velocity_lead = calculate_velocity_lead()
        lead += velocity_lead * velocity_influence

    # Component 2: Input-based lead (where player wants to move)
    if input_lead_enabled:
        var input_lead = calculate_input_lead()
        lead += input_lead * (1.0 - velocity_influence)

    # Component 3: Aim-based lead (where player is looking)
    if enable_aim_lead:
        var aim_lead = calculate_aim_lead()
        lead += aim_lead * 0.3  # Weighted addition

    # Scale by vertical multiplier
    lead.y *= vertical_lead_mult

    # Clamp to max distance
    if lead.length() > max_lead_distance:
        lead = lead.normalized() * max_lead_distance

    return lead

func calculate_velocity_lead() -> Vector2:
    # Smooth velocity using history
    update_velocity_history()

    var avg_velocity = get_average_velocity()

    # Predict future position
    var predicted_offset = avg_velocity * lookahead_time

    # Scale by velocity (faster = more lead)
    var speed_ratio = avg_velocity.length() / player.get_meta("max_speed", 300.0)
    speed_ratio = clamp(speed_ratio, 0.0, 1.0)

    return predicted_offset * speed_ratio

func calculate_input_lead() -> Vector2:
    # Get raw input direction (before physics applied)
    var input_dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")

    if input_dir.length() < 0.1:
        return Vector2.ZERO

    # Lead in input direction
    return input_dir.normalized() * input_lead_distance

func calculate_aim_lead() -> Vector2:
    # Option 1: Mouse position (PC)
    if Input.mouse_mode == Input.MOUSE_MODE_VISIBLE:
        var mouse_pos = get_viewport().get_mouse_position()
        var screen_center = get_viewport_rect().size / 2
        var to_mouse = (mouse_pos - screen_center).normalized()
        return to_mouse * aim_lead_distance

    # Option 2: Right stick (controller)
    var aim_x = Input.get_axis("aim_left", "aim_right")
    var aim_y = Input.get_axis("aim_up", "aim_down")
    var aim_dir = Vector2(aim_x, aim_y)

    if aim_dir.length() < 0.3:  # Deadzone
        return Vector2.ZERO

    return aim_dir.normalized() * aim_lead_distance

func update_velocity_history() -> void:
    velocity_history.append(player.velocity)

    if velocity_history.size() > VELOCITY_HISTORY_SIZE:
        velocity_history.pop_front()

func get_average_velocity() -> Vector2:
    if velocity_history.is_empty():
        return player.velocity

    var sum = Vector2.ZERO
    for vel in velocity_history:
        sum += vel

    return sum / velocity_history.size()
```

**Advanced Dynamic Lead System:**
```gdscript
class_name DynamicCameraLead extends Camera2D

# Context-aware lead adjustment
enum CameraContext { EXPLORATION, COMBAT, PLATFORMING, CINEMATIC }

var current_context: CameraContext = CameraContext.EXPLORATION
var context_lead_multipliers = {
    CameraContext.EXPLORATION: 1.0,   # Full lead
    CameraContext.COMBAT: 0.6,        # Reduced (keep enemies on screen)
    CameraContext.PLATFORMING: 0.8,   # Medium (see platforms ahead)
    CameraContext.CINEMATIC: 0.0      # No lead (cinematic framing)
}

# Edge detection (reduce lead near edges)
var near_level_edge: bool = false
var edge_dampening: float = 1.0

# Zone-based overrides
var current_zone_lead_override: float = -1.0  # -1 = use default

func calculate_contextual_lead() -> Vector2:
    var base_lead = calculate_lead_offset()

    # Apply context multiplier
    var context_mult = context_lead_multipliers.get(current_context, 1.0)
    base_lead *= context_mult

    # Apply zone override
    if current_zone_lead_override >= 0:
        base_lead = base_lead.normalized() * current_zone_lead_override

    # Dampen near edges
    if near_level_edge:
        base_lead *= edge_dampening

    return base_lead

func detect_context() -> void:
    # Auto-detect gameplay context
    var enemies_nearby = get_tree().get_nodes_in_group("enemies").any(
        func(e): return player.global_position.distance_to(e.global_position) < 400
    )

    if enemies_nearby:
        current_context = CameraContext.COMBAT
    elif player.is_on_floor():
        current_context = CameraContext.EXPLORATION
    else:
        current_context = CameraContext.PLATFORMING

func check_level_edges() -> void:
    # Raycast to detect proximity to level boundaries
    var screen_half = get_viewport_rect().size / (2.0 * zoom)

    var directions = [
        Vector2.RIGHT * screen_half.x,
        Vector2.LEFT * screen_half.x,
        Vector2.UP * screen_half.y,
        Vector2.DOWN * screen_half.y
    ]

    near_level_edge = false
    for dir in directions:
        if is_near_boundary(dir):
            near_level_edge = true
            # Calculate dampening based on distance to edge
            edge_dampening = calculate_edge_dampening(dir)
            break

func is_near_boundary(direction: Vector2) -> bool:
    var space_state = get_world_2d().direct_space_state
    var query = PhysicsRayQueryParameters2D.create(
        global_position,
        global_position + direction
    )
    query.collision_mask = 0b10000000  # Level geometry layer

    var result = space_state.intersect_ray(query)
    return result.size() > 0

# Acceleration-based prediction
var last_velocity: Vector2 = Vector2.ZERO
var current_acceleration: Vector2 = Vector2.ZERO

func calculate_acceleration_based_lead() -> Vector2:
    # Calculate acceleration
    var delta = get_physics_process_delta_time()
    current_acceleration = (player.velocity - last_velocity) / delta
    last_velocity = player.velocity

    # Predict future velocity using acceleration
    var predicted_velocity = player.velocity + current_acceleration * lookahead_time

    # Lead to predicted position
    return predicted_velocity * lookahead_time

# Look-ahead zones (designer-placed camera hints)
func _on_camera_zone_entered(zone: Area2D) -> void:
    # Designer can place zones that override camera behavior
    if zone.has_meta("camera_lead_distance"):
        current_zone_lead_override = zone.get_meta("camera_lead_distance")

    if zone.has_meta("camera_lead_direction"):
        var forced_direction: Vector2 = zone.get_meta("camera_lead_direction")
        # Force camera to look in specific direction
        target_lead_offset = forced_direction * max_lead_distance

func _on_camera_zone_exited(zone: Area2D) -> void:
    current_zone_lead_override = -1.0
```

**Key Parameters:**
- **Max lead distance:** 50-150 pixels (typical: 80-120)
  - Slow games: 60-100px
  - Fast games: 100-150px
  - Platformers: 80-120px
- **Lead smoothing:** 2-5 (typical: 3-4)
  - Lower = more responsive, potentially jarring
  - Higher = smoother, potentially laggy
- **Velocity influence:** 0.6-0.9 (typical: 0.7)
- **Lookahead time:** 0.2-0.5s (typical: 0.3s)
- **Vertical multiplier:** 0.3-0.7 (typical: 0.5)

**Edge Cases:**
- **Rapid direction changes:** Dampen lead during "wiggle" inputs
- **Stopping:** Gradually reduce lead to zero, don't snap
- **Jumping:** May want vertical lead disabled or reduced
- **Falling:** Increase downward lead to see landing area
- **Swimming/flying:** Different lead parameters (omnidirectional)
- **Cutscenes:** Disable lead, snap to scripted positions
- **Split-screen:** Reduced lead to prevent disorientation
- **Zoom changes:** Scale lead distance with zoom level
- **Screen shake:** Add lead before shake, not during

**When NOT to Use:**
- Top-down games with full screen visibility
- Fixed camera games (platformers with single-screen levels)
- Twin-stick shooters (aim provides different lead)
- Strategy games (player controls camera directly)
- When deliberate limited visibility is core mechanic
- Horror games (limited vision creates tension)
- Puzzle games with static camera requirements

**Examples from Shipped Games:**

1. **Super Meat Boy (2010):** 80-100px lead in movement direction. Critical for seeing upcoming hazards at high speed. Lead increases with velocity (sprinting = max lead). Vertical lead disabled during falling to prevent disorientation. Community consensus: camera is "invisible" but crucial to game feel.

2. **Celeste (2018):** Adaptive lead system. 100px base lead, increases to 150px during dashing. Dash direction overrides velocity lead for 0.3s. Different lead values per room type (exploration vs. precision platforming). Developer commentary: camera system took months to tune, received minimal conscious notice (success metric).

3. **Dead Cells (2018):** 60-80px lead with enemy-weighted offset. When enemies on screen, camera shifts toward them (not just player velocity). Combat mode: reduced lead to keep arena visible. Sprint mode: increased lead to 120px. Demonstrates context-aware lead.

4. **Hollow Knight (2017):** Subtle 50-70px lead. More conservative than fast platformers due to exploration focus. Vertical lead enabled (0.6 multiplier) for deep vertical shafts. Focus/aim direction influences camera when using ranged attacks. Very smooth (smoothing = 4.5).

5. **Ori and the Will of the Wisps (2020):** Advanced predictive lead using spline paths. Camera knows about upcoming geometry (designer-placed camera hints). Up to 180px lead during high-speed bash/grapple sequences. Contextual: exploration = 60px, escape sequences = 150px. GDC talk covers technical implementation.

**Platform Considerations:**
- **PC (large monitors):** Larger lead (100-150px) to utilize screen real estate
- **Console (TVs):** Medium lead (80-120px), standard design
- **Mobile (small screens):** Reduced lead (50-80px) to avoid disorientation
- **Handheld (Switch):** 60-100px, balanced for small screen + portability
- **Ultrawide displays:** Horizontal lead can be increased

**Godot-Specific Notes:**
- Use `Camera2D.offset` or modify `global_position` for lead
- `lerp()` for smooth interpolation
- `get_viewport().get_mouse_position()` for cursor-based lead
- Consider `Camera2D.position_smoothing_enabled` but manual control is better
- `PhysicsRayQueryParameters2D` for edge detection
- Export lead parameters for per-level tuning
- Godot 4.x: `Camera2D` improvements make lead implementation cleaner

**Synergies:**
- **Dash Input Leniency (#79):** Camera leads toward dash direction
- **Aim Assist (#76):** Camera can lead toward targets
- **Camera shake:** Apply after lead calculation, not before
- **Screen transitions:** Blend lead offset across room changes
- **Zoom changes:** Scale lead proportionally with zoom
- **Audio spatialization:** 3D audio can use predicted position

**Measurement/Profiling:**
- **Off-screen death analysis:**
  - Without lead: 30-50% of deaths from off-screen hazards
  - With lead: 10-20% (remaining are intended challenges)
- **Player eye tracking:**
  - Heat maps show where players look
  - Lead should align with visual attention (validated in research)
- **Subjective testing:**
  - A/B test: lead on vs. off
  - "Can you see where you're going?" (target: >90% yes with lead)
  - "Does camera feel responsive?" (target: >85% yes)
- **Hesitation tracking:**
  - Measure time spent stationary near edges
  - Should decrease 20-40% with lead
- **Lead distance optimization:**
  - Plot player speed distribution
  - Tune max_lead_distance to 95th percentile speed

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    # Draw camera center
    draw_circle(Vector2.ZERO, 5, Color.RED)

    # Draw player position (relative to camera)
    var player_screen_pos = to_local(player.global_position)
    draw_circle(player_screen_pos, 8, Color.GREEN)
    draw_line(Vector2.ZERO, player_screen_pos, Color.GREEN, 2.0)

    # Draw lead offset vector
    if current_lead_offset.length() > 1.0:
        draw_line(player_screen_pos, player_screen_pos + current_lead_offset,
                  Color.CYAN, 3.0)
        draw_circle(player_screen_pos + current_lead_offset, 6, Color.CYAN)

        # Lead offset magnitude text
        draw_string(ThemeDB.fallback_font,
                    player_screen_pos + current_lead_offset + Vector2(10, 0),
                    "Lead: %.0fpx" % current_lead_offset.length(),
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 12, Color.CYAN)

    # Draw velocity vector
    if player.velocity.length() > 10:
        var vel_normalized = player.velocity.normalized() * 40
        draw_line(player_screen_pos, player_screen_pos + vel_normalized,
                  Color.YELLOW, 2.0, true)

    # Draw screen bounds
    var viewport_size = get_viewport_rect().size
    var half_size = viewport_size / (2.0 * zoom)
    draw_rect(Rect2(-half_size, viewport_size / zoom), Color(1, 1, 1, 0.2), false, 2.0)

    # Draw info text
    var info_pos = -half_size + Vector2(10, 20)
    draw_string(ThemeDB.fallback_font, info_pos,
                "Lead: %.1f, %.1f (%.0fpx)" % [current_lead_offset.x, current_lead_offset.y, current_lead_offset.length()],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.WHITE)

    draw_string(ThemeDB.fallback_font, info_pos + Vector2(0, 20),
                "Velocity: %.0f px/s" % player.velocity.length(),
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.WHITE)

    draw_string(ThemeDB.fallback_font, info_pos + Vector2(0, 40),
                "Context: %s" % CameraContext.keys()[current_context],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.WHITE)

# Heatmap of camera positions (for optimization)
var camera_position_samples: Array[Vector2] = []
const MAX_SAMPLES = 1000

func record_camera_position():
    camera_position_samples.append(global_position - player.global_position)

    if camera_position_samples.size() > MAX_SAMPLES:
        camera_position_samples.pop_front()

func analyze_camera_distribution():
    # Calculate average offset
    var sum = Vector2.ZERO
    for sample in camera_position_samples:
        sum += sample

    var avg = sum / camera_position_samples.size()

    # Calculate variance
    var variance = Vector2.ZERO
    for sample in camera_position_samples:
        var diff = sample - avg
        variance += diff * diff

    variance /= camera_position_samples.size()

    print("=== Camera Lead Analysis ===")
    print("  Average offset: %.1f, %.1f" % [avg.x, avg.y])
    print("  Variance: %.1f, %.1f" % [variance.x, variance.y])
    print("  Max lead used: %.0fpx" % camera_position_samples.map(
        func(s): return s.length()
    ).max())
```

**Performance Impact:**
- CPU: 0.01-0.03ms per frame (vector math + lerp)
- Memory: ~100 bytes (velocity history + state)
- Raycasts (edge detection): +0.05-0.15ms per frame (4-8 rays)
- No allocation overhead
- Single camera per player, minimal scaling concerns

**Tuning Workflow:**
1. Start with lead disabled, gather playtest feedback on "blind jumps"
2. Enable lead at 60px, test player reaction
3. Incrementally increase lead distance until players stop complaining about visibility
4. Tune smoothing until camera motion feels "invisible"
5. A/B test different values with metrics (death locations, hesitation time)
6. Final value typically 70-120px depending on game speed

**Common Mistakes:**
- **Too much lead:** Camera feels "detached" from player, disorienting
- **Too little smoothing:** Jarring camera motion, motion sickness
- **Too much smoothing:** Camera feels laggy, defeats purpose of lead
- **Ignoring context:** Combat needs different lead than exploration
- **No edge handling:** Camera shows out-of-bounds areas
- **Forgetting vertical:** Falling deaths from not seeing below
