### 81. Roll/Dodge Cancel Windows

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players unable to dodge out of attack animations, leading to unavoidable damage and frustration. Without cancel windows, attacks lock player into 300-800ms animations with no escape option, causing 40-60% of deaths in action games to feel "unfair." Creates binary choice: commit to full attack or never attack. Results in overly defensive play, reduced combat enjoyment, and perception of "clunky" controls.

**Technical Explanation:**
Dodge cancel windows allow interrupting attack/ability animations with defensive actions at specific timing points. System defines per-animation: (1) cancel window start/end frames, (2) allowed cancel actions (dodge, block, parry), (3) cost modifiers (stamina penalty for early cancel), (4) animation blending rules.

Implementation uses animation frame tracking plus state machine. Each attack animation defines cancel windows: "heavy_attack can cancel to dodge on frames 1-15 (early, 150% stamina) or 25-40 (late, 100% stamina)." On dodge input, check current animation frame against windows. If valid, interrupt animation, apply costs, transition to dodge state.

Key design balance: Early cancel windows preserve responsiveness but reduce animation commitment (less tactical). Late-only windows increase skill ceiling. Best implementations use graduated costs—can cancel anytime but early cancels cost more resources.

**Algorithmic Complexity:**
- Cancel check: O(k) where k = number of cancel windows (typically 1-3 per animation)
- Animation interruption: O(1) state transition

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

# Cancel window configuration
@export_group("Cancel Windows")
@export var enable_cancel_windows: bool = true
@export var early_cancel_stamina_mult: float = 1.5  # 50% more stamina for early cancel
@export var enable_cancel_buffering: bool = true

# Animation state
var current_animation: String = ""
var animation_frame: int = 0
var can_cancel_current: bool = false

# Cancel window definitions
class CancelWindow:
    var start_frame: int
    var end_frame: int
    var allowed_actions: Array[String]  # ["dodge", "block", "jump"]
    var stamina_cost_mult: float = 1.0
    var priority: int = 1  # Higher = overrides other windows

class AnimationCancelData:
    var animation_name: String
    var total_frames: int
    var cancel_windows: Array[CancelWindow] = []
    var default_cancellable: bool = false  # Can cancel anytime if true

var animation_cancel_db: Dictionary = {}

# Dodge state
const DODGE_STAMINA_COST = 20.0
const DODGE_DURATION = 0.3
const DODGE_SPEED = 400.0

var is_dodging: bool = false
var dodge_time_remaining: float = 0.0
var stamina: float = 100.0
var max_stamina: float = 100.0

func _ready() -> void:
    setup_cancel_database()
    $AnimationPlayer.animation_changed.connect(_on_animation_changed)

func setup_cancel_database() -> void:
    # Light attack - generous cancel windows
    var light_attack = AnimationCancelData.new()
    light_attack.animation_name = "light_attack"
    light_attack.total_frames = 24  # 400ms at 60fps

    # Early cancel (expensive)
    var early_window = CancelWindow.new()
    early_window.start_frame = 1
    early_window.end_frame = 12
    early_window.allowed_actions = ["dodge", "block"]
    early_window.stamina_cost_mult = 1.5
    early_window.priority = 1

    # Late cancel (normal cost)
    var late_window = CancelWindow.new()
    late_window.start_frame = 18
    late_window.end_frame = 24
    late_window.allowed_actions = ["dodge", "block", "jump"]
    late_window.stamina_cost_mult = 1.0
    late_window.priority = 2

    light_attack.cancel_windows = [early_window, late_window]
    animation_cancel_db["light_attack"] = light_attack

    # Heavy attack - restricted cancel windows
    var heavy_attack = AnimationCancelData.new()
    heavy_attack.animation_name = "heavy_attack"
    heavy_attack.total_frames = 48  # 800ms at 60fps

    # No early cancel (commitment required)
    var heavy_late = CancelWindow.new()
    heavy_late.start_frame = 35
    heavy_late.end_frame = 48
    heavy_late.allowed_actions = ["dodge"]
    heavy_late.stamina_cost_mult = 1.2
    heavy_late.priority = 1

    heavy_attack.cancel_windows = [heavy_late]
    animation_cancel_db["heavy_attack"] = heavy_attack

    # Dodge - can cancel into another dodge (chain dodging)
    var dodge = AnimationCancelData.new()
    dodge.animation_name = "dodge"
    dodge.total_frames = 18  # 300ms

    var dodge_chain = CancelWindow.new()
    dodge_chain.start_frame = 12
    dodge_chain.end_frame = 18
    dodge_chain.allowed_actions = ["dodge"]
    dodge_chain.stamina_cost_mult = 1.8  # Expensive to chain
    dodge_chain.priority = 1

    dodge.cancel_windows = [dodge_chain]
    animation_cancel_db["dodge"] = dodge

