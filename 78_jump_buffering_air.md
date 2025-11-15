### 78. Jump Buffering in Air

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players pressing jump button before landing and having the input ignored, forcing them to time jump exactly on landing frame (16.67ms window at 60fps). Without air jump buffering, 25-40% of landing jumps fail because players press 50-150ms too early. Creates perception of "unresponsive double jumps" or "broken air controls." Particularly frustrating in precision platformers where maintaining momentum through jump chains is critical.

**Technical Explanation:**
Jump buffering in air stores jump inputs pressed while airborne, then executes them immediately upon landing. When jump pressed mid-air (and all air-jumps exhausted), store timestamp. On landing detection, check if buffered jump is within window (typically 150-200ms). If valid, execute jump on the landing frame, preserving momentum.

Critical distinction from ground input buffering (#72): This specifically handles aerial state → grounded state transition. Allows players to "queue" their next jump before current jump completes, creating seamless jump chains without frame-perfect timing. Combined with coyote time (#71), creates 300-400ms total input window (150ms before + 150ms after landing = massive forgiveness).

Key implementation detail: Buffer should NOT consume air-jump charges—only activate on ground landing. Prevents accidental double-jump consumption.

**Algorithmic Complexity:** O(1) - simple buffer check on landing event

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Jump configuration
const JUMP_VELOCITY = -400.0
const AIR_JUMP_COUNT = 1  # Number of allowed air jumps
const JUMP_BUFFER_WINDOW = 0.15  # 150ms / 9 frames at 60fps

# Jump state
var air_jumps_remaining: int = AIR_JUMP_COUNT
var is_grounded: bool = false
var jump_buffer_time: float = -1.0  # Timestamp of buffered jump (-1 = none)

func _physics_process(delta: float) -> void:
    var was_grounded = is_grounded
    is_grounded = is_on_floor()

    # Detect landing
    if not was_grounded and is_grounded:
        on_landed()

    # Handle jump input
    if Input.is_action_just_pressed("jump"):
        attempt_jump()

    # Decay jump buffer
    if jump_buffer_time >= 0:
        var buffer_age = Time.get_ticks_msec() / 1000.0 - jump_buffer_time
        if buffer_age > JUMP_BUFFER_WINDOW:
            jump_buffer_time = -1.0  # Expired

    # Apply gravity and movement
    if not is_grounded:
        velocity.y += ProjectSettings.get_setting("physics/2d/default_gravity") * delta

    move_and_slide()

func attempt_jump() -> void:
    var current_time = Time.get_ticks_msec() / 1000.0

    # Case 1: Grounded jump (immediate)
    if is_grounded:
        execute_jump()
        return

    # Case 2: Air jump available (immediate)
    if air_jumps_remaining > 0:
        execute_air_jump()
        return

    # Case 3: No jumps available - buffer for landing
    jump_buffer_time = current_time
    print("Jump buffered (airborne, no air jumps)")

func on_landed() -> void:
    # Restore air jumps on landing
    air_jumps_remaining = AIR_JUMP_COUNT

    # Check for buffered jump
    if jump_buffer_time >= 0:
        var current_time = Time.get_ticks_msec() / 1000.0
        var buffer_age = current_time - jump_buffer_time

        if buffer_age <= JUMP_BUFFER_WINDOW:
            # Execute buffered jump immediately on landing
            execute_jump()
            print("Buffered jump executed on landing (%.0fms early)" % (buffer_age * 1000))
            jump_buffer_time = -1.0  # Consume buffer
            return

    # No buffered jump, normal landing
    jump_buffer_time = -1.0

func execute_jump() -> void:
    velocity.y = JUMP_VELOCITY
    $AnimationPlayer.play("jump")
    $JumpSound.play()
    print("Ground jump executed")

func execute_air_jump() -> void:
    velocity.y = JUMP_VELOCITY * 0.9  # Slightly weaker air jump
    air_jumps_remaining -= 1
    $AnimationPlayer.play("air_jump")
    $AirJumpParticles.restart()
    print("Air jump executed (%d remaining)" % air_jumps_remaining)
```

**Advanced Multi-Action Buffer:**
```gdscript
class_name AerialInputBuffer

# Buffer multiple actions, not just jump
enum AerialAction { JUMP, ATTACK, DASH, SPECIAL }

class BufferedAction:
    var action: AerialAction
    var timestamp: float
    var consumed: bool = false

    func _init(p_action: AerialAction, p_time: float):
        action = p_action
        timestamp = p_time

var action_buffer: Array[BufferedAction] = []
var buffer_window: float = 0.15

# Priority order (higher = execute first on landing)
const ACTION_PRIORITY = {
    AerialAction.JUMP: 3,     # Highest - maintain momentum
    AerialAction.DASH: 2,     # Medium - repositioning
    AerialAction.ATTACK: 1,   # Low - can wait
    AerialAction.SPECIAL: 1   # Low
}

func buffer_action(action: AerialAction) -> void:
    var timestamp = Time.get_ticks_msec() / 1000.0

    # Check for duplicate recent buffer
    for buffered in action_buffer:
        if buffered.action == action and not buffered.consumed:
            if timestamp - buffered.timestamp < 0.05:  # 50ms debounce
                return

    # Add to buffer
    var buffered_action = BufferedAction.new(action, timestamp)
    action_buffer.append(buffered_action)

    # Sort by priority
    action_buffer.sort_custom(func(a, b):
        return ACTION_PRIORITY[a.action] > ACTION_PRIORITY[b.action]
    )

func on_landed() -> void:
    var current_time = Time.get_ticks_msec() / 1000.0

    # Process buffered actions in priority order
    for buffered in action_buffer:
        if buffered.consumed:
            continue

        var age = current_time - buffered.timestamp
        if age > buffer_window:
            continue

        # Execute highest priority valid action, then stop
        match buffered.action:
            AerialAction.JUMP:
                execute_jump()
                buffered.consumed = true
                break  # Only execute one action
            AerialAction.ATTACK:
                execute_attack()
                buffered.consumed = true
                break
            AerialAction.DASH:
                execute_dash()
                buffered.consumed = true
                break

    # Clean expired/consumed actions
    action_buffer = action_buffer.filter(func(a):
        if a.consumed:
            return false
        var age = current_time - a.timestamp
        return age <= buffer_window
    )

# Specific use case: Hold jump for variable height
var jump_held_buffer: bool = false  # Did player hold jump during buffered input?

func buffer_jump_with_hold_state() -> void:
    var timestamp = Time.get_ticks_msec() / 1000.0
    jump_buffer_time = timestamp
    jump_held_buffer = Input.is_action_pressed("jump")

func execute_buffered_jump_with_variable_height() -> void:
    execute_jump()

    # If player was holding jump when buffered, apply full height
    if jump_held_buffer:
        # Set flag for variable jump height system
        set_meta("jump_hold_active", true)
```

**Key Parameters:**
- **Buffer window:** 0.10-0.20s (typical: 0.15s / 9 frames)
  - Precision platformers: 0.15-0.20s (generous)
  - Fast-paced action: 0.10-0.13s (tighter)
  - Combo-heavy games: 0.12-0.16s
- **Max buffered actions:** 1-3 (prevent action spam)
- **Priority levels:** 2-4 tiers for conflict resolution
- **Debounce window:** 0.03-0.05s (prevent duplicate buffers)

**Edge Cases:**
- **Landing on moving platform:** Buffer should work, preserve platform velocity
- **Landing in water/hazard:** Clear buffer on damage/death
- **Wall landing:** Separate wall-jump buffer (if applicable)
- **Ledge grab:** May cancel jump buffer, or trigger buffered ledge climb
- **Variable jump height:** Buffer should respect held vs. tapped jump
- **Double-jump interaction:** Buffer only for ground jumps, not air jumps
- **Cutscene/death:** Clear all buffers on state change
- **Ceiling collision:** Don't buffer jump if bonking head
- **One-way platforms:** Buffer should work for dropping through

**When NOT to Use:**
- Games without jumping mechanic
- Turn-based games
- Realistic physics sims (momentum preservation may feel wrong)
- When precise landing timing is core mechanic
- Swimming/flying states (different control model)
- Climbing systems (different input paradigm)

**Examples from Shipped Games:**

1. **Celeste (2018):** 0.15s jump buffer in air, stacks with 0.15s ground buffer and 0.15s coyote time. Creates nearly 500ms total input window for jump chains. Essential for advanced techniques like wave-dashing. Developer commentary: "Players should never fight the controls; buffer everything."

2. **Super Meat Boy (2010):** 0.10s air jump buffer. Critical for maintaining speed through wall-jump sequences. Combined with instant wall-jump reset, allows continuous upward movement without frame-perfect inputs. Speedrunning community relies on this heavily.

3. **Hollow Knight (2017):** 0.13s buffer for jumps and attacks. Works mid-air and on landing. Particularly important for pogo-jumping technique (down-slash bouncing). Buffer allows focusing on enemy positioning rather than timing precision.

4. **Dead Cells (2018):** 0.10s buffer for jump and dodge roll. Shorter than platformers due to combat focus. Still present to prevent input drops during aerial attacks. Dodge buffer especially important for landing combos.

5. **Ori and the Will of the Wisps (2020):** Adaptive buffer: 0.15s for standard jumps, 0.20s for bash (directional boost). Longer buffer for complex movement abilities. Essential for maintaining flow through parkour sections with multiple movement abilities.

**Platform Considerations:**
- **PC (60+ fps):** 0.12-0.15s standard, smooth frametimes make buffer feel natural
- **Console (30-60fps):** 0.15-0.18s to compensate for frame timing variance
- **Mobile (touch):** 0.18-0.25s to account for touchscreen latency
- **Input lag consideration:** Add buffer proportional to system latency
- **Network games:** Client-side buffer, server validates landing

**Godot-Specific Notes:**
- Use `is_on_floor()` for reliable ground detection
- `was_grounded` pattern essential for detecting landing event
- `Time.get_ticks_msec()` for precise buffer timing
- Store buffer in player state, not global variables
- Consider `CharacterBody2D.floor_snap_length` for slope transitions
- Export buffer window as `@export_range(0.05, 0.25)` for tuning
- Profile with Godot debugger: buffer should add <0.01ms overhead

**Synergies:**
- **Coyote Time (#71):** Combined = 300-400ms jump window (before + after landing)
- **Input Buffering (#72):** General case; air jump is specific application
- **Ledge Forgiveness (#73):** Buffer works with magnetized landings
- **Variable Jump Height:** Buffer must preserve hold/tap distinction
- **Combo Systems:** Buffer next attack during aerial recovery
- **Animation Canceling:** Buffer allows canceling landing animation with jump

**Measurement/Profiling:**
- **Buffer usage statistics:**
  - Track % of jumps that used buffer vs. direct input
  - Typical: 15-30% of jumps use buffer
  - Higher % indicates players pressing early (good)
- **Timing distribution:**
  - Plot histogram of buffer age when consumed
  - Identifies optimal buffer window (95th percentile)
- **Failed jump tracking:**
  - Log jump presses that didn't register (no buffer or ground)
  - Should be <5% with good buffer
- **Player testing:**
  - Ask playtesters to "press jump early before landing"
  - Measure success rate (target: >95%)
- **Speedrun analysis:**
  - Record buffer usage in optimal play
  - Verify buffer doesn't enable unintended skips

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    # Draw buffer status
    if jump_buffer_time >= 0:
        var current_time = Time.get_ticks_msec() / 1000.0
        var buffer_age = current_time - jump_buffer_time
        var remaining = JUMP_BUFFER_WINDOW - buffer_age

        # Buffer active indicator
        var color = Color.YELLOW
        color.a = remaining / JUMP_BUFFER_WINDOW
        draw_circle(Vector2(0, -30), 8, color)

        # Countdown text
        draw_string(ThemeDB.fallback_font, Vector2(-25, -40),
                    "Jump Buffered: %.2fs" % remaining,
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 12, color)

    # Air jump count
    var air_jump_color = Color.GREEN if air_jumps_remaining > 0 else Color.RED
    for i in range(AIR_JUMP_COUNT):
        var pos = Vector2(-20 + i * 15, -50)
        var filled = i < air_jumps_remaining
        if filled:
            draw_circle(pos, 5, air_jump_color)
        else:
            draw_arc(pos, 5, 0, TAU, 16, Color.GRAY, 2.0)

# Statistics tracking
var buffer_stats = {
    "total_jumps": 0,
    "buffered_jumps": 0,
    "buffer_ages": [],  # Distribution of buffer ages when consumed
}

func track_jump_execution(was_buffered: bool, buffer_age: float = 0.0):
    buffer_stats.total_jumps += 1

    if was_buffered:
        buffer_stats.buffered_jumps += 1
        buffer_stats.buffer_ages.append(buffer_age * 1000)  # Store in ms

    # Log every 50 jumps
    if buffer_stats.total_jumps % 50 == 0:
        print_buffer_statistics()

func print_buffer_statistics():
    var usage_percent = 100.0 * buffer_stats.buffered_jumps / buffer_stats.total_jumps

    print("=== Jump Buffer Statistics ===")
    print("  Total jumps: %d" % buffer_stats.total_jumps)
    print("  Buffered jumps: %d (%.1f%%)" % [buffer_stats.buffered_jumps, usage_percent])

    if buffer_stats.buffer_ages.size() > 0:
        var ages = buffer_stats.buffer_ages
        ages.sort()
        var median = ages[ages.size() / 2]
        var avg = ages.reduce(func(acc, val): return acc + val, 0.0) / ages.size()

        print("  Buffer timing:")
        print("    Average: %.0fms early" % avg)
        print("    Median: %.0fms early" % median)
        print("    95th percentile: %.0fms early" % ages[int(ages.size() * 0.95)])

# Visual buffer timeline
@onready var buffer_history: Array[float] = []

func update_buffer_visualization():
    var current_time = Time.get_ticks_msec() / 1000.0

    # Track when jumps were buffered
    if jump_buffer_time >= 0:
        buffer_history.append(jump_buffer_time)

    # Keep only recent history (last 5 seconds)
    buffer_history = buffer_history.filter(func(t):
        return current_time - t < 5.0
    )

func _draw_buffer_timeline():
    if buffer_history.is_empty():
        return

    var current_time = Time.get_ticks_msec() / 1000.0
    var timeline_width = 200.0
    var timeline_pos = Vector2(10, -80)

    # Draw timeline
    draw_line(timeline_pos, timeline_pos + Vector2(timeline_width, 0),
              Color.WHITE, 2.0)

    # Draw buffer events
    for buffer_time in buffer_history:
        var age = current_time - buffer_time
        var x_pos = timeline_pos.x + timeline_width - (age / 5.0) * timeline_width

        var color = Color.YELLOW if age < JUMP_BUFFER_WINDOW else Color.GRAY
        draw_circle(Vector2(x_pos, timeline_pos.y), 3, color)
```

**Performance Impact:**
- CPU: <0.01ms per frame (simple timestamp check)
- Memory: 16-24 bytes per buffered action
- No allocation during gameplay (value types only)
- Safe for 100+ entities with jump buffering simultaneously
- GC pressure: None if using fixed-size buffer arrays

**Advanced: Buffer Decay Curve:**
```gdscript
# Instead of binary cutoff, use decay curve for buffer strength
func get_buffer_strength() -> float:
    if jump_buffer_time < 0:
        return 0.0

    var current_time = Time.get_ticks_msec() / 1000.0
    var age = current_time - jump_buffer_time

    if age > JUMP_BUFFER_WINDOW:
        return 0.0

    # Exponential decay: stronger if pressed closer to landing
    var normalized_age = age / JUMP_BUFFER_WINDOW
    return 1.0 - (normalized_age * normalized_age)  # Quadratic falloff

func execute_jump_with_decay():
    var strength = get_buffer_strength()

    # Could modify jump velocity based on buffer age
    # Reward more precise timing with slightly higher jumps
    var velocity_modifier = 1.0 + (strength * 0.1)  # Up to 10% bonus
    velocity.y = JUMP_VELOCITY * velocity_modifier
```
