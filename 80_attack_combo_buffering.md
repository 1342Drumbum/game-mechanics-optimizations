### 80. Attack Combo Buffering

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players pressing next attack button during current attack animation and having input ignored, breaking combo chains. Without buffering, combos require frame-perfect timing (1-3 frames at 60fps = 16-50ms windows), causing 40-60% of intended combo chains to drop. Results in frustration, perception of "unresponsive combat," and inability to execute advanced techniques. Particularly severe in action games where animation commitment is 300-800ms but input window is only 50-100ms.

**Technical Explanation:**
Combo buffering stores attack inputs pressed during active attack animations, then executes them in the appropriate cancel window. System tracks: (1) current attack state and frame, (2) buffered input queue with timestamps, (3) cancel windows (frames where next attack can start), (4) combo tree (valid transitions from current attack).

Implementation maintains state machine of attack animations. Each attack defines cancel windows: "attack_1 can cancel into attack_2 on frames 18-24, into dodge on frames 1-30." When input received, check if immediate execution valid; if not, buffer for later frames. On each animation frame, check buffers against cancel windows.

Key insight: Buffer doesn't just store "press attack," it stores specific attack type, timing, and directional modifiers. Fighting game implementation: buffer stores button sequence (LP, LP, HP = jab-jab-fierce combo).

**Algorithmic Complexity:**
- Buffer insertion: O(1)
- Cancel window check: O(k) where k = number of buffered inputs (typically 1-3)
- Combo tree traversal: O(d) where d = tree depth (typically 3-8 attacks)

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Combo system configuration
@export_group("Combo System")
@export var combo_buffer_window: float = 0.20  # 200ms / 12 frames
@export var max_buffered_inputs: int = 3
@export var enable_buffer_indicator: bool = true

# Attack definitions
enum AttackType { LIGHT, HEAVY, SPECIAL, LAUNCHER }

class AttackData:
    var type: AttackType
    var animation: String
    var damage: float
    var duration: float  # Total animation time
    var cancel_windows: Array[CancelWindow]  # When can cancel into next attack
    var can_buffer_during: bool = true

class CancelWindow:
    var start_frame: int
    var end_frame: int
    var allowed_attacks: Array[AttackType]  # What attacks can cancel into

# Combat state
var current_attack: AttackData = null
var attack_time_elapsed: float = 0.0
var combo_count: int = 0

# Input buffer
class BufferedInput:
    var attack_type: AttackType
    var timestamp: float
    var direction: Vector2  # For directional attacks
    var consumed: bool = false

var input_buffer: Array[BufferedInput] = []

# Combo definitions
var attack_database: Dictionary = {}

func _ready() -> void:
    setup_attack_database()

func setup_attack_database() -> void:
    # Define attack chains
    var light_attack = AttackData.new()
    light_attack.type = AttackType.LIGHT
    light_attack.animation = "light_attack"
    light_attack.damage = 10.0
    light_attack.duration = 0.4  # 400ms
    light_attack.cancel_windows = [
        create_cancel_window(12, 20, [AttackType.LIGHT, AttackType.HEAVY]),  # Can chain
        create_cancel_window(1, 24, [AttackType.SPECIAL])  # Can special cancel
    ]
    attack_database[AttackType.LIGHT] = light_attack

    # Heavy attack
    var heavy_attack = AttackData.new()
    heavy_attack.type = AttackType.HEAVY
    heavy_attack.animation = "heavy_attack"
    heavy_attack.damage = 25.0
    heavy_attack.duration = 0.6  # 600ms
    heavy_attack.cancel_windows = [
        create_cancel_window(15, 30, [AttackType.LAUNCHER]),  # Launch combo
        create_cancel_window(1, 35, [AttackType.SPECIAL])  # Special cancel
    ]
    attack_database[AttackType.HEAVY] = heavy_attack

    # Continue for other attack types...