func _physics_process(delta: float) -> void:
    # Update animation frame
    update_animation_frame()

    # Handle dodge input
    if Input.is_action_just_pressed("dodge"):
        attempt_dodge()

    # Update dodge state
    if is_dodging:
        update_dodge(delta)

    # Regenerate stamina
    if stamina < max_stamina and not is_dodging:
        stamina = min(stamina + 20.0 * delta, max_stamina)

    move_and_slide()

func update_animation_frame() -> void:
    if not $AnimationPlayer.is_playing():
        return

    var animation = $AnimationPlayer.current_animation
    var position = $AnimationPlayer.current_animation_position
    var length = $AnimationPlayer.current_animation_length

    # Calculate current frame (assume 60fps animations)
    animation_frame = int((position / length) * 60.0)
    current_animation = animation

func attempt_dodge() -> void:
    # Check if can cancel current animation
    if current_animation != "" and current_animation != "idle":
        if not can_cancel_to_action("dodge"):
            # Buffer dodge input
            if enable_cancel_buffering:
                buffer_cancel_action("dodge")
            return

    # Execute dodge if we have stamina
    var cost = calculate_dodge_cost()
    if stamina >= cost:
        execute_dodge(cost)

func can_cancel_to_action(action: String) -> bool:
    if not enable_cancel_windows:
        return true  # No restrictions if system disabled

    if not animation_cancel_db.has(current_animation):
        return true  # Unknown animations are freely cancellable

    var cancel_data: AnimationCancelData = animation_cancel_db[current_animation]

    # Check if always cancellable
    if cancel_data.default_cancellable:
        return true

    # Check cancel windows
    for window in cancel_data.cancel_windows:
        if animation_frame >= window.start_frame and animation_frame <= window.end_frame:
            if action in window.allowed_actions:
                return true

    return false

func calculate_dodge_cost() -> float:
    var base_cost = DODGE_STAMINA_COST

    # Check if canceling from another animation
    if current_animation != "" and current_animation != "idle":
        var multiplier = get_cancel_cost_multiplier("dodge")
        return base_cost * multiplier

    return base_cost

func get_cancel_cost_multiplier(action: String) -> float:
    if not animation_cancel_db.has(current_animation):
        return 1.0

    var cancel_data: AnimationCancelData = animation_cancel_db[current_animation]

    # Find applicable window
    var highest_priority = -1
    var multiplier = 1.0

    for window in cancel_data.cancel_windows:
        if animation_frame >= window.start_frame and animation_frame <= window.end_frame:
            if action in window.allowed_actions:
                if window.priority > highest_priority:
                    highest_priority = window.priority
                    multiplier = window.stamina_cost_mult

    return multiplier

func execute_dodge(stamina_cost: float) -> void:
    # Deduct stamina
    stamina -= stamina_cost

    # Cancel current animation
    if $AnimationPlayer.is_playing():
        $AnimationPlayer.stop()

    # Start dodge
    is_dodging = true
    dodge_time_remaining = DODGE_DURATION

    # Play dodge animation
    $AnimationPlayer.play("dodge")

    # Get dodge direction
    var input_dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    if input_dir.length() > 0.1:
        velocity = input_dir.normalized() * DODGE_SPEED
    else:
        # Default dodge away from facing direction
        velocity = Vector2(-sign($Sprite2D.scale.x), 0) * DODGE_SPEED

    # Visual feedback
    $DodgeParticles.restart()
    $DodgeSound.play()

    # Invincibility frames
    set_collision_layer_value(1, false)

    print("Dodge executed (cost: %.0f stamina)" % stamina_cost)

