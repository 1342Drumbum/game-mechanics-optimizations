### 83. Air Strafing (Quake-Style)

**Category:** Game Feel - Movement

**Problem It Solves:** Movement feels restrictive and predictable in mid-air. Players lack air control, making jumping feel "on-rails" and removing skill expression. Traditional physics feels too realistic, limiting player agency and reducing movement depth. Skilled players want to optimize aerial trajectories for speed and positioning.

**Technical Explanation:**
Air strafing allows players to gain speed and control direction mid-air by combining directional input with mouse movement. Unlike realistic physics, velocity increases when strafe input direction aligns with current velocity vector at specific angles (typically 0-45 degrees). The technique applies acceleration perpendicular to view direction while preserving forward momentum, creating a curved trajectory. Core mechanic: `velocity += strafe_direction * air_accel * dot(strafe_dir, wish_dir) * delta`. The dot product creates the "sweet spot" angle where maximum acceleration occurs. When implemented correctly, skilled players can chain strafes to gain significant speed (150-300% base speed).

**Algorithmic Complexity:**
- Per-frame calculation: O(1) - single vector math operation
- No collision checks during air movement
- CPU cost: ~0.01ms per player (negligible)
- Memory: 3 Vector3 variables (~36 bytes)

**Implementation Pattern:**
```gdscript
class_name AirStrafingController extends CharacterBody3D

# Air control parameters
@export var air_acceleration: float = 800.0
@export var air_speed_cap: float = 30.0
@export var air_control: float = 0.3  # How much control in air vs ground
@export var strafe_speed_boost: float = 1.0  # Classic Quake value

var wish_dir: Vector3 = Vector3.ZERO

func _physics_process(delta: float) -> void:
    if not is_on_floor():
        apply_air_strafe(delta)
    move_and_slide()

func apply_air_strafe(delta: float) -> void:
    # Get input direction relative to camera
    var input_dir = Input.get_vector("left", "right", "forward", "back")
    var camera_basis = $Camera3D.global_transform.basis
    wish_dir = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    if wish_dir.length() < 0.1:
        return

    # Calculate current horizontal velocity
    var current_velocity = Vector3(velocity.x, 0, velocity.z)
    var current_speed = current_velocity.length()

    # Project wish direction onto horizontal plane
    var wish_speed = wish_dir.length() * air_speed_cap

    # Calculate acceleration direction
    # This is the key: acceleration is perpendicular to current velocity
    var accel_dir = wish_dir

    # CPM/Quake-style air control
    var projection = current_velocity.dot(wish_dir)
    var accel = air_acceleration * delta

    # Only accelerate if not exceeding speed cap in that direction
    if projection + accel > wish_speed:
        accel = max(0, wish_speed - projection)

    # Apply acceleration - scaled by air control factor
    velocity.x += accel_dir.x * accel * air_control
    velocity.z += accel_dir.z * accel * air_control

    # Optional: Speed boost when strafe angle is optimal (30-45 degrees)
    var strafe_angle = current_velocity.angle_to(wish_dir)
    if current_speed > 1.0 and strafe_angle > 0.3 and strafe_angle < 0.8:
        var boost = strafe_speed_boost * delta
        velocity.x += wish_dir.x * boost
        velocity.z += wish_dir.z * boost

func get_optimal_strafe_angle() -> float:
    # Helper for UI/tutorial - shows optimal angle
    return 0.523599  # ~30 degrees in radians

# Advanced: Separate air acceleration for forward/strafe
func apply_advanced_air_strafe(delta: float) -> void:
    var input_dir = Input.get_vector("left", "right", "forward", "back")
    var camera_basis = $Camera3D.global_transform.basis
    wish_dir = (camera_basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    if wish_dir.length() < 0.1:
        return

    var current_velocity = Vector3(velocity.x, 0, velocity.z)

    # Split into forward and strafe components
    var forward = camera_basis.z
    var right = camera_basis.x

    var forward_speed = current_velocity.dot(forward)
    var strafe_speed = current_velocity.dot(right)

    # Different acceleration for forward vs strafe
    var forward_accel = air_acceleration * input_dir.y * air_control * delta
    var strafe_accel = air_acceleration * input_dir.x * air_control * 1.2 * delta

    velocity += forward * forward_accel
    velocity += right * strafe_accel
```

**Key Parameters:**
- **air_acceleration:** 600-1200 units/s² (800 typical)
- **air_speed_cap:** 20-40 units/s (30 balanced)
- **air_control:** 0.2-0.5 (0.3 = 30% of ground control)
- **strafe_speed_boost:** 0.5-2.0 (1.0 = no boost, 1.5 = classic Quake feel)
- **optimal_angle:** 25-45 degrees (30° sweet spot)
- **max_air_speed:** 300-500% of base speed (uncapped for advanced players)

