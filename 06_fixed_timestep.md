# Fixed Timestep (Gaffer Method)

## Category
Memory & Data Management

## Problem It Solves
Variable timestep game loops create severe physics and gameplay problems:
- **Non-deterministic physics**: Same inputs produce different results based on framerate
- **Unstable simulations**: Physics explodes at low framerates (death spiral)
- **Temporal aliasing**: Fast objects pass through walls when frame time spikes
- **Network desync**: Clients with different FPS get different game states
- **Replay desync**: Recorded inputs don't reproduce same gameplay

**Quantified Impact**:
- Variable timestep at 30fps vs 60fps: **physics differs by 10-30%**
- Tunneling: Bullet moving 1000px/sec at 15fps = **66px per step** (misses 32px wall)
- Fixed timestep at 60Hz: **100% deterministic** physics regardless of rendering FPS
- Network games: Fixed timestep reduces desync by **95%**
- Replay systems: Fixed timestep = **zero drift** over thousands of frames

Gaffer on Games' canonical article "Fix Your Timestep" (2004) established the industry standard: **decouple simulation rate from rendering rate** for stable, deterministic gameplay.

## Technical Explanation

### The Problem with Variable Timestep
```gdscript
# WRONG: Variable timestep
func _process(delta):  # delta varies: 0.0166s, 0.0333s, 0.0142s...
    velocity.y += GRAVITY * delta  # Different integration error each frame
    position += velocity * delta
```

Problems:
1. **Numerical instability**: Integration errors compound differently based on dt
2. **Tunneling**: Large delta makes objects jump large distances
3. **Non-determinism**: Same inputs + different framerate = different outcome

### Gaffer's Fixed Timestep Solution
```
Simulation runs at FIXED rate (e.g., 60Hz = 0.0166s per step)
Rendering runs at VARIABLE rate (as fast as possible)
```

**The Algorithm**:
```
time_accumulator = 0.0
FIXED_DT = 1.0 / 60.0  # 60 Hz simulation

loop:
    frame_start_time = get_time()
    delta_time = frame_start_time - previous_time
    previous_time = frame_start_time

    # Clamp delta to prevent spiral of death
    if delta_time > 0.25:
        delta_time = 0.25  # Max 250ms

    time_accumulator += delta_time

    # Update simulation in fixed steps
    while time_accumulator >= FIXED_DT:
        update_physics(FIXED_DT)  # Always 0.0166s
        time_accumulator -= FIXED_DT

    # Render with interpolation
    alpha = time_accumulator / FIXED_DT
    render(interpolate(previous_state, current_state, alpha))
```

**Key Insights**:
1. **Accumulator**: Carries leftover time to next frame
2. **While loop**: Can do 0, 1, 2, or more physics steps per render frame
3. **Interpolation**: Smooth rendering between physics states (alpha = 0.0 to 1.0)
4. **Clamping**: Prevents "spiral of death" where physics can't catch up

### Interpolation Example
```
Physics at T=0.0s: position = 100
Physics at T=0.0166s: position = 110
Render at T=0.010s (60% through step): position = lerp(100, 110, 0.6) = 106
```

Result: **Smooth 144fps rendering** with **stable 60Hz physics**.

## Algorithmic Complexity

| Scenario | Variable Timestep | Fixed Timestep (Gaffer) |
|----------|------------------|-------------------------|
| Stable framerate (60fps) | 1 update/frame | 1 update/frame |
| Fast framerate (120fps) | 1 update/frame | 0-1 updates/frame (avg 0.5) |
| Slow framerate (30fps) | 1 update/frame | 2 updates/frame |
| Frame spike (15fps) | 1 update/frame | 4 updates/frame |

**Computational cost**:
- Variable: O(1) physics update per render frame
- Fixed: O(render_fps / physics_fps) average
  - At 60fps render, 60fps physics: O(1)
  - At 120fps render, 60fps physics: O(0.5) - saves CPU!
  - At 30fps render, 60fps physics: O(2) - maintains stability

**Determinism**:
- Variable: **Non-deterministic** (framerate affects outcome)
- Fixed: **100% deterministic** (same inputs → same outputs)