func update_dodge(delta: float) -> void:
    dodge_time_remaining -= delta

    if dodge_time_remaining <= 0:
        end_dodge()

func end_dodge() -> void:
    is_dodging = false

    # Restore collision
    set_collision_layer_value(1, true)

    # Reduce velocity
    velocity *= 0.3

func _on_animation_changed(old_name: String, new_name: String) -> void:
    animation_frame = 0
    current_animation = new_name

# Cancel buffering
var buffered_cancel_action: String = ""
var buffered_cancel_time: float = -1.0
const CANCEL_BUFFER_WINDOW = 0.15

func buffer_cancel_action(action: String) -> void:
    buffered_cancel_action = action
    buffered_cancel_time = Time.get_ticks_msec() / 1000.0
    print("Cancel action buffered: %s" % action)

func check_buffered_cancel() -> void:
    if buffered_cancel_action == "":
        return

    var current_time = Time.get_ticks_msec() / 1000.0
    var age = current_time - buffered_cancel_time

    if age > CANCEL_BUFFER_WINDOW:
        buffered_cancel_action = ""
        return

    # Try to execute buffered cancel
    if can_cancel_to_action(buffered_cancel_action):
        match buffered_cancel_action:
            "dodge":
                var cost = calculate_dodge_cost()
                if stamina >= cost:
                    execute_dodge(cost)
                    buffered_cancel_action = ""
```

**Advanced Cancel System:**
```gdscript
class_name AdvancedCancelSystem

# Cancel types
enum CancelType {
    NONE,          # Cannot cancel
    SOFT,          # Can cancel but lose attack benefits
    HARD,          # Can always cancel
    SPECIAL,       # Can cancel only with specific inputs
    CONDITIONAL    # Can cancel based on game state
}

class ConditionalCancel:
    var condition_type: String  # "hit_enemy", "perfect_timing", "combo_active"
    var condition_met: Callable
    var bonus: Dictionary  # Rewards for conditional cancel

# Parry/Perfect dodge system
const PARRY_WINDOW_FRAMES = 6  # Tight 100ms window
var last_dodge_frame: int = -1

func check_parry_timing(current_frame: int) -> bool:
    # Perfect dodge if dodged within 6 frames of attack
    if current_frame - last_dodge_frame <= PARRY_WINDOW_FRAMES:
        return true
    return false

func execute_perfect_dodge():
    # Rewards for perfect timing
    stamina = min(stamina + 15.0, max_stamina)  # Stamina refund
    slow_time(0.3, 0.5)  # Brief slow-motion
    enable_counter_attack_window()

# Resource system integration
class CancelCost:
    var stamina: float = 0.0
    var mana: float = 0.0
    var cooldown: float = 0.0
    var health: float = 0.0  # Life-for-dodge mechanics

func check_can_afford_cancel(cost: CancelCost) -> bool:
    if stamina < cost.stamina:
        return false
    if mana < cost.mana:
        return false
    # etc...
    return true

# Animation blending for smooth cancels
func execute_cancel_with_blend(from_anim: String, to_anim: String, blend_time: float = 0.1):
    # Use AnimationTree for smooth transitions
    var tree: AnimationTree = $AnimationTree
    tree.set("parameters/cancel_blend/blend_amount", 0.0)

    # Tween blend over time
    var tween = create_tween()
    tween.tween_property(tree, "parameters/cancel_blend/blend_amount", 1.0, blend_time)

# Hit-confirm cancels (fighting game mechanic)
var last_attack_hit: bool = false

func on_attack_hit():
    last_attack_hit = true
    # Enable special cancel windows only on hit
    enable_hit_confirm_cancels()

func enable_hit_confirm_cancels():
    # Can cancel into special moves only if attack connected
    var current_cancel = animation_cancel_db.get(current_animation)
    if current_cancel:
        for window in current_cancel.cancel_windows:
            if "special" in window.allowed_actions:
                window.stamina_cost_mult = 0.5  # Reduced cost on hit
