### 71. Coyote Time (Jump Grace Period)

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Players pressing jump just after walking off a ledge and feeling cheated when the jump doesn't register. Without coyote time, precise timing is required (1-2 frames at 60fps = 16-33ms window), causing 15-30% of intended jumps to fail. This creates player frustration, perception of "broken controls," and reduced flow state.

**Technical Explanation:**
Coyote time allows the player to jump for a brief period after leaving solid ground (typically 100-150ms or 6-9 frames at 60fps). The system tracks time since last grounded state. When jump is pressed, check if `time_since_grounded < coyote_threshold`. If true, execute jump even though player is technically airborne. Named after Wile E. Coyote who doesn't fall until he looks down. Exploits human reaction time delay (~200ms) and compensates for visual-motor coordination lag.

The technique works because players visually perceive themselves as still on the ledge when they've already moved 1-2 pixels past it. The brain initiates the jump command based on visual feedback that's already outdated. Coyote time bridges this perception-reality gap.

**Algorithmic Complexity:** O(1) - simple timestamp comparison per jump input

**Implementation Pattern:**
```gdscript
# Godot implementation
extends CharacterBody2D

const COYOTE_TIME = 0.12  # 120ms / ~7 frames at 60fps
const JUMP_VELOCITY = -400.0

var time_since_grounded: float = 0.0
var is_grounded: bool = false
var last_grounded_time: float = 0.0
var coyote_used: bool = false

func _physics_process(delta: float) -> void:
    # Check if grounded (Godot's built-in ground detection)
    var was_grounded = is_grounded
    is_grounded = is_on_floor()

    # Update grounded timer
    if is_grounded:
        time_since_grounded = 0.0
        last_grounded_time = Time.get_ticks_msec() / 1000.0
        coyote_used = false
    else:
        time_since_grounded += delta

    # Handle jump input
    if Input.is_action_just_pressed("jump"):
        if can_coyote_jump():
            execute_jump()

func can_coyote_jump() -> bool:
    # Can jump if grounded OR within coyote time window
    if is_grounded:
        return true

    # Coyote time check
    if time_since_grounded <= COYOTE_TIME and not coyote_used:
        coyote_used = true  # Prevent double-jumps from coyote
        return true

    return false

func execute_jump():
    velocity.y = JUMP_VELOCITY
    # Optional: Visual/audio feedback
    $JumpParticles.emit()
    $JumpSound.play()

# Visual debug helper
func _draw():
    if not Engine.is_editor_hint():
        return

    # Draw coyote time indicator
    if time_since_grounded <= COYOTE_TIME and not is_grounded:
        var percentage = 1.0 - (time_since_grounded / COYOTE_TIME)
        draw_circle(Vector2(0, 20), 5, Color(1, 1, 0, percentage))
```

**Advanced Implementation with State Tracking:**
```gdscript
class_name CoyoteTimeTracker

var coyote_duration: float = 0.12
var time_left: float = 0.0
var is_active: bool = false
var was_consumed: bool = false

func start() -> void:
    time_left = coyote_duration
    is_active = true
    was_consumed = false

func update(delta: float) -> void:
    if is_active:
        time_left -= delta
        if time_left <= 0.0:
            is_active = false

func can_use() -> bool:
    return is_active and not was_consumed

func consume() -> bool:
    if can_use():
        was_consumed = true
        is_active = false
        return true
    return false

func reset() -> void:
    is_active = false
    was_consumed = false
    time_left = 0.0
```

**Key Parameters:**
- Coyote duration: 0.08-0.15s (typical: 0.12s / 7 frames)
- Platformers: 0.10-0.15s (longer for casual games)
- Precision platformers: 0.08-0.12s
- Action games: 0.06-0.10s (tighter timing)
- Mobile: 0.15-0.20s (compensate for touch latency)

**Edge Cases:**
- **Multiple platforms:** Reset timer on any ground contact, not just initial landing
- **Moving platforms:** Track platform velocity; maintain coyote even if platform moves away
- **Slopes:** Consider slope normals; coyote should work on any walkable angle
- **Wall jumps:** Separate wall-coyote timer (usually shorter: 0.05-0.08s)
- **Double jump interaction:** Coyote jump doesn't consume double-jump charge
- **Ledge grab:** If player has ledge grab, coyote shouldn't interfere
- **One-way platforms:** Coyote should not activate when jumping down through
- **Damage knockback:** Reset coyote timer on hit to prevent exploitation

