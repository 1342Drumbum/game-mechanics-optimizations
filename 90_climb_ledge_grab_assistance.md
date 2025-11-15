### 90. Climb/Ledge Grab Assistance

**Category:** Game Feel - Movement

**Problem It Solves:** Pixel-perfect ledge grabbing frustrates players and breaks flow. Failed grabs due to small positioning errors feel unfair. Manual ledge detection requires precise timing and positioning. Vertical level design becomes inaccessible without forgiving climbing. Skilled players still succeed but casual players struggle disproportionately.

**Technical Explanation:**
Automatic ledge detection and "magnetism" that snaps player to valid grab points within tolerance range. System uses forward/upward raycasts plus collision checks to detect ledges. When near ledge (within snap_distance, typically 0.3-0.8m), smoothly interpolate position to grab point. Implementation: detect ledge → check height validity (head-level to 0.5m above) → verify clearance above ledge → snap to position → play grab animation. Position snap uses lerp over 0.1-0.2s for smooth transition. Additional "ledge forgiveness" extends grab window 0.1-0.15s after leaving ledge (coyote time for vertical movement). Clearance check prevents grabbing ledges with insufficient landing space (raycasts check 1-2m above ledge). Creates satisfying "magnetic" feel while maintaining player agency.

**Algorithmic Complexity:**
- Ledge detection: O(1) - 3-5 raycasts per frame
- Position snap: O(1) - simple lerp calculation
- Per-frame: O(1) when near ledges, O(0) when far
- Memory: ~128 bytes (ledge point, normal, clearance data)
- CPU: <0.05ms per character (raycast overhead)