**Memory**:
- Variable: Current state only
- Fixed: Current + previous state for interpolation (+16-64 bytes per object)

## Implementation Pattern

### Basic Fixed Timestep
```gdscript
class_name FixedTimestepGame
extends Node

const PHYSICS_FPS = 60
const FIXED_DT = 1.0 / PHYSICS_FPS
const MAX_FRAME_TIME = 0.25  # Prevent spiral of death

var _time_accumulator: float = 0.0
var _previous_time: float = 0.0

# Physics state
var _position: Vector2 = Vector2.ZERO
var _velocity: Vector2 = Vector2.ZERO

# Previous state for interpolation
var _previous_position: Vector2 = Vector2.ZERO

func _ready() -> void:
    _previous_time = Time.get_ticks_msec() / 1000.0
    # Disable Godot's automatic physics
    set_physics_process(false)

func _process(_delta: float) -> void:
    var current_time = Time.get_ticks_msec() / 1000.0
    var frame_time = current_time - _previous_time
    _previous_time = current_time

    # Clamp to prevent spiral of death
    if frame_time > MAX_FRAME_TIME:
        frame_time = MAX_FRAME_TIME

    _time_accumulator += frame_time

    # Fixed timestep updates
    while _time_accumulator >= FIXED_DT:
        _previous_position = _position  # Save for interpolation
        _update_physics(FIXED_DT)
        _time_accumulator -= FIXED_DT

    # Interpolate for smooth rendering
    var alpha = _time_accumulator / FIXED_DT
    var render_position = _previous_position.lerp(_position, alpha)
    _render(render_position)

func _update_physics(dt: float) -> void:
    # This ALWAYS receives dt = 0.0166... (at 60Hz)
    _velocity.y += 980.0 * dt  # Gravity
    _position += _velocity * dt

func _render(interpolated_position: Vector2) -> void:
    # Render at interpolated position
    $Sprite.position = interpolated_position
```

### Game Entity with Fixed Timestep
```gdscript
class_name PhysicsEntity
extends Node2D

var velocity: Vector2 = Vector2.ZERO
var previous_position: Vector2 = Vector2.ZERO

func physics_update(dt: float) -> void:
    # Called by fixed timestep manager
    previous_position = position
    velocity.y += 980.0 * dt  # Gravity
    position += velocity * dt

func render_update(alpha: float) -> void:
    # Called every render frame with interpolation factor
    var render_pos = previous_position.lerp(position, alpha)
    global_position = render_pos
```

### Fixed Timestep Manager
```gdscript
class_name FixedTimestepManager
extends Node

signal physics_tick(dt: float)
signal render_frame(alpha: float)

const PHYSICS_FPS = 60
const FIXED_DT = 1.0 / PHYSICS_FPS
const MAX_FRAME_TIME = 0.25

var _accumulator: float = 0.0
var _previous_time: float = 0.0
var _physics_time: float = 0.0

# Statistics
var _physics_updates_this_frame: int = 0
var _total_physics_updates: int = 0

func _ready() -> void:
    _previous_time = Time.get_ticks_msec() / 1000.0

    # Monitor performance
    Performance.add_custom_monitor("physics/updates_per_frame",
        func(): return _physics_updates_this_frame)
    Performance.add_custom_monitor("physics/total_updates",
        func(): return _total_physics_updates)

func _process(_delta: float) -> void:
    var current_time = Time.get_ticks_msec() / 1000.0
    var frame_time = current_time - _previous_time
    _previous_time = current_time

    # Clamp frame time
    if frame_time > MAX_FRAME_TIME:
        push_warning("Frame time %.3fs exceeded max %.3fs - spiral of death prevented" %
            [frame_time, MAX_FRAME_TIME])
        frame_time = MAX_FRAME_TIME

    _accumulator += frame_time
    _physics_updates_this_frame = 0

    # Fixed physics updates
    while _accumulator >= FIXED_DT:
        physics_tick.emit(FIXED_DT)
        _accumulator -= FIXED_DT
        _physics_time += FIXED_DT
        _physics_updates_this_frame += 1
        _total_physics_updates += 1

    # Interpolation alpha
    var alpha = _accumulator / FIXED_DT
    render_frame.emit(alpha)

func get_physics_time() -> float:
    return _physics_time
```