func create_cancel_window(start: int, end: int, allowed: Array[AttackType]) -> CancelWindow:
    var window = CancelWindow.new()
    window.start_frame = start
    window.end_frame = end
    window.allowed_attacks = allowed
    return window

func _physics_process(delta: float) -> void:
    # Update attack animation
    if current_attack:
        attack_time_elapsed += delta
        var current_frame = int((attack_time_elapsed / current_attack.duration) * 60.0)  # Assume 60fps

        # Check if we can execute buffered attacks
        check_buffered_attacks(current_frame)

        # End attack if animation complete
        if attack_time_elapsed >= current_attack.duration:
            end_current_attack()

    # Handle new input
    if Input.is_action_just_pressed("attack_light"):
        attempt_attack(AttackType.LIGHT)

    if Input.is_action_just_pressed("attack_heavy"):
        attempt_attack(AttackType.HEAVY)

    if Input.is_action_just_pressed("attack_special"):
        attempt_attack(AttackType.SPECIAL)

    # Clean expired buffers
    clean_expired_buffers()

    move_and_slide()

func attempt_attack(attack_type: AttackType) -> void:
    var current_time = Time.get_ticks_msec() / 1000.0

    # If no current attack, execute immediately
    if not current_attack:
        execute_attack(attack_type)
        return

    # Check if can immediately cancel into this attack
    var current_frame = int((attack_time_elapsed / current_attack.duration) * 60.0)

    if can_cancel_into(current_attack, attack_type, current_frame):
        execute_attack(attack_type)
        return

    # Can't execute now, buffer it
    buffer_attack(attack_type, current_time)

func can_cancel_into(from_attack: AttackData, to_type: AttackType, current_frame: int) -> bool:
    for window in from_attack.cancel_windows:
        if current_frame >= window.start_frame and current_frame <= window.end_frame:
            if to_type in window.allowed_attacks:
                return true
    return false

func buffer_attack(attack_type: AttackType, timestamp: float) -> void:
    # Don't buffer if attack doesn't allow buffering
    if current_attack and not current_attack.can_buffer_during:
        return

    # Check buffer limit
    if input_buffer.size() >= max_buffered_inputs:
        # Remove oldest unbuffered input
        for i in range(input_buffer.size()):
            if not input_buffer[i].consumed:
                input_buffer.remove_at(i)
                break

    # Add to buffer
    var direction = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    var buffered = BufferedInput.new()
    buffered.attack_type = attack_type
    buffered.timestamp = timestamp
    buffered.direction = direction

    input_buffer.append(buffered)
    print("Buffered attack: %s" % AttackType.keys()[attack_type])

func check_buffered_attacks(current_frame: int) -> void:
    if input_buffer.is_empty():
        return

    # Check each buffered input against current cancel windows
    for buffered in input_buffer:
        if buffered.consumed:
            continue

        if can_cancel_into(current_attack, buffered.attack_type, current_frame):
            # Execute buffered attack
            execute_attack(buffered.attack_type, buffered.direction)
            buffered.consumed = true
            return  # Only execute one buffered attack

func execute_attack(attack_type: AttackType, direction: Vector2 = Vector2.ZERO) -> void:
    var attack_data = attack_database.get(attack_type)
    if not attack_data:
        return

    # End previous attack if any
    if current_attack:
        combo_count += 1
    else:
        combo_count = 1

    # Start new attack
    current_attack = attack_data
    attack_time_elapsed = 0.0

    # Play animation
    $AnimationPlayer.play(attack_data.animation)

    # Apply directional modifier if applicable
    if direction.length() > 0.5:
        # Face attack direction
        $Sprite2D.flip_h = direction.x < 0

    # Visual/audio feedback
    $AttackSound.play()
    spawn_attack_effects()

    print("Executed attack: %s (Combo: %d)" % [AttackType.keys()[attack_type], combo_count])

