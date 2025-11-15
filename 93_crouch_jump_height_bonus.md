### 93. Crouch Jump Height Bonus

**Category:** Game Feel - Movement

**Problem It Solves:** Fixed jump height limits level design options. Players can't reach challenging areas without double jump. No skill-based alternative for vertical progression. Crouch feels disconnected from other movement mechanics. Missing depth in single-jump platforming.

**Technical Explanation:**
Crouching before/during jump adds height bonus (10-30% increase) by compressing character collision capsule and applying stronger upward force. Implementation: if crouch held during jump input, apply `jump_velocity * crouch_bonus_multiplier` (typically 1.15-1.3x). Optionally require crouch timing window (0.1-0.3s before jump) for skill expression. Physics justification: compressed stance allows stronger leg extension. Collision shape shrinks during crouch (standing height 2m → 1m), enabling crawl under obstacles. Crouch jump combines both: lower starting position + higher velocity = maximum clearance. Advanced variant: "crouch-hop" chains multiple crouch jumps for rhythmic height gain. Common in Source engine games where crouch jump is 20-25% higher than normal.

**Algorithmic Complexity:**
- Jump calculation: O(1) - simple multiplication
- Collision shape switch: O(1) - toggle collision shape
- State check: O(1) - boolean flags
- Memory: ~16 bytes (crouch state, timer)
- CPU: <0.002ms per jump event