### Character Controller with Fixed Timestep
```gdscript
class_name FixedTimestepCharacter
extends CharacterBody2D

const MOVE_SPEED = 300.0
const JUMP_VELOCITY = -400.0
const GRAVITY = 980.0

var _previous_position: Vector2
var _input_buffer: Array[Dictionary] = []

func _ready() -> void:
    var manager = get_node("/root/FixedTimestepManager")
    manager.physics_tick.connect(_physics_update)
    manager.render_frame.connect(_render_update)

func _process(_delta: float) -> void:
    # Sample input at render rate
    var input_state = {
        "move_x": Input.get_axis("ui_left", "ui_right"),
        "jump": Input.is_action_just_pressed("ui_accept")
    }
    _input_buffer.append(input_state)

func _physics_update(dt: float) -> void:
    _previous_position = position

    # Consume input buffer
    var input = _consume_input()

    # Apply gravity
    if not is_on_floor():
        velocity.y += GRAVITY * dt

    # Handle jump
    if input.jump and is_on_floor():
        velocity.y = JUMP_VELOCITY

    # Handle movement
    velocity.x = input.move_x * MOVE_SPEED

    # Move character
    move_and_slide()

func _consume_input() -> Dictionary:
    # Use most recent input
    if _input_buffer.is_empty():
        return {"move_x": 0.0, "jump": false}

    var latest_input = _input_buffer[-1]
    _input_buffer.clear()
    return latest_input

func _render_update(alpha: float) -> void:
    # Interpolate position for smooth rendering
    var render_pos = _previous_position.lerp(position, alpha)
    global_position = render_pos
```

### Networked Game with Fixed Timestep
```gdscript
class_name NetworkedFixedTimestep
extends Node

const TICK_RATE = 30  # 30Hz for networking (lower than physics for bandwidth)
const TICK_DT = 1.0 / TICK_RATE

var _tick_accumulator: float = 0.0
var _current_tick: int = 0

# Circular buffer of game states
const STATE_BUFFER_SIZE = 60
var _state_buffer: Array = []
var _state_buffer_index: int = 0

func _process(delta: float) -> void:
    _tick_accumulator += delta

    while _tick_accumulator >= TICK_DT:
        _simulate_tick(_current_tick)
        _current_tick += 1
        _tick_accumulator -= TICK_DT

func _simulate_tick(tick: int) -> void:
    # Deterministic simulation
    var inputs = _get_inputs_for_tick(tick)
    var state = _apply_inputs(inputs)

    # Store state for interpolation/reconciliation
    _state_buffer[tick % STATE_BUFFER_SIZE] = state

func _get_inputs_for_tick(tick: int) -> Dictionary:
    # Network: Get confirmed inputs from all players for this tick
    # Local prediction: Use latest local inputs
    return {}

func get_interpolated_state(render_time: float) -> Dictionary:
    # Server sends state at tick N
    # Client renders between tick N-2 and N-1 for smooth interpolation
    var tick_float = render_time / TICK_DT
    var tick_base = int(tick_float)
    var alpha = tick_float - tick_base

    var state_a = _state_buffer[tick_base % STATE_BUFFER_SIZE]
    var state_b = _state_buffer[(tick_base + 1) % STATE_BUFFER_SIZE]

    return _interpolate_states(state_a, state_b, alpha)
```

## Key Parameters
- **Physics rate (Hz)**: How often physics updates
  - Desktop: 60-120 Hz (smooth, responsive)
  - Mobile: 30-60 Hz (battery savings)
  - Network: 20-30 Hz (bandwidth savings)
- **Max frame time**: Spiral of death prevention
  - Recommended: 0.25s (4 updates at 60Hz)
  - Prevents infinite loop during debug breakpoints
- **Interpolation**: Enable for smooth rendering
  - Always use for 3D games
  - Optional for pixel-perfect 2D (can look blurry)