**When NOT to Use:**
- Deliberate gap-crossing challenges (designer wants precise timing)
- Physics puzzles requiring exact ledge timing
- Speedrunning tech that relies on frame-perfect inputs (unless community accepts it)
- Games with floaty jumping where it feels excessive
- Climbing/ladder systems (different input model)
- Swimming/flying states (no ground reference)

**Examples from Shipped Games:**

1. **Celeste (2018):** 0.15s coyote time, one of the most generous implementations. Combined with input buffering, creates extremely responsive feel. Developer commentary confirms this was critical to accessibility and feel. Players can consistently execute difficult jumps without frustration.

2. **Super Meat Boy (2010):** ~0.10s coyote time. Essential for wall-jump sequences. Edmund McMillen stated "the game feels broken without it" in post-mortems. Allows players to focus on strategy rather than fighting controls.

3. **Hollow Knight (2017):** 0.12s coyote time, works on floors, walls, and ceilings. Critical for maintaining flow during combat platforming. Especially important for Mantis Claw (wall jump) sections.

4. **Dead Cells (2018):** 0.08s coyote time, shorter than platformers due to combat focus. Still present to prevent frustration during vertical traversal in combat arenas.

5. **Ori and the Blind Forest (2015):** 0.13s coyote time, tuned for fluid movement. Works seamlessly with Bash and other movement abilities. Essential for maintaining momentum through complex environments.

**Platform Considerations:**
- **PC (60+ fps):** 0.10-0.12s is standard, smooth frame pacing makes it feel natural
- **Console (30-60fps):** Slightly longer (0.12-0.15s) to compensate for frame delay
- **Mobile (touch):** 0.15-0.20s to account for ~50-100ms touch latency
- **VR:** Shorter (0.08-0.10s) due to increased motion sensitivity
- **Input latency:** Add 1-2 frames worth of coyote time per 16ms of system latency

**Godot-Specific Notes:**
- Use `is_on_floor()` for ground detection, reliable and optimized
- Avoid `move_and_collide()` for coyote checks; use `CharacterBody2D` built-ins
- `Time.get_ticks_msec()` more precise than `delta` accumulation for timing
- Consider using `PhysicsServer2D.body_test_motion()` for complex ground detection
- Godot 4.x: `move_and_slide()` improvements make ground detection more reliable
- Export coyote duration as `@export_range(0.05, 0.25)` for designer tuning
- Use `@onready` for timer references to avoid null checks

**Synergies:**
- **Input Buffering (#72):** Combined, these create "it just works" feel. Buffer jump before landing + coyote after leaving = massive input window
- **Jump Buffering in Air (#78):** Coyote extends grounded jump window, air buffer extends aerial jump
- **Ledge Forgiveness (#73):** Both compensate for positional imprecision
- **Variable Jump Height:** Coyote jump should respect full jump height range
- **Wall Jump (#74):** Implement separate wall-coyote with shorter duration

**Measurement/Profiling:**
- **Playtest metrics:** Track jump success rate at ledge edges
  - Without coyote: 70-80% success rate
  - With coyote: 95-99% success rate
- **Input recording:** Capture frame-perfect data of jump press vs. grounded state
- **Analytics:** Monitor failed jump attempts (jump pressed with no response)
- **A/B testing:** Vary coyote duration, measure player completion rates
- **Developer tools:**
  - Display coyote timer on-screen during testing
  - Log coyote jump activations vs. normal jumps
  - Heat map of where coyote activates most frequently
- **Target metrics:**
  - Coyote usage: 10-25% of all jumps (indicates natural use)
  - Failed jump rate: <5% (from 20-30% without)
  - Player complaints about "missing jumps": Should approach zero

**Debug Visualization:**
```gdscript
# Add to player script for testing
func _draw():
    if OS.is_debug_build():
        # Draw coyote time window
        if time_since_grounded <= COYOTE_TIME and not is_grounded:
            var color = Color.YELLOW
            color.a = 1.0 - (time_since_grounded / COYOTE_TIME)
            draw_circle(Vector2.ZERO, 8, color)

            # Draw timer text
            var time_left = COYOTE_TIME - time_since_grounded
            draw_string(ThemeDB.fallback_font, Vector2(-20, -30),
                "Coyote: %.2f" % time_left, HORIZONTAL_ALIGNMENT_LEFT,
                -1, 12, color)
```

**Performance Impact:**
- CPU: Negligible (<0.01ms per frame)
- Memory: 12-16 bytes per entity with coyote time
- No allocation overhead (value types only)
- Safe to implement on 100+ entities simultaneously