func end_current_attack() -> void:
    current_attack = null
    attack_time_elapsed = 0.0

    # Reset combo if no buffered follow-up
    if input_buffer.is_empty() or not has_valid_buffered_attack():
        reset_combo()

func has_valid_buffered_attack() -> bool:
    var current_time = Time.get_ticks_msec() / 1000.0
    for buffered in input_buffer:
        if not buffered.consumed:
            var age = current_time - buffered.timestamp
            if age <= combo_buffer_window:
                return true
    return false

func reset_combo() -> void:
    if combo_count > 1:
        print("Combo ended: %d hits" % combo_count)
        # Show combo counter UI
        show_combo_counter(combo_count)

    combo_count = 0

func clean_expired_buffers() -> void:
    var current_time = Time.get_ticks_msec() / 1000.0

    input_buffer = input_buffer.filter(func(buffered):
        if buffered.consumed:
            return false
        var age = current_time - buffered.timestamp
        return age <= combo_buffer_window
    )

func spawn_attack_effects():
    # Visual effects for attacks
    pass

func show_combo_counter(count: int):
    # UI feedback
    pass
```

**Advanced Combo Tree System:**
```gdscript
class_name ComboTreeSystem

# Combo tree node
class ComboNode:
    var attack_type: AttackType
    var inputs: Array[String]  # Button sequence to reach this attack
    var children: Array[ComboNode]  # Possible follow-ups
    var properties: Dictionary  # Damage, animation, etc.

    func can_transition_to(next_attack: AttackType) -> bool:
        for child in children:
            if child.attack_type == next_attack:
                return true
        return false

var combo_root: ComboNode
var current_combo_node: ComboNode

# Sequence buffering (for complex inputs like SF2 hadoken: ⬇️↘️➡️P)
class SequenceBuffer:
    var inputs: Array[String] = []
    var timestamps: Array[float] = []
    var max_sequence_length: int = 6
    var sequence_timeout: float = 0.5  # 500ms window for full sequence

func add_input_to_sequence(input: String) -> void:
    var current_time = Time.get_ticks_msec() / 1000.0

    # Add input
    sequence_buffer.inputs.append(input)
    sequence_buffer.timestamps.append(current_time)

    # Trim old inputs
    while sequence_buffer.inputs.size() > sequence_buffer.max_sequence_length:
        sequence_buffer.inputs.pop_front()
        sequence_buffer.timestamps.pop_front()

    # Check for matching combo sequences
    check_combo_sequences()

func check_combo_sequences() -> bool:
    # Check if current sequence matches any combo
    var sequence_str = ",".join(sequence_buffer.inputs)

    # Example: "down,down-right,right,punch" = hadoken
    var combo_patterns = {
        "down,down-right,right,punch": "fireball",
        "right,down,down-right,punch": "dragon_punch",
        "punch,punch,right,punch": "triple_jab_special"
    }

    for pattern in combo_patterns:
        if sequence_str.ends_with(pattern):
            execute_special_attack(combo_patterns[pattern])
            sequence_buffer.inputs.clear()
            sequence_buffer.timestamps.clear()
            return true

    return false

# Timing-based combo system (rhythm game style)
class TimingWindow:
    var perfect_time: float  # Exact ideal timing
    var good_range: float = 0.05  # ±50ms for "good"
    var ok_range: float = 0.10  # ±100ms for "ok"

    func evaluate_timing(actual_time: float) -> String:
        var diff = abs(actual_time - perfect_time)

        if diff <= good_range:
            return "perfect"
        elif diff <= ok_range:
            return "good"
        else:
            return "miss"

var combo_timing_windows: Array[TimingWindow] = []
var current_timing_index: int = 0

func check_timing_based_combo() -> void:
    if current_timing_index >= combo_timing_windows.size():
        return

    var window = combo_timing_windows[current_timing_index]
    var timing_result = window.evaluate_timing(attack_time_elapsed)

    match timing_result:
        "perfect":
            apply_damage_multiplier(1.5)  # Bonus damage for perfect timing
        "good":
            apply_damage_multiplier(1.2)
        "miss":
            break_combo()  # Timing too far off

    current_timing_index += 1
