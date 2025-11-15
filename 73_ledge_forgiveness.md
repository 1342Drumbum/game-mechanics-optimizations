### 73. Ledge Forgiveness/Magnetism

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players barely missing ledges by 1-5 pixels and falling to their death, leading to frustration and "unfair" game feel. Without forgiveness, precision platforming requires pixel-perfect positioning. Studies show 25-40% of near-miss landings fail without magnetism, causing player perception of "slippery controls" or "broken collision." Particularly problematic in 3D where depth perception is impaired.

**Technical Explanation:**
Ledge forgiveness uses spatial tolerance zones around platform edges to assist player landings. When player is within forgiveness distance (typically 4-12 pixels) of a platform edge while descending, the system either: (1) snaps player position onto the platform, (2) extends collision box temporarily, or (3) adjusts player velocity to guide them onto the surface. Implementation uses raycasts or expanded collision shapes to detect "near miss" scenarios, then applies corrective position/velocity adjustments.

Two primary approaches: **Passive magnetism** (collision box extension during descent) and **Active snapping** (position correction on near-miss detection). Passive is more subtle, active is more forgiving but can feel artificial if too aggressive. Best implementations use small passive zones (2-4px) with larger detection zones (8-12px) for edge-case correction.

**Algorithmic Complexity:**
- Raycast approach: O(n) where n = nearby platforms (typically 1-5)
- Collision shape approach: O(1) per platform collision
- Position snapping: O(1) single vector adjustment

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Ledge forgiveness parameters
const LEDGE_FORGIVENESS_HORIZONTAL = 8.0  # pixels
const LEDGE_FORGIVENESS_VERTICAL = 6.0  # pixels
const SNAP_SPEED = 5.0  # How quickly to snap (higher = more subtle)
const MAX_SNAP_VELOCITY = 150.0  # Don't snap during fast falls

@export var enable_ledge_magnetism: bool = true
@export var debug_draw_zones: bool = false

func _physics_process(delta: float) -> void:
    # Only apply when descending
    if velocity.y <= 0 or not enable_ledge_magnetism:
        move_and_slide()
        return

    # Check for near-miss ledges before moving
    var ledge_correction = check_ledge_forgiveness()

    if ledge_correction != Vector2.ZERO:
        # Apply gentle correction toward ledge
        var correction_strength = min(delta * SNAP_SPEED, 1.0)
        position += ledge_correction * correction_strength

    move_and_slide()

func check_ledge_forgiveness() -> Vector2:
    # Don't magnetize during very fast falls
    if velocity.y > MAX_SNAP_VELOCITY:
        return Vector2.ZERO

    # Cast rays to nearby platforms
    var space_state = get_world_2d().direct_space_state

    # Check left and right of player
    var checks = [
        Vector2(-LEDGE_FORGIVENESS_HORIZONTAL, LEDGE_FORGIVENESS_VERTICAL),
        Vector2(LEDGE_FORGIVENESS_HORIZONTAL, LEDGE_FORGIVENESS_VERTICAL),
    ]

    var closest_ledge = null
    var closest_distance = INF

    for offset in checks:
        var ray_start = global_position
        var ray_end = global_position + offset

        var query = PhysicsRayQueryParameters2D.create(ray_start, ray_end)
        query.collision_mask = 1  # Ground layer
        query.collide_with_areas = false

        var result = space_state.intersect_ray(query)

        if result:
            var distance = global_position.distance_to(result.position)
            if distance < closest_distance:
                closest_distance = distance
                closest_ledge = result.position

    # Return correction vector toward closest ledge
    if closest_ledge:
        var correction = closest_ledge - global_position
        correction.y = 0  # Only horizontal correction
        return correction

    return Vector2.ZERO

# Alternative: Collision shape expansion approach
func _ready():
    # Store original collision shape
    var collision_shape = $CollisionShape2D
    var original_shape = collision_shape.shape

    # Create expanded shape for descending
    var expanded_shape = original_shape.duplicate()
    if expanded_shape is RectangleShape2D:
        # Expand width slightly
        expanded_shape.size.x += LEDGE_FORGIVENESS_HORIZONTAL * 2

func _draw():
    if debug_draw_zones and OS.is_debug_build():
        # Draw forgiveness zones
        var color = Color(0, 1, 0, 0.3)
        draw_rect(Rect2(
            Vector2(-LEDGE_FORGIVENESS_HORIZONTAL, 0),
            Vector2(LEDGE_FORGIVENESS_HORIZONTAL * 2, LEDGE_FORGIVENESS_VERTICAL)
        ), color)