```

**Key Parameters:**
- **Cancel window size:** 4-15 frames (67-250ms at 60fps)
  - Early windows: 4-8 frames (tight)
  - Late windows: 8-15 frames (generous)
- **Stamina cost multipliers:**
  - Early cancel: 1.3-1.8x (punish panic cancels)
  - Late cancel: 1.0-1.2x (normal)
  - Perfect timing: 0.5-0.8x (reward skill)
- **Cancel buffer window:** 0.10-0.15s
- **Invincibility frames:** 6-12 frames (100-200ms)

**Edge Cases:**
- **Cancel into same action:** Prevent spam, apply cooldown
- **Multi-hit attacks:** Each hit may have separate cancel windows
- **Charged attacks:** Hold-to-charge may not be cancellable until released
- **Aerial attacks:** Different cancel rules than grounded
- **Knockback/hitstun:** Cannot cancel during stun
- **Out of stamina:** Cannot cancel or default to slow roll
- **Boss attacks:** Some enemy attacks may have un-dodgeable properties
- **Environmental hazards:** Cancel rules during special states (burning, frozen)
- **Weapon-specific:** Heavy weapons = restricted cancels, light weapons = generous

**When NOT to Use:**
- Turn-based combat
- When animation commitment is core difficulty (Dark Souls design philosophy)
- Puzzle-based combat (timing is the puzzle)
- Stealth games (no combat focus)
- When players should plan attacks carefully, not react
- Horror games (vulnerability creates tension)

**Examples from Shipped Games:**

1. **Devil May Cry 5 (2019):** Can cancel almost any attack with dodge or jump. Early cancels cost no resource but lose combo potential. "Jump cancel" technique allows infinite combos. Design philosophy: style over restriction. Generous windows (10-20 frames) for accessibility.

2. **Dark Souls III (2016):** Intentionally restrictive. Can only cancel during specific recovery frames (last 20-30% of animation). Roll costs stamina, early cancels cost 50% more. Creates tactical combat—must commit to attacks. Shows when NOT to use generous cancels.

3. **Bayonetta 2 (2014):** "Dodge offset" mechanic: Can cancel any attack with dodge, then resume combo afterward. Perfect dodge (parry window: 6 frames) triggers Witch Time. Graduates difficulty: casual players can dodge anytime, pros master parry timing.

4. **Monster Hunter: World (2018):** Weapon-dependent. Great Sword: very limited cancels (animation commitment). Dual Blades: generous cancels. Balance: heavy weapons = high damage + commitment, light weapons = low damage + mobility. Cancel windows tuned per weapon class.

5. **Sekiro: Shadows Die Twice (2019):** Can cancel attacks into block/parry at any time. Parry window: 10 frames. Design emphasis on reaction, not commitment. Different philosophy than Dark Souls despite same developer. Shows cancels can work in "difficult" games.

**Platform Considerations:**
- **PC (60+ fps):** Frame counts match design exactly
- **Console (30fps):** Half the frame count for same timing
- **Input lag:** Add 1-2 frames to cancel windows per 16ms of lag
- **Network games:** Client-side cancel, server validates timing
- **Accessibility:** Option to extend cancel windows by 50-100%

**Godot-Specific Notes:**
- `AnimationPlayer.current_animation_position` for frame tracking
- `AnimationPlayer.stop()` to interrupt animations
- `AnimationTree` for smooth cancel blending
- `Tween` for stamina deduction animations
- Export cancel window sizes for per-animation tuning
- Profile: Cancel checks should be <0.02ms per frame

**Synergies:**
- **Input Buffering (#72):** Buffer cancel action during non-cancelable frames
- **Attack Combo Buffering (#80):** Can buffer next attack after cancel
- **Invincibility Frames:** Dodges typically include i-frames
- **Stamina System:** Balances cancel spam
- **Perfect Timing Rewards:** Skill-based bonuses for frame-perfect cancels
- **Animation Blending:** Smooth transitions on cancel

**Measurement/Profiling:**
- **Cancel usage rate:**
  - % of attacks that were canceled
  - Typical: 20-40% in action games
- **Cancel timing distribution:**
  - Plot histogram of cancel frame timing
  - Identifies if players use early vs. late windows
- **Death analysis:**
  - % of deaths where player tried to cancel but couldn't
  - Should be <10% (indicates windows are adequate)
- **Stamina consumption:**
  - Track average stamina spent on cancels
  - Balance cost to prevent spam
- **Perfect dodge rate:**
  - % of dodges in parry window
  - Typical: 5-15% (skill-based)

**Debug Visualization:**
```gdscript
func _draw():
    if not OS.is_debug_build():
        return

    # Draw cancel window timeline
    if current_animation != "" and animation_cancel_db.has(current_animation):
        var cancel_data: AnimationCancelData = animation_cancel_db[current_animation]
        var timeline_width = 200.0
        var timeline_pos = Vector2(10, -80)

        # Timeline background
        draw_rect(Rect2(timeline_pos, Vector2(timeline_width, 15)),
                  Color(0.2, 0.2, 0.2))

        # Current frame indicator
        var frame_progress = float(animation_frame) / float(cancel_data.total_frames)
        draw_rect(Rect2(timeline_pos, Vector2(timeline_width * frame_progress, 15)),
                  Color.GREEN)

        # Cancel windows
        for window in cancel_data.cancel_windows:
            var start_x = timeline_pos.x + (float(window.start_frame) / cancel_data.total_frames) * timeline_width
            var end_x = timeline_pos.x + (float(window.end_frame) / cancel_data.total_frames) * timeline_width

            var color = Color.CYAN
            if window.stamina_cost_mult > 1.2:
                color = Color.YELLOW  # Expensive cancel
            elif window.stamina_cost_mult < 1.0:
                color = Color.GREEN  # Cheap cancel

            draw_rect(Rect2(Vector2(start_x, timeline_pos.y - 5),
                           Vector2(end_x - start_x, 25)),
                     Color(color.r, color.g, color.b, 0.3), false, 2.0)

        # Frame counter
        draw_string(ThemeDB.fallback_font, timeline_pos + Vector2(0, -10),
                    "Frame: %d/%d" % [animation_frame, cancel_data.total_frames],
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 10, Color.WHITE)

    # Stamina bar
    var stamina_bar_pos = Vector2(10, -100)
    var stamina_bar_width = 100.0
    draw_rect(Rect2(stamina_bar_pos, Vector2(stamina_bar_width, 8)),
              Color(0.2, 0.2, 0.2))
    draw_rect(Rect2(stamina_bar_pos,
                   Vector2(stamina_bar_width * (stamina / max_stamina), 8)),
              Color.GREEN)

    # Buffered cancel indicator
    if buffered_cancel_action != "":
        var buffer_age = Time.get_ticks_msec() / 1000.0 - buffered_cancel_time
        var remaining = CANCEL_BUFFER_WINDOW - buffer_age
        var alpha = remaining / CANCEL_BUFFER_WINDOW

        draw_string(ThemeDB.fallback_font, Vector2(10, -110),
                    "Buffered: %s (%.2fs)" % [buffered_cancel_action, remaining],
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 12, Color(1, 1, 0, alpha))