**Implementation Pattern:**
```gdscript
class_name LedgeGrabController extends CharacterBody3D

# Ledge detection parameters
@export var ledge_detect_distance: float = 1.0  # Forward raycast distance
@export var ledge_height_min: float = 1.2  # Minimum ledge height (chest level)
@export var ledge_height_max: float = 2.5  # Maximum grabbable height
@export var ledge_snap_distance: float = 0.5  # Auto-grab range
@export var ledge_forgiveness_time: float = 0.15  # Coyote time for ledges

# Snapping and positioning
@export var snap_speed: float = 8.0  # Speed of position interpolation
@export var grab_offset: Vector3 = Vector3(0, -0.3, 0.4)  # Offset from ledge
@export var min_clearance_above: float = 1.5  # Space needed above ledge

# State tracking
enum ClimbState { NONE, DETECTING, SNAPPING, HANGING, CLIMBING }
var climb_state: ClimbState = ClimbState.NONE
var ledge_point: Vector3 = Vector3.ZERO
var ledge_normal: Vector3 = Vector3.ZERO
var ledge_forgiveness_timer: float = 0.0
var snap_progress: float = 0.0
var target_grab_position: Vector3 = Vector3.ZERO

# Detection raycasts
@onready var forward_ray: RayCast3D = $ForwardRay
@onready var ledge_check_rays: Array[RayCast3D] = [
    $LedgeCheckTop,
    $LedgeCheckMiddle,
    $LedgeCheckBottom
]

func _physics_process(delta: float) -> void:
    match climb_state:
        ClimbState.NONE:
            check_ledge_detection()
        ClimbState.DETECTING:
            update_ledge_detection()
        ClimbState.SNAPPING:
            update_snap_to_ledge(delta)
        ClimbState.HANGING:
            update_hanging_state(delta)
        ClimbState.CLIMBING:
            update_climb_up(delta)

    # Ledge forgiveness timer
    if ledge_forgiveness_timer > 0:
        ledge_forgiveness_timer -= delta

    if climb_state != ClimbState.HANGING and climb_state != ClimbState.CLIMBING:
        move_and_slide()

func check_ledge_detection() -> void:
    # Only detect when in air and moving upward/forward
    if is_on_floor():
        return

    if velocity.y < -5.0:  # Falling too fast
        return

    # Check if near ledge
    var ledge_data = detect_ledge()
    if ledge_data.has("ledge_point"):
        initiate_ledge_grab(ledge_data)

func detect_ledge() -> Dictionary:
    var result = {}

    # Forward check at chest height
    var chest_height = global_position + Vector3(0, ledge_height_min, 0)
    var forward_dir = -global_transform.basis.z

    # Check for wall in front
    var space_state = get_world_3d().direct_space_state
    var wall_query = PhysicsRayQueryParameters3D.create(
        chest_height,
        chest_height + forward_dir * ledge_detect_distance
    )
    wall_query.exclude = [self]

    var wall_hit = space_state.intersect_ray(wall_query)
    if not wall_hit:
        return result  # No wall found

    var wall_point = wall_hit.position
    var wall_normal = wall_hit.normal

    # Check for ledge (gap above wall)
    var ledge_check_start = wall_point + Vector3(0, 0.5, 0) - wall_normal * 0.1
    var ledge_check_end = ledge_check_start + wall_normal * 1.0

    var ledge_query = PhysicsRayQueryParameters3D.create(
        ledge_check_start,
        ledge_check_end
    )
    ledge_query.exclude = [self]

    var ledge_hit = space_state.intersect_ray(ledge_query)
    if not ledge_hit:
        return result  # No ledge surface found

    var ledge_point_candidate = ledge_hit.position
    var ledge_height = ledge_point_candidate.y - global_position.y

    # Validate height range
    if ledge_height < ledge_height_min or ledge_height > ledge_height_max:
        return result

    # Check clearance above ledge
    if not check_ledge_clearance(ledge_point_candidate):
        return result

    # Valid ledge found
    result["ledge_point"] = ledge_point_candidate
    result["ledge_normal"] = wall_normal
    result["ledge_height"] = ledge_height

    return result

func check_ledge_clearance(ledge_point: Vector3) -> bool:
    # Check if player can stand on ledge
    var clearance_start = ledge_point
    var clearance_end = ledge_point + Vector3(0, min_clearance_above, 0)

    var space_state = get_world_3d().direct_space_state
    var clearance_query = PhysicsRayQueryParameters3D.create(
        clearance_start,
        clearance_end
    )
    clearance_query.exclude = [self]

    var clearance_hit = space_state.intersect_ray(clearance_query)
    return not clearance_hit  # True if no obstruction

func initiate_ledge_grab(ledge_data: Dictionary) -> void:
    # Check if within snap distance
    var distance = global_position.distance_to(ledge_data.ledge_point)
    if distance > ledge_snap_distance:
        # Within forgiveness range but not instant grab
        ledge_forgiveness_timer = ledge_forgiveness_time
        return

    # Start grab sequence
    climb_state = ClimbState.SNAPPING
    ledge_point = ledge_data.ledge_point
    ledge_normal = ledge_data.ledge_normal
    snap_progress = 0.0

    # Calculate target position (offset from ledge)
    target_grab_position = ledge_point + ledge_normal * grab_offset.z + Vector3(0, grab_offset.y, 0)

    # Zero velocity during snap
    velocity = Vector3.ZERO

    # Play grab animation
    play_ledge_grab_animation()

func update_snap_to_ledge(delta: float) -> void:
    snap_progress += delta * snap_speed

    # Smoothly interpolate to grab position
    global_position = global_position.lerp(target_grab_position, snap_progress)

    # Align to face wall
    var target_rotation = Vector2(-ledge_normal.z, -ledge_normal.x).angle()
    rotation.y = lerp_angle(rotation.y, target_rotation, snap_progress)

    # Complete snap
    if snap_progress >= 1.0:
        climb_state = ClimbState.HANGING
        global_position = target_grab_position

func update_hanging_state(delta: float) -> void:
    # Maintain position at ledge
    global_position = target_grab_position
    velocity = Vector3.ZERO

    # Check for climb input
    if Input.is_action_just_pressed("jump"):
        start_climb_up()
    elif Input.is_action_just_pressed("crouch") or Input.is_action_pressed("back"):
        release_ledge()

func start_climb_up() -> void:
    climb_state = ClimbState.CLIMBING
    play_climb_animation()

    # Start position tween
    var climb_target = ledge_point + Vector3(0, 0.5, 0) - ledge_normal * 0.3
    animate_climb_movement(climb_target)

func animate_climb_movement(target: Vector3) -> void:
    # Smooth climb movement
    var tween = create_tween()
    tween.tween_property(self, "global_position", target, 0.5).set_trans(Tween.TRANS_QUAD).set_ease(Tween.EASE_OUT)
    tween.finished.connect(complete_climb)

func complete_climb() -> void:
    climb_state = ClimbState.NONE
    velocity = -ledge_normal * 3.0  # Small push away from edge

func release_ledge() -> void:
    climb_state = ClimbState.NONE
    velocity = ledge_normal * 5.0 + Vector3.DOWN * 2.0  # Push away and drop

func play_ledge_grab_animation() -> void:
    # Animation system integration
    if has_node("AnimationTree"):
        $AnimationTree.set("parameters/climbing/transition_request", "grab")

func play_climb_animation() -> void:
    if has_node("AnimationTree"):
        $AnimationTree.set("parameters/climbing/transition_request", "climb_up")

# Advanced: Multiple grab points detection
func find_nearest_valid_ledge() -> Dictionary:
    # Check multiple points along detection arc
    var best_ledge = {}
    var best_distance = INF

    for angle in range(-30, 31, 15):  # -30° to +30° in 15° increments
        var rotated_dir = global_transform.basis.z.rotated(Vector3.UP, deg_to_rad(angle))
        var ledge_data = detect_ledge_at_direction(rotated_dir)

        if ledge_data.has("ledge_point"):
            var distance = global_position.distance_to(ledge_data.ledge_point)
            if distance < best_distance:
                best_distance = distance
                best_ledge = ledge_data

    return best_ledge

# Alternative: Discrete ledge markers (designer-placed)
func detect_ledge_markers() -> Dictionary:
    # Check for LedgeMarker nodes in range
    var ledge_markers = get_tree().get_nodes_in_group("ledge_markers")
    var nearest_marker = null
    var nearest_distance = ledge_snap_distance

    for marker in ledge_markers:
        var distance = global_position.distance_to(marker.global_position)
        if distance < nearest_distance:
            nearest_marker = marker
            nearest_distance = distance

    if nearest_marker:
        return {
            "ledge_point": nearest_marker.global_position,
            "ledge_normal": -marker.global_transform.basis.z,
            "ledge_height": nearest_marker.global_position.y - global_position.y
        }

    return {}
```