**Implementation Pattern:**
```gdscript
class_name CrouchJumpController extends CharacterBody3D

# Jump parameters
@export var base_jump_velocity: float = 10.0
@export var crouch_jump_multiplier: float = 1.25  # 25% height bonus
@export var crouch_jump_timing_window: float = 0.3  # Must crouch within this time

# Crouch parameters
@export var standing_height: float = 2.0
@export var crouched_height: float = 1.0
@export var crouch_transition_speed: float = 10.0  # Speed of height change

# Optional: Require pre-crouch (skill-based)
@export var require_pre_crouch: bool = false  # Must crouch before jump
@export var pre_crouch_duration: float = 0.1  # Minimum crouch time

# State tracking
var is_crouching: bool = false
var crouch_timer: float = 0.0
var current_height: float = 2.0
var can_crouch_jump: bool = false

# Collision shapes
@onready var standing_collision: CollisionShape3D = $StandingCollision
@onready var crouched_collision: CollisionShape3D = $CrouchedCollision
@onready var ceiling_check: RayCast3D = $CeilingCheck

func _ready() -> void:
    current_height = standing_height
    crouched_collision.disabled = true

func _physics_process(delta: float) -> void:
    # Update crouch state
    update_crouch_state(delta)

    # Handle jump input
    if Input.is_action_just_pressed("jump") and is_on_floor():
        perform_jump()

    move_and_slide()

func update_crouch_state(delta: float) -> void:
    var wants_to_crouch = Input.is_action_pressed("crouch")

    if wants_to_crouch and not is_crouching:
        start_crouch()
    elif not wants_to_crouch and is_crouching:
        try_stand_up()

    # Update crouch timer
    if is_crouching:
        crouch_timer += delta

        # Check if pre-crouch requirement met
        if require_pre_crouch and crouch_timer >= pre_crouch_duration:
            can_crouch_jump = true
    else:
        crouch_timer = 0.0
        can_crouch_jump = false

    # Animate height transition
    if is_crouching:
        current_height = move_toward(current_height, crouched_height, crouch_transition_speed * delta)
    else:
        current_height = move_toward(current_height, standing_height, crouch_transition_speed * delta)

func start_crouch() -> void:
    is_crouching = true
    crouch_timer = 0.0

    # Switch collision shapes
    standing_collision.disabled = true
    crouched_collision.disabled = false

    # Adjust visual height (model/animation)
    update_visual_height()

func try_stand_up() -> void:
    # Check if there's room to stand up
    if ceiling_check.is_colliding():
        # Not enough clearance, stay crouched
        return

    is_crouching = false
    can_crouch_jump = false

    # Switch collision shapes
    standing_collision.disabled = false
    crouched_collision.disabled = true

    update_visual_height()

func perform_jump() -> void:
    var jump_velocity = base_jump_velocity

    # Check for crouch jump conditions
    var is_crouch_jumping = false

    if require_pre_crouch:
        # Skill-based: Must have crouched for minimum duration
        if can_crouch_jump:
            jump_velocity *= crouch_jump_multiplier
            is_crouch_jumping = true
    else:
        # Simple: Just check if crouch is held
        if is_crouching:
            jump_velocity *= crouch_jump_multiplier
            is_crouch_jumping = true

    # Alternative: Check timing window (was crouched recently)
    if not require_pre_crouch and not is_crouching and crouch_timer < crouch_jump_timing_window:
        jump_velocity *= crouch_jump_multiplier
        is_crouch_jumping = true

    # Apply jump
    velocity.y = jump_velocity

    # Optional: Visual/audio feedback
    if is_crouch_jumping:
        play_crouch_jump_effects()
    else:
        play_normal_jump_effects()

    # Reset crouch timer
    crouch_timer = 0.0

func update_visual_height() -> void:
    # Adjust model position/scale or animation
    if has_node("PlayerModel"):
        var target_scale = crouched_height / standing_height if is_crouching else 1.0
        $PlayerModel.scale.y = target_scale

func play_crouch_jump_effects() -> void:
    # Distinct sound/particle for crouch jump
    if has_node("CrouchJumpSound"):
        $CrouchJumpSound.play()

func play_normal_jump_effects() -> void:
    if has_node("JumpSound"):
        $JumpSound.play()

# Advanced: Crouch-hop (repeated crouch jumps)
var crouch_hop_chain: int = 0

func track_crouch_hop() -> void:
    if is_crouch_jumping:
        crouch_hop_chain += 1
    else:
        crouch_hop_chain = 0

    # Optional: Bonus for sustained crouch hopping (like bhop)
    if crouch_hop_chain > 3:
        # Apply small speed bonus
        var horizontal_speed = Vector3(velocity.x, 0, velocity.z).length()
        horizontal_speed *= 1.05  # 5% bonus per chain
        # Apply bonus...

# Alternative: Variable crouch depth (hold duration)
@export var max_crouch_depth: float = 0.5  # Meters lower
@export var crouch_charge_time: float = 0.5  # Time to full crouch

func calculate_variable_crouch_bonus() -> float:
    # Longer crouch = higher jump
    var charge_ratio = min(crouch_timer / crouch_charge_time, 1.0)
    return 1.0 + (crouch_jump_multiplier - 1.0) * charge_ratio

# Helper: Calculate crouch jump height
func calculate_crouch_jump_height() -> float:
    var jump_vel = base_jump_velocity * crouch_jump_multiplier
    var gravity = ProjectSettings.get_setting("physics/3d/default_gravity", 9.8)

    # Height formula: h = v² / (2g)
    return (jump_vel * jump_vel) / (2.0 * gravity)

func calculate_normal_jump_height() -> float:
    var gravity = ProjectSettings.get_setting("physics/3d/default_gravity", 9.8)
    return (base_jump_velocity * base_jump_velocity) / (2.0 * gravity)

# UI indicator (tutorial/learning aid)
func get_crouch_jump_ready_status() -> String:
    if require_pre_crouch:
        if can_crouch_jump:
            return "CROUCH JUMP READY"
        else:
            var remaining = pre_crouch_duration - crouch_timer
            return "Crouch for %.1fs" % remaining
    else:
        return "Hold crouch to jump higher"

# Advanced: Collision detection during crouch
func check_crouch_clearance() -> bool:
    # Ensure player can actually crouch (not in tight space)
    # Use raycast or shape query
    return true  # Simplified

# Debug visualization
func _draw_debug() -> void:
    if is_crouching:
        var normal_height = calculate_normal_jump_height()
        var crouch_height = calculate_crouch_jump_height()
        var bonus = crouch_height - normal_height

        print("Normal: %.1fm | Crouch: %.1fm | Bonus: +%.1fm (%.0f%%)" % [
            normal_height, crouch_height, bonus,
            ((crouch_height / normal_height - 1.0) * 100.0)
        ])
```

**Key Parameters:**
- **base_jump_velocity:** 8-12 units/s (10 typical)
- **crouch_jump_multiplier:** 1.15-1.35 (1.25 = 25% height bonus)
- **crouch_timing_window:** 0.2-0.5s (0.3s forgiving)
- **pre_crouch_duration:** 0.05-0.2s (if skill-based, 0.1s balanced)
- **standing_height:** 1.8-2.0m (realistic human)
- **crouched_height:** 0.9-1.2m (half standing height)
- **crouch_transition_speed:** 8-15 units/s (10 = smooth)

