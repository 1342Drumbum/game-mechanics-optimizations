### 72. Input Buffering

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players pressing buttons slightly early (50-150ms before action is valid) and having inputs ignored. Without buffering, 20-40% of intended actions fail because timing precision required is unrealistic (~3-5 frames at 60fps). Results in "eaten inputs," player frustration, and perception of unresponsive controls. Particularly severe during animation locks, landing recovery, or state transitions.

**Technical Explanation:**
Input buffering stores recent button presses in a time-stamped queue (typically 150-200ms window). Each frame, check if any buffered inputs can now execute based on current state. When an action becomes valid, consume the oldest matching input from buffer. This transforms "exact timing required" into "press anywhere in window," expanding input acceptance from 1-2 frames to 9-12 frames at 60fps.

System maintains separate buffers for different input types (jump, attack, dash, etc.) to prevent conflicts. Buffer entries include: input type, timestamp, and consumed flag. On state change (e.g., landing from jump), immediately check buffers for pending actions. This allows players to "queue" their next action before current one finishes.

**Algorithmic Complexity:**
- Buffer insertion: O(1)
- Buffer check: O(n) where n = buffer size (typically 5-10 entries)
- Cleanup: O(n) to remove expired entries
- Overall: O(1) amortized cost per frame

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Buffer configuration
const BUFFER_WINDOW = 0.15  # 150ms / 9 frames at 60fps
const MAX_BUFFER_SIZE = 10  # Prevent unbounded growth

# Input buffer structure
class InputBufferEntry:
    var action: String
    var timestamp: float
    var consumed: bool = false

    func _init(p_action: String, p_time: float):
        action = p_action
        timestamp = p_time

var input_buffer: Array[InputBufferEntry] = []

func _physics_process(delta: float) -> void:
    # Capture inputs into buffer
    var current_time = Time.get_ticks_msec() / 1000.0

    if Input.is_action_just_pressed("jump"):
        add_to_buffer("jump", current_time)

    if Input.is_action_just_pressed("attack"):
        add_to_buffer("attack", current_time)

    if Input.is_action_just_pressed("dash"):
        add_to_buffer("dash", current_time)

    # Clean expired entries
    clean_buffer(current_time)

    # Process buffered inputs based on current state
    if can_jump():
        if consume_buffered_input("jump", current_time):
            execute_jump()

    if can_attack():
        if consume_buffered_input("attack", current_time):
            execute_attack()

    if can_dash():
        if consume_buffered_input("dash", current_time):
            execute_dash()

func add_to_buffer(action: String, timestamp: float) -> void:
    # Prevent duplicate entries within short time
    for entry in input_buffer:
        if entry.action == action and not entry.consumed:
            if timestamp - entry.timestamp < 0.05:  # 50ms debounce
                return

    # Add new entry
    var entry = InputBufferEntry.new(action, timestamp)
    input_buffer.append(entry)

    # Limit buffer size (FIFO)
    while input_buffer.size() > MAX_BUFFER_SIZE:
        input_buffer.pop_front()

func consume_buffered_input(action: String, current_time: float) -> bool:
    # Find oldest valid input of this type
    for entry in input_buffer:
        if entry.action == action and not entry.consumed:
            if current_time - entry.timestamp <= BUFFER_WINDOW:
                entry.consumed = true
                return true

    return false

func clean_buffer(current_time: float) -> void:
    # Remove expired or consumed entries
    input_buffer = input_buffer.filter(func(entry):
        if entry.consumed:
            return false
        if current_time - entry.timestamp > BUFFER_WINDOW:
            return false
        return true
    )

func execute_jump():
    velocity.y = JUMP_VELOCITY
    print("Buffered jump executed")

func execute_attack():
    $AnimationPlayer.play("attack")
    print("Buffered attack executed")

func execute_dash():
    apply_dash_velocity()
    print("Buffered dash executed")