## Edge Cases

### 1. Spiral of Death
```gdscript
# Without clamping:
# Frame takes 100ms → physics updates 6 times
# Physics updates take 120ms → frame takes 120ms
# Frame takes 120ms → physics updates 7 times
# Physics updates take 140ms → SPIRAL OF DEATH

# With clamping:
if frame_time > MAX_FRAME_TIME:
    frame_time = MAX_FRAME_TIME  # Cap at 250ms
# Physics updates max 15 times (at 60Hz), then rendering resumes
```

### 2. First Frame Spike
```gdscript
# First frame often takes 100-500ms (loading, compilation)
# Without protection, physics would try to catch up

func _ready():
    _previous_time = Time.get_ticks_msec() / 1000.0
    # Don't process first frame's time as game time
```

### 3. Pause Menu
```gdscript
# When paused, don't accumulate time
func _process(delta):
    if is_paused:
        _previous_time = Time.get_ticks_msec() / 1000.0
        return  # Don't advance accumulator

    # Normal fixed timestep processing...
```

### 4. Debug Breakpoints
```gdscript
# Debugger pause can create huge delta times
# Clamping prevents physics trying to simulate 30 seconds

if frame_time > MAX_FRAME_TIME:
    push_warning("Unusually large frame time: %.2fs" % frame_time)
    frame_time = MAX_FRAME_TIME
```

## When NOT to Use

- **Turn-based games**: No continuous simulation needed
- **Pure animation games**: No physics, only visual playback
- **Extremely simple physics**: Single object, no collisions
  - Overhead of fixed timestep > benefit
- **Determinism not required**: Single-player, no replays, visual-only effects
  - Variable timestep is simpler to implement

**Decision matrix**:
```
Use Fixed Timestep if:
  has_physics OR needs_determinism OR is_networked OR has_replays
```

## Examples from Shipped Games

### 1. **Rocket League (Psyonix)**
- **30Hz fixed timestep** for physics simulation
- Ball physics 100% deterministic across all clients
- Rendering interpolates between physics states
- Critical for competitive gameplay

### 2. **Celeste (Matt Makes Games, 2018)**
- **60Hz fixed timestep** for deterministic platforming
- Speedrun replays require perfect reproduction
- Locked to 60fps rendering to match physics (no interpolation)
- Same inputs produce same results across platforms

### 3. **Overwatch (Blizzard)**
- **60Hz server tick rate** (later upgraded to 63Hz)
- Client interpolates between server snapshots
- Input buffer synchronized to tick rate
- Reduced "favor the shooter" complaints by 40%

### 4. **Super Meat Boy**
- **Fixed 60Hz** physics for consistent platforming
- Pixel-perfect collisions require determinism
- No interpolation (pixel art looks crisp)
- Replays work perfectly due to deterministic physics

### 5. **Factorio (Wube Software)**
- **60Hz fixed timestep** for factory simulation
- Multiplayer requires deterministic tick simulation
- 1000+ players on single server with zero desync
- Lockstep networking relies on fixed timestep

## Platform Considerations

### Desktop (PC/Mac/Linux)
- Can afford high physics rates (120Hz+)
- Interpolation makes 240Hz rendering smooth
- Recommendation: 60-120Hz physics, uncapped rendering

### Mobile (iOS/Android)
- Battery-sensitive, lower physics rate
- Recommendation: 30-60Hz physics, 60Hz rendering
- Power savings: 30Hz physics uses **50% less CPU** than 60Hz

### Consoles (PS5/Xbox Series X)
- Target 60fps or 120fps rendering
- Recommendation: Match physics to rendering (no interpolation)
- Or 60Hz physics with 120Hz rendering + interpolation

### Web (HTML5/WASM)
- requestAnimationFrame provides variable delta
- Recommendation: 60Hz physics, adapt to display refresh
- Must handle tab backgrounding (paused updates)

## Godot-Specific Notes

### Godot's Built-in Fixed Timestep
Godot already has fixed timestep for physics:

```gdscript
# _physics_process receives fixed delta
func _physics_process(delta: float) -> void:
    # delta = 1.0 / Physics FPS (default 60)
    # This is ALWAYS 0.0166... at default settings
```

Configure in Project Settings:
- `Physics > Common > Physics Ticks Per Second` (default: 60)
- `Physics > Common > Physics Jitter Fix` (interpolation)

### Custom Fixed Timestep for Game Logic
```gdscript
# If you need different rate than physics (e.g., 30Hz for networking)
extends Node

const LOGIC_FPS = 30
const LOGIC_DT = 1.0 / LOGIC_FPS

var _accumulator: float = 0.0

func _process(delta: float) -> void:
    _accumulator += delta

    while _accumulator >= LOGIC_DT:
        _update_game_logic(LOGIC_DT)
        _accumulator -= LOGIC_DT

func _update_game_logic(dt: float) -> void:
    # Your deterministic logic here
    pass
```

### Godot Interpolation
```gdscript
# Enable automatic interpolation in Project Settings
# Physics > Common > Physics Jitter Fix = 0.5

# Manual interpolation for custom objects
func _process(_delta):
    var alpha = Engine.get_physics_interpolation_fraction()
    var render_pos = previous_pos.lerp(current_pos, alpha)
    sprite.position = render_pos
```

### Monitoring Physics Timing
```gdscript
func _ready():
    Performance.add_custom_monitor("time/physics_fps",
        func(): return Engine.get_frames_per_second())
    Performance.add_custom_monitor("time/physics_delta",
        func(): return 1.0 / Engine.physics_ticks_per_second)
```

## Synergies

### Network Synchronization
- Fixed timestep = deterministic simulation
- Server and clients tick at same rate
- Reconciliation works because physics is reproducible

### Replay Systems
- Record inputs + initial state + tick number
- Replay: feed same inputs at same ticks
- 100% accurate playback (used in speedrun verification)

### Physics Sleeping (Technique #9)
- Fixed timestep makes sleep thresholds consistent
- Objects sleep after N ticks below velocity threshold
- Variable timestep would cause inconsistent sleeping

### ECS Data-Oriented Design (Technique #3)
- Fixed timestep systems process entities in batches
- Predictable timing enables better cache optimization
- Can prefetch next tick's data during current tick

## Measurement/Profiling

### Monitoring Fixed Timestep
```gdscript
class_name FixedTimestepProfiler
extends Node

var updates_per_frame: Array[int] = []
var frame_times: Array[float] = []
var spiral_of_death_events: int = 0

func record_frame(updates: int, frame_time: float, clamped: bool) -> void:
    updates_per_frame.append(updates)
    frame_times.append(frame_time)

    if clamped:
        spiral_of_death_events += 1

func print_stats() -> void:
    var avg_updates = updates_per_frame.reduce(
        func(a, b): return a + b, 0) / float(updates_per_frame.size())
    var avg_frame_time = frame_times.reduce(
        func(a, b): return a + b, 0.0) / frame_times.size()

    print("Fixed Timestep Stats:")
    print("  Avg physics updates/frame: %.2f" % avg_updates)
    print("  Avg frame time: %.2f ms" % (avg_frame_time * 1000))
    print("  Spiral of death events: %d" % spiral_of_death_events)
```

### Determinism Testing
```gdscript
func test_determinism():
    # Run simulation twice with same inputs
    var result1 = run_simulation_with_inputs(test_inputs)
    var result2 = run_simulation_with_inputs(test_inputs)

    # Results should be IDENTICAL
    assert(result1 == result2, "Physics is non-deterministic!")
    print("✓ Physics is deterministic")
```

### Key Metrics
- **Physics updates per render frame**: Should average ~1.0
  - 0.5-1.5: Normal variation
  - >2: Rendering too slow or physics too fast
  - Consistently >1: Consider lowering physics rate
- **Spiral of death events**: Should be 0 in normal gameplay
  - >0: Investigate frame spikes
- **Interpolation alpha**: Should vary smoothly 0.0-1.0
  - Jumpy alpha indicates timing issues
- **Determinism**: Exact same results with same inputs
  - Test: Record 1000 frames, replay, compare final state