**Key Parameters:**
- **ledge_detect_distance:** 0.8-1.5m (1.0m standard)
- **ledge_height_min:** 1.0-1.5m (chest to head height)
- **ledge_height_max:** 2.0-3.0m (2.5m = challenging but doable)
- **ledge_snap_distance:** 0.3-0.8m (0.5m balanced forgiveness)
- **snap_speed:** 6-12 units/s (8 feels smooth)
- **ledge_forgiveness_time:** 0.1-0.2s (0.15s standard coyote time)
- **min_clearance_above:** 1.5-2.0m (prevent grab on low ceilings)

**Edge Cases:**
- **Corner ledges:** Check both axes, prioritize nearest valid point
- **Moving platforms:** Update ledge_point each frame while hanging
- **Sloped ledges:** Reject if angle >30° from horizontal
- **Thin ledges:** Check landing space depth (1m minimum)
- **Multiple nearby ledges:** Grab nearest valid ledge within cone
- **Damaged state:** Disable ledge grab when hurt/stunned
- **Over-grab:** Prevent grabbing ledges below player height

**When NOT to Use:**
- **Realistic games:** Modern military/tactical shooters
- **Fast-paced games:** Doom Eternal, Quake (ruins flow)
- **2D platformers:** Use simpler wall jump/wall slide
- **Top-down games:** No vertical movement
- **Racing/sports games:** Not applicable
- **Horror games:** Struggling to climb can add tension (design choice)