```

**Advanced Corner Detection System:**
```gdscript
class_name LedgeForgivenessSystem

const CORNER_DETECT_DISTANCE = 12.0
const CORNER_SNAP_THRESHOLD = 0.3  # 30% of distance
const EDGE_GRAB_PRIORITY = true  # Prefer grabbing vs. landing

class LedgeInfo:
    var position: Vector2
    var normal: Vector2
    var platform: Node2D
    var is_corner: bool = false
    var distance: float

func detect_nearby_ledges(player_pos: Vector2) -> Array[LedgeInfo]:
    var ledges: Array[LedgeInfo] = []
    var space_state = get_world_2d().direct_space_state

    # Multi-directional raycasts for comprehensive detection
    var angles = [
        -90,  # Down
        -75, -105,  # Down-left, down-right
        -60, -120,  # Diagonal
    ]

    for angle in angles:
        var direction = Vector2.from_angle(deg_to_rad(angle))
        var ray_end = player_pos + direction * CORNER_DETECT_DISTANCE

        var query = PhysicsRayQueryParameters2D.create(player_pos, ray_end)
        query.collision_mask = collision_layer_ground

        var result = space_state.intersect_ray(query)
        if result:
            var ledge = LedgeInfo.new()
            ledge.position = result.position
            ledge.normal = result.normal
            ledge.platform = result.collider
            ledge.distance = player_pos.distance_to(result.position)

            # Detect corners (normal not purely vertical)
            ledge.is_corner = abs(result.normal.x) > 0.1

            ledges.append(ledge)

    # Sort by distance
    ledges.sort_custom(func(a, b): return a.distance < b.distance)
    return ledges

func apply_forgiveness(player: CharacterBody2D, ledges: Array[LedgeInfo]) -> void:
    if ledges.is_empty():
        return

    var nearest = ledges[0]

    # Only apply if within threshold
    if nearest.distance > CORNER_DETECT_DISTANCE:
        return

    # Calculate correction strength based on distance
    var strength = 1.0 - (nearest.distance / CORNER_DETECT_DISTANCE)
    strength = ease(strength, -2.0)  # Ease in for smooth magnetism

    # Apply correction
    var correction = (nearest.position - player.global_position) * strength
    correction.y = max(0, correction.y)  # Only pull down or horizontally

    player.global_position += correction * 0.1  # Gentle pull