# Statistics
var cancel_stats = {
    "total_attacks": 0,
    "canceled_attacks": 0,
    "early_cancels": 0,
    "late_cancels": 0,
    "perfect_cancels": 0,
}

func log_attack_cancel(was_canceled: bool, window_type: String = ""):
    cancel_stats.total_attacks += 1

    if was_canceled:
        cancel_stats.canceled_attacks += 1

        match window_type:
            "early":
                cancel_stats.early_cancels += 1
            "late":
                cancel_stats.late_cancels += 1
            "perfect":
                cancel_stats.perfect_cancels += 1

    if cancel_stats.total_attacks % 30 == 0:
        print_cancel_stats()

func print_cancel_stats():
    var cancel_rate = 100.0 * cancel_stats.canceled_attacks / cancel_stats.total_attacks

    print("=== Cancel Statistics ===")
    print("  Total attacks: %d" % cancel_stats.total_attacks)
    print("  Cancel rate: %.1f%%" % cancel_rate)
    print("  Early: %d, Late: %d, Perfect: %d" % [
        cancel_stats.early_cancels,
        cancel_stats.late_cancels,
        cancel_stats.perfect_cancels
    ])
```

**Performance Impact:**
- CPU: 0.01-0.03ms per frame (frame tracking + window checks)
- Memory: ~150 bytes per animation cancel data
- Cancel check: O(k) where k = 1-3 windows typically
- Safe for 30+ entities with cancel-capable animations
- GC pressure: None if cancel data is static