**Examples from Shipped Games:**

1. **Uncharted Series:** 0.6m snap distance, 0.2s forgiveness. Auto-grabs ledges within 60° cone ahead. Ledge markers placed by designers. Yellow paint indicates grabbable surfaces. Snap speed 10 u/s. Critical to Nathan Drake's acrobatic feel. ~80% of traversal uses ledge grabs.

2. **Assassin's Creed (modern):** 1.0m snap distance, very forgiving. 90° detection cone. Auto-climbs when holding forward. Parkour-down system auto-drops from ledges. Minimal player input required. Prioritizes flow over precision. Clearance check prevents ceiling grabs.

3. **Horizon Zero Dawn:** 0.5m snap distance, designated grab points (white/yellow ledges). Semi-automatic - requires jump input near ledge. Fast snap (12 u/s). Combat disables auto-grab. Clear visual language for climbable surfaces. ~40% of exploration uses climbing.

4. **The Last of Us Part II:** 0.4m snap, realistic approach. Requires explicit jump input (no auto-grab). Slower snap (6 u/s) for weighty feel. Limited to obvious ledges (waist-high or above). Clearance checks strict. Maintains grounded, believable movement.

5. **Breath of the Wild:** Climb anything philosophy - no ledge grab, continuous climbing. Stamina-based system replaces discrete grabs. Different paradigm: "is surface climbable?" vs "is this a ledge?" No snap/magnetism, direct player control. Shows alternative to ledge grab design.

**Platform Considerations:**
- **All platforms:** Core mechanic, no variance needed
- **Mobile:** Increase snap_distance to 0.8m (less precise touch input)
- **VR:** First-person ledge grab with hand tracking (different implementation)
- **Controller:** Aim assist cone wider (±20°) vs mouse (±10°)
- **Accessibility:** Option for auto-climb (no button press required)
- **Performance:** Raycasts are cheap, <0.05ms for 5 rays

**Godot-Specific Notes:**
- `PhysicsRayQueryParameters3D` for 3D ledge detection
- `RayCast3D` nodes for persistent detection (enable/disable as needed)
- Use collision layers for climbable surfaces (e.g., layer 3 = "climbable")
- `Tween` for smooth snap/climb animations
- `AnimationTree` state machine for climb states
- Godot 4.x: Improved raycast performance vs 3.x
- Consider `ShapeCast3D` for more complex ledge shapes
- Markers: Use `Node3D` with custom script in "ledge_markers" group

**Synergies:**
- **Wall Running (#87):** Transition from wall run to ledge grab
- **Jump Height Variability (#85):** Jump near ledge triggers grab
- **Landing Recovery (#86):** Bypass recovery with ledge grab
- **Coyote Time:** Ledge forgiveness extends effective grab window
- **Double Jump:** Second jump can initiate ledge grab attempt
- **Climb Stamina System:** Limit hang time (optional difficulty mechanic)
- **Combat:** Ledge grab during combat creates tactical positioning

**Measurement/Profiling:**
- **Grab success rate:** % of intended grabs that succeed (target >95%)
- **False positive rate:** Unwanted grabs (target <5%)
- **Average snap distance:** Mean distance when grabs occur (0.2-0.4m ideal)
- **Forgiveness usage:** % grabs using coyote time (10-20% = good tuning)
- **Player feedback:** "Do ledges feel easy to grab?" (target >90% yes)
- **Clearance rejections:** How often valid-looking ledges rejected? (minimize)
- **Raycast performance:** <0.05ms per frame (5 rays × 0.01ms each)
- **Level design:** Are all intended ledges being grabbed? (playtest verification)