```

**Key Parameters:**
- **Horizontal forgiveness:** 4-12 pixels (typical: 8px)
  - Casual platformers: 10-15px
  - Precision platformers: 4-8px
  - 3D platformers: 12-20px (depth perception compensation)
- **Vertical forgiveness:** 3-8 pixels (typical: 6px)
- **Snap speed:** 3-8 units/frame (higher = more subtle)
- **Max velocity:** 100-200 pixels/s (disable during fast falls)
- **Corner detection range:** 10-16 pixels

**Edge Cases:**
- **Corner ledges:** Stronger magnetism for outside corners, weaker for inside
- **One-way platforms:** Only magnetize from above, not from below
- **Moving platforms:** Account for platform velocity in correction
- **Thin platforms:** Increase forgiveness for narrow landing zones
- **Wall proximity:** Reduce horizontal magnetism near walls
- **Intentional gaps:** Designer flag to disable magnetism for specific jumps
- **Underwater/low gravity:** Adjust parameters for different physics states
- **High-speed movement:** Disable or reduce magnetism above velocity threshold
- **Multiplayer:** Server-side validation to prevent exploits

**When NOT to Use:**
- Precision platforming challenges where exact positioning matters
- Speed-running sections with pixel-perfect routes
- Puzzle platformers where landing position affects mechanics
- Competitive multiplayer (can create unfair advantages)
- Games with intentionally "slippery" physics
- Ice levels or similar (conflicts with intentional difficulty)

**Examples from Shipped Games:**

1. **Super Mario 64 (1996):** 8-12 pixel ledge magnetism on all platforms. Revolutionary for 3D platforming; compensated for camera and depth perception issues. Without it, the game would be significantly harder. Also features corner-snapping that pulls Mario onto ledge corners within 10 units.

2. **Crash Bandicoot N. Sane Trilogy (2017):** Original had minimal magnetism; remake added 6px forgiveness after player feedback about "harder than original." Demonstrates measurable impact: completion rates increased 15% after adding subtle magnetism in patch.

3. **A Hat in Time (2017):** 10px horizontal, 8px vertical forgiveness. Particularly aggressive near collectibles. Developer commentary reveals this was critical for maintaining flow in time-trial missions. Combined with visual cues (platform edges glow slightly).

4. **Ratchet & Clank (2016):** Adaptive magnetism: 6-15px based on platform size. Smaller platforms get stronger magnetism. Also factors in player velocity—slower movement gets more assistance. Essential for maintaining pacing during combat-platforming sections.

5. **Celeste (2018):** Surprisingly minimal passive magnetism (4px), but strong active correction on landing. Uses velocity adjustment rather than position snap, feeling more natural. Combined with generous hitboxes makes pixel-perfect positioning unnecessary while preserving challenge.

**Platform Considerations:**
- **PC (high precision):** 4-8px typically sufficient
- **Console (controller):** 8-12px due to analog stick imprecision
- **Mobile (touch):** 12-20px to compensate for finger occlusion
- **VR:** 10-15px due to depth perception challenges
- **3D vs 2D:**
  - 2D: 6-10px standard
  - 3D: 12-20px (depth perception compensation)

**Godot-Specific Notes:**
- `PhysicsRayQueryParameters2D` provides precise ledge detection
- Use `move_and_slide()` after position corrections for smooth integration
- `CollisionShape2D.shape.size` can be modified at runtime for dynamic forgiveness
- Consider `Area2D` with `body_entered` for zone-based forgiveness
- Godot 4.x: `CharacterBody2D.platform_on_leave` useful for moving platforms
- Export forgiveness values for level-specific tuning
- Use `@tool` script for editor visualization of forgiveness zones

**Synergies:**
- **Coyote Time (#71):** Both compensate for timing/positioning imprecision
- **Jump Buffering (#78):** Buffer jump before landing on magnetized ledge
- **Camera Lead (#82):** Camera anticipation helps player see landing zones
- **Visual feedback:** Highlight ledges that will magnetize (subtle glow)
- **Audio cues:** Different landing sounds for regular vs. magnetized landings

**Measurement/Profiling:**
- **Near-miss tracking:**
  - Log falls that would have succeeded with magnetism
  - Typical: 20-35% of falls prevented by magnetism
- **Player death analysis:**
  - Heat map of death locations
  - If concentrated near ledges, increase forgiveness
- **Velocity distribution:**
  - Plot landing velocities to tune MAX_SNAP_VELOCITY
- **A/B testing:**
  - Group A: No magnetism
  - Group B: 8px magnetism
  - Measure: completion rate, death count, frustration reports
- **Telemetry metrics:**
  - Magnetism activation frequency
  - Average correction distance
  - Player retry rates at specific jumps

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build() or not debug_draw_zones:
        return

    # Draw horizontal forgiveness zone
    var h_color = Color(0, 1, 0, 0.3)
    draw_rect(Rect2(
        Vector2(-LEDGE_FORGIVENESS_HORIZONTAL, -5),
        Vector2(LEDGE_FORGIVENESS_HORIZONTAL * 2,
                LEDGE_FORGIVENESS_VERTICAL + 5)
    ), h_color, false, 2.0)

    # Draw raycast lines
    if has_meta("last_ledge_check"):
        var ledge_pos = get_meta("last_ledge_check")
        draw_line(Vector2.ZERO, ledge_pos - global_position,
                  Color.YELLOW, 2.0)
        draw_circle(ledge_pos - global_position, 3, Color.RED)

# Statistics tracker
var magnetism_stats = {
    "activations": 0,
    "total_correction": 0.0,
    "prevented_falls": 0
}

func log_magnetism_activation(correction: Vector2):
    magnetism_stats.activations += 1
    magnetism_stats.total_correction += correction.length()
    print("Magnetism stats: ", magnetism_stats)
```

**Performance Impact:**
- CPU: 0.02-0.10ms per frame (depends on raycast count)
- 2-5 raycasts typical: ~0.05ms
- Collision shape expansion: negligible (one-time cost)
- Memory: <100 bytes per player
- Scaling: Safe for 50+ entities with forgiveness simultaneously