```

**Key Parameters:**
- **Buffer window:** 0.15-0.25s (typical: 0.20s / 12 frames)
  - Fast combat: 0.12-0.18s
  - Slower combat: 0.20-0.30s
  - Fighting games: 0.10-0.15s (frame-perfect emphasis)
- **Max buffered inputs:** 1-3 (prevent spam)
- **Cancel window size:** 4-12 frames per attack
- **Sequence timeout:** 0.3-0.8s for complex inputs

**Edge Cases:**
- **Interrupted attacks:** Clear buffer on damage/knockback
- **Animation blending:** Ensure buffer works across blend transitions
- **Weapon switching:** Reset combo tree on weapon change
- **Multiple enemies:** Buffer should work regardless of target
- **Mashed inputs:** Debounce to prevent accidental multi-buffers
- **Priority conflicts:** Heavy attack buffered + light pressed = take most recent
- **Death during combo:** Clear buffers on death/respawn
- **Cutscene triggers:** Disable buffering during non-combat
- **Stamina system:** Buffer only if stamina available for next attack

**When NOT to Use:**
- Turn-based combat
- Gun-based combat (unless melee combos)
- Games where deliberate pacing is core (Dark Souls stamina management)
- When animation commitment is intentional difficulty
- Stealth games (one-hit kills, no combos)
- When button mashing is discouraged

**Examples from Shipped Games:**

1. **Devil May Cry 5 (2019):** 0.12s buffer for all attack transitions. Essential for SSS-rank combos. Multiple buffer layers: attack buffer, style buffer, weapon switch buffer. Can buffer next attack up to 7 frames before cancel window. Allows players to focus on style rather than frame-perfect timing.

2. **God of War (2018):** 0.20s buffer for light/heavy attack chains. Longer buffer (0.25s) for runic attacks during regular attacks. Buffer persists through dodge rolls—can buffer attack, dodge, then attack executes. Makes combat feel responsive despite animation commitment.

3. **Bayonetta 2 (2014):** Frame-perfect combos become accessible through 0.15s buffer. "Dodge offset" technique relies on buffering attacks during dodge animation. Buffer allows queuing entire combo sequence ahead of time. Competitive players still aim for frame-perfect for optimal DPS.

4. **Monster Hunter: World (2018):** Intentionally minimal buffering (0.08s) to preserve deliberate combat. Heavy weapons have longer commitment with less buffer forgiveness. Design choice: animation commitment is core difficulty mechanic. Shows when NOT to use generous buffering.

5. **Hades (2020):** 0.18s buffer for all attacks and dashes. Can buffer next attack during current attack or dash. Critical for fast-paced combat with 60fps requirement. Buffer allows aggressive play style without constant input drops.

**Platform Considerations:**
- **PC (keyboard):** Precise inputs, can use tighter buffer (0.12-0.15s)
- **Console (controller):** Standard buffer (0.15-0.20s)
- **Mobile (touch):** Longer buffer (0.20-0.30s) for touchscreen latency
- **60fps vs 30fps:** Same ms window = different frame counts
- **Network games:** Client-side buffer, server validates combo validity

**Godot-Specific Notes:**
- `AnimationPlayer.current_animation_position` for frame tracking
- `AnimationPlayer.animation_finished` signal for combo reset detection
- Use `@onready var` for attack database to avoid null references
- Consider `AnimationTree` for complex combo animation blending
- `Input.is_action_just_pressed()` for precise input capture
- Export buffer window for per-character tuning
- Profile: Buffer system should add <0.05ms per frame

**Synergies:**
- **Input Buffering (#72):** Combo buffer is specialized application
- **Roll/Dodge Cancel (#81):** Dodge can cancel attacks, buffer next action
- **Animation Canceling:** Define which attacks can cancel into which
- **Hitstun System:** Buffer during hitstun for combo extensions
- **Camera Shake:** Enhanced feedback for successful combo chains
- **UI Combo Counter:** Visual feedback for successful buffering

**Measurement/Profiling:**
- **Combo completion rate:**
  - Without buffer: 40-60% of intended combos succeed
  - With buffer: 85-95% succeed
- **Buffer usage statistics:**
  - % of attacks that used buffer vs. direct input
  - Typical: 30-50% of combo attacks buffered
- **Timing distribution:**
  - Plot histogram of input timing relative to cancel windows
  - Shows how early players press buttons
- **Combo length:**
  - Average combo length with/without buffer
  - Should increase 30-50% with buffer
- **Player satisfaction:**
  - "Do combos feel responsive?" (target: >85% yes)

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    # Draw current attack timeline
    if current_attack:
        var timeline_width = 200.0
        var timeline_pos = Vector2(10, -100)
        var progress = attack_time_elapsed / current_attack.duration

        # Timeline bar
        draw_rect(Rect2(timeline_pos, Vector2(timeline_width, 10)),
                  Color(0.3, 0.3, 0.3))
        draw_rect(Rect2(timeline_pos, Vector2(timeline_width * progress, 10)),
                  Color.GREEN)

        # Cancel windows
        var frame_count = 60  # Assume 60fps
        for window in current_attack.cancel_windows:
            var start_x = timeline_pos.x + (float(window.start_frame) / frame_count) * timeline_width
            var end_x = timeline_pos.x + (float(window.end_frame) / frame_count) * timeline_width

            draw_rect(Rect2(Vector2(start_x, timeline_pos.y - 5),
                           Vector2(end_x - start_x, 20)),
                     Color(0, 1, 1, 0.3))

    # Draw buffered inputs
    var y_offset = -120.0
    for i in range(input_buffer.size()):
        var buffered = input_buffer[i]
        var current_time = Time.get_ticks_msec() / 1000.0
        var age = current_time - buffered.timestamp
        var remaining = combo_buffer_window - age
        var alpha = remaining / combo_buffer_window

        var color = Color.YELLOW if not buffered.consumed else Color.GRAY
        color.a = alpha

        draw_string(ThemeDB.fallback_font, Vector2(10, y_offset),
                    "Buffered: %s (%.2fs)" % [AttackType.keys()[buffered.attack_type], remaining],
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 12, color)
        y_offset -= 15

    # Combo counter
    if combo_count > 0:
        draw_string(ThemeDB.fallback_font, Vector2(10, -140),
                    "COMBO: %d" % combo_count,
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 20, Color.YELLOW)

# Statistics
var combo_stats = {
    "total_attacks": 0,
    "buffered_attacks": 0,
    "max_combo": 0,
    "combos_completed": 0,
}

func log_attack(was_buffered: bool):
    combo_stats.total_attacks += 1
    if was_buffered:
        combo_stats.buffered_attacks += 1

    combo_stats.max_combo = max(combo_stats.max_combo, combo_count)

    if combo_stats.total_attacks % 50 == 0:
        print_combo_stats()

func print_combo_stats():
    var buffer_rate = 100.0 * combo_stats.buffered_attacks / combo_stats.total_attacks

    print("=== Combo Statistics ===")
    print("  Total attacks: %d" % combo_stats.total_attacks)
    print("  Buffered: %.1f%%" % buffer_rate)
    print("  Max combo: %d" % combo_stats.max_combo)
```

**Performance Impact:**
- CPU: 0.02-0.08ms per frame (depends on buffer size and combo complexity)
- Memory: ~200 bytes per buffered input
- Combo tree traversal: O(d) but d typically ≤8
- Safe for 20-30 combatants with active buffers
- GC pressure: Minimal if using object pools for buffer entries