# State checks (examples)
func can_jump() -> bool:
    return is_on_floor() or has_air_jumps()

func can_attack() -> bool:
    return not $AnimationPlayer.is_playing() or can_cancel_into_attack()

func can_dash() -> bool:
    return dash_cooldown <= 0.0
```

**Advanced Multi-Priority Buffer:**
```gdscript
class_name PriorityInputBuffer

# Priority levels (higher = more important)
enum Priority { LOW = 0, NORMAL = 1, HIGH = 2, CRITICAL = 3 }

class BufferedInput:
    var action: String
    var timestamp: float
    var priority: int
    var consumed: bool = false
    var can_expire: bool = true

    func _init(p_action: String, p_time: float, p_priority: int = Priority.NORMAL):
        action = p_action
        timestamp = p_time
        priority = p_priority

var buffers: Dictionary = {}  # action -> Array of BufferedInput
var buffer_window: float = 0.15

func add_input(action: String, priority: int = Priority.NORMAL) -> void:
    var timestamp = Time.get_ticks_msec() / 1000.0

    if not buffers.has(action):
        buffers[action] = []

    var buffer_entry = BufferedInput.new(action, timestamp, priority)
    buffers[action].append(buffer_entry)

    # Sort by priority, then timestamp
    buffers[action].sort_custom(func(a, b):
        if a.priority != b.priority:
            return a.priority > b.priority  # Higher priority first
        return a.timestamp < b.timestamp  # Older first
    )

func consume(action: String, current_time: float) -> bool:
    if not buffers.has(action):
        return false

    for entry in buffers[action]:
        if not entry.consumed:
            if current_time - entry.timestamp <= buffer_window:
                entry.consumed = true
                return true

    return false

func clear_all() -> void:
    buffers.clear()

func clear_action(action: String) -> void:
    if buffers.has(action):
        buffers[action].clear()