**Edge Cases:**
- **Mid-air crouch:** Allow crouch toggle in air (doesn't affect current jump)
- **Ceiling during crouch jump:** Collision stops upward motion, preserve horizontal
- **Uncrouch during jump:** Maintain jump height, transition smoothly
- **Slope crouch jumps:** Height bonus applies, adjust for slope angle
- **Damage cancels crouch:** Force stand if hit while crouched
- **Water/swimming:** Disable crouch jump, use different mechanics
- **Controller deadzone:** Trigger crouch at >0.8 on analog trigger (if using trigger)

**When NOT to Use:**
- **Games without crouch mechanic:** Obviously (add crouch first if needed)
- **Pure combat games:** Fighting games, MOBAs (no vertical platforming)
- **Realistic sports games:** Basketball, soccer (players don't crouch jump)
- **Puzzle games without platforming:** Tetris, match-3
- **Top-down games:** 2D perspective doesn't show height difference
- **When level design doesn't support it:** If all gaps are normal jump height

**Examples from Shipped Games:**

1. **Half-Life Series:** Classic crouch jump (+20% height). Can reach 2.4m vs 2.0m normal jump. Essential for shortcut routes. Speedrunners use extensively. No timing requirement - just hold crouch. Can crouch mid-air to fit through gaps. Became iconic Source engine technique.

2. **Counter-Strike (all versions):** Crouch jump +25% height, critical for competitive play. Used to reach boost spots, jump on boxes. "KZ" (climb) maps entirely based on crouch jumping mastery. Can combine with strafing for long jumps. Skill-based execution (timing matters slightly).

3. **Minecraft:** Crouch jump clears 1.25 blocks vs 1.0 normal. Simple implementation (hold shift + space). Essential for parkour maps. No timing window - always works if crouched. Can maintain crouch in air. Enables 4-block long jumps with sprint.

4. **Team Fortress 2:** 20% height bonus, class-dependent effectiveness. Scout's double jump can use crouch on second jump. Soldier/Demo rocket jumps amplified by crouch. Competitive meta uses crouch jumps for positioning. Source engine implementation (like Half-Life).

5. **Apex Legends:** Subtle crouch jump (10-15% bonus). Used to climb ledges slightly out of reach. Less pronounced than Source games. Combined with slide-jump for mobility. Tactical advantage in combat (unpredictable movement).

**Platform Considerations:**
- **PC (Keyboard):** Easy to execute (Ctrl + Space simultaneously)
- **Console (Controller):** Button mapping important (B/Circle + A/X can be awkward)
- **Mobile:** Touch controls - dedicated "crouch jump" button vs two buttons
- **Accessibility:** Option to make crouch jump automatic (auto-crouch on jump input)
- **Competitive:** Ensure timing consistent across all framerates
- **VR:** Physical crouch + jump button (novel interaction)

**Godot-Specific Notes:**
- Multiple `CollisionShape3D` nodes, toggle `.disabled` property
- `RayCast3D` pointing upward for ceiling detection (1-2m range)
- `move_toward()` for smooth height transitions
- AnimationTree: Blend crouch animations based on `current_height`
- Export multiplier for easy designer tuning
- Godot 4.x: Better collision shape switching than 3.x
- `CharacterBody3D.velocity.y` for jump force application
- Visual feedback: Adjust camera Y position during crouch

**Synergies:**
- **Jump Height Variability (#85):** Crouch jump + hold jump = maximum height
- **Strafe Jump (#92):** Crouch jump while strafe jumping = chained height boost
- **Slide Momentum (#88):** Slide → crouch jump = speed + height combo
- **Wall Jump:** Crouch jump off wall for extra height
- **Double Jump:** Crouch on second jump for even higher reach
- **Ledge Grab (#90):** Crouch jump increases grab range effectively
- **Parkour System:** Core component of movement skill progression

**Measurement/Profiling:**
- **Usage rate:** % of jumps that use crouch (20-40% = healthy integration)
- **Success rate:** Do crouch jumps reach intended areas? (>90% if designed correctly)
- **Height distribution:** Normal vs crouch jump heights (should see clear separation)
- **Tutorial completion:** % players who learn crouch jump (target >85%)
- **Competitive usage:** Do high-skill players use extensively? (should be yes)
- **False positives:** Accidental crouch jumps (should be <5%)
- **Timing precision:** If skill-based, track timing accuracy (0.05-0.15s variance typical)
- **Performance:** Collision shape swap <0.02ms, calculation <0.002ms
- **Level design validation:** Are crouch jump gaps learnable? (playtest 20+ players)