**Edge Cases:**
- **Zero velocity start:** Handle division by zero when calculating angles
- **Vertical velocity:** Only apply to horizontal (XZ) plane, preserve Y velocity
- **Wall collision mid-air:** Don't reset velocity, allow wall sliding
- **Max speed cap:** Decide if hard cap or soft diminishing returns
- **Input buffering:** Store input during brief floor contact for smooth transitions
- **Controller vs M+KB:** May need separate tuning for analog stick precision

**When NOT to Use:**
- Realistic military/tactical shooters (breaks immersion)
- Puzzle platformers requiring precise jump arcs
- Games with strict physics simulation (racing, sports)
- Casual/mobile games where complexity alienates players
- Narrative-focused games where movement shouldn't be skill-gated
- When your level design requires predictable jump trajectories

**Examples from Shipped Games:**

1. **Quake III Arena / Quake Champions:** Original implementation - air strafing is core to competitive play. Pro players reach 600-800 ups (units per second) through strafe jumping. Optimal angle ~10-30° from current velocity. Essential for circle jumping starts and maintaining speed.

2. **Team Fortress 2:** Simplified air strafing on Soldier/Demo rocket/sticky jumping. Air acceleration capped at ~1000 u/s² but allows significant directional control. "Pogo" jumping technique uses air strafing to chain explosive jumps. Critical for competitive 6v6 play.

3. **CS:GO/CS2:** Limited air strafing (air_accelerate 12 vs ground 5.5). Preserves some air control without enabling "bhop" speed exploitation. Allows subtle mid-air corrections but maintains gunplay focus. Optimal for slightly curving long jumps.

4. **Titanfall 2:** Advanced air strafing combined with wallrunning. Air speed cap removed for chained movement. Grapple + air strafe = 40+ m/s speeds. Core to "movement meta" that defined the game. Tutorial teaches optimal angles.

5. **Apex Legends:** Modified Source engine air strafing. "Tap-strafing" on PC allows sharp turns (controversial, console can't replicate). Bunny hopping removed but air momentum preserved. Balance between accessibility and skill ceiling.

**Platform Considerations:**
- **PC (M+KB):** Full implementation - precise mouse control enables optimal angles
- **Console (Controller):** Reduce air acceleration 20-30%, wider optimal angle range (analog stick less precise)
- **Mobile:** Consider auto-assist or simplified controls, reduce skill requirement
- **VR:** Motion sickness risk - reduce effectiveness or disable entirely
- **Competitive:** Ensure consistent framerate doesn't affect acceleration values (delta-time independent)

**Godot-Specific Notes:**
- Use `CharacterBody3D.velocity` directly, don't use `move_and_slide_with_snap()` in air
- `is_on_floor()` can be framerate-dependent, add 0.1s coyote time buffer
- Camera basis calculations each frame are efficient (pre-multiplied matrices)
- For multiplayer: Server-authoritative with client prediction
- Export parameters with `@export_range()` for easy designer tuning
- Consider `PhysicsServer3D` for more control over collision response
- Godot 4.x: `move_and_slide()` auto-applies delta, don't double-apply

**Synergies:**
- **Jump Height Variability (#85):** Combine variable jump height with air strafe for skill expression
- **Momentum Curves (#89):** Air strafe feeds into momentum preservation systems
- **Strafe Jump Technique (#92):** Ground-based strafe jumping chains into air strafing
- **Wall Running Momentum (#87):** Exit wall run with preserved speed, air strafe to next wall
- **Slide Momentum (#88):** Slide → jump → air strafe = speed retention combo
- **Acceleration Curves (#84):** Apply ease-in to air acceleration for more predictable feel

**Measurement/Profiling:**
- **Speed tracking:** Display current velocity magnitude during playtesting
- **Optimal angle indicator:** Visual feedback when player hits 30° sweet spot (tutorial mode)
- **Heatmaps:** Track where players gain/lose speed in levels
- **Analytics:** Measure % of jumps utilizing air strafe (low = needs tutorial)
- **A/B testing:** Test air_control values 0.2 vs 0.3 vs 0.4, measure player retention
- **Frame time:** Should add <0.01ms per player (vector math is cheap)
- **Playtesting feedback:** "Does air control feel responsive?" "Can you understand why you're gaining speed?"
- **Skill curve:** Track speed improvement over first 10 hours of play