```

**Key Parameters:**
- **Buffer window:** 0.10-0.20s (typical: 0.15s / 9 frames at 60fps)
  - Platformers: 0.15-0.20s (generous)
  - Fighting games: 0.10-0.13s (tighter)
  - Action games: 0.12-0.16s
  - Rhythm games: 0.05-0.08s (precise timing matters)
- **Max buffer size:** 5-15 entries (prevent spam)
- **Debounce window:** 0.03-0.05s (prevent duplicate entries)
- **Priority levels:** 2-4 tiers for complex systems

**Edge Cases:**
- **Conflicting inputs:** Priority system or FIFO order
- **State transition timing:** Check buffer immediately on state change
- **Animation canceling:** Some inputs should clear buffer, others preserve it
- **Cutscenes/dialogue:** Disable buffer during non-gameplay
- **Death/respawn:** Clear buffer on major state resets
- **Menu opening:** Preserve or clear based on menu type
- **Multiplayer:** Client-side buffer, server validation
- **Input spam:** Debounce or rate-limit buffer additions
- **Combo systems:** Separate combo buffer from action buffer

**When NOT to Use:**
- Rhythm games where timing precision is core mechanic
- Deliberate input challenges (e.g., specific timing puzzles)
- Turn-based games (no timing element)
- Menu navigation (direct input better)
- Quick-time events where timing is the test
- Frame-perfect speedrun tricks (unless accepted by community)

**Examples from Shipped Games:**

1. **Celeste (2018):** 0.15s input buffer for jump, dash, and grab. Stacks with coyote time for combined ~300ms input window. Developer Matt Thorson stated this made the difference between "frustrating" and "challenging but fair." Critical for complex tech like wave-dashing and ultra-dashing.

2. **Super Smash Bros. Ultimate (2018):** 0.10s buffer for all actions. Allows players to buffer next attack during current attack animation. Essential for competitive play; enables true combos and advanced techniques. Can be exploited for unintended strings.

3. **Hollow Knight (2017):** 0.13s buffer for attacks and spells. Allows spell buffering during nail swing recovery. Particularly important for boss fights where animation commitment is high. Enables aggressive play without punishing early inputs.

4. **Devil May Cry 5 (2019):** 0.12s buffer for combo actions. Allows queueing next attack before current finishes. Critical for stylish combos and maintaining SSS rank. Multiple buffer layers for different move categories.

5. **Dead Cells (2018):** 0.10s buffer for attacks and dodge rolls. Shorter buffer due to focus on reactive gameplay. Still present to prevent input drops during animation locks. Tuned to feel responsive without removing challenge.

**Platform Considerations:**
- **PC (keyboard/mouse):** 0.12-0.15s buffer, lower input latency allows tighter timing
- **Console (controller):** 0.15-0.18s to account for wireless latency (~5-15ms)
- **Mobile (touch):** 0.20-0.25s buffer to compensate for touchscreen latency (~50-100ms)
- **Online multiplayer:** Buffer client-side, validate server-side within latency tolerance
- **VR:** Shorter buffer (0.08-0.10s) due to immediate physical feedback

**Godot-Specific Notes:**
- Use `Input.is_action_just_pressed()` for frame-perfect capture
- `Time.get_ticks_msec()` provides millisecond precision timestamps
- Consider `@export` for buffer window to allow designer tuning
- Use typed arrays (`Array[InputBufferEntry]`) in Godot 4.x for performance
- Avoid allocations in hot path; pre-allocate buffer array
- `queue_free()` buffer on scene change to prevent memory leaks
- Profile with Godot Profiler; should be <0.05ms per frame

**Synergies:**
- **Coyote Time (#71):** Combined create massive input forgiveness. Jump buffer before landing + coyote after leaving ledge
- **Jump Buffering in Air (#78):** Specific application of input buffering for aerial actions
- **Attack Combo Buffering (#80):** Specialized buffer system for combo chains
- **Roll/Dodge Cancel Windows (#81):** Buffer dodge during attack recovery
- **Turn-Around Frames (#74):** Buffer attack during turn animation

**Measurement/Profiling:**
- **Success rate metrics:**
  - Without buffering: 60-75% action success rate
  - With buffering: 90-98% action success rate
- **Buffer usage statistics:**
  - Track how often buffered inputs execute vs. direct inputs
  - Typical: 15-30% of actions use buffer
- **Timing analysis:**
  - Log time delta between input press and execution
  - Plot distribution to find optimal buffer window
- **Playtesting:**
  - Ask testers to press buttons "slightly early"
  - Measure frustration incidents related to input timing

**Debug Tools:**
```gdscript
# Visual buffer debugger
func _draw():
    if OS.is_debug_build():
        var current_time = Time.get_ticks_msec() / 1000.0
        var y_offset = -50.0

        for entry in input_buffer:
            var age = current_time - entry.timestamp
            var percentage = 1.0 - (age / BUFFER_WINDOW)

            if percentage > 0.0 and not entry.consumed:
                var color = Color.GREEN if percentage > 0.5 else Color.YELLOW
                color.a = percentage

                draw_circle(Vector2(-20, y_offset), 4, color)
                draw_string(ThemeDB.fallback_font, Vector2(-10, y_offset + 4),
                    "%s: %.2f" % [entry.action, age],
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 10, color)

                y_offset -= 15.0

# Console logging
func debug_log_buffer():
    print("=== Input Buffer State ===")
    for entry in input_buffer:
        var status = "CONSUMED" if entry.consumed else "ACTIVE"
        print("  %s [%s]: %.3fs ago" % [entry.action, status,
            Time.get_ticks_msec() / 1000.0 - entry.timestamp])
```

**Performance Impact:**
- CPU: 0.01-0.05ms per frame (depends on buffer size)
- Memory: ~40 bytes per buffer entry × max size
- Allocation: Minimal if buffer size is pre-allocated
- Scaling: Safe for 10-50 buffered entities simultaneously
- GC pressure: None if using value types and fixed-size arrays
