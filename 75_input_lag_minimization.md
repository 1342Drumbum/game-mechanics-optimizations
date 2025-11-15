### 75. Input Lag Minimization

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Delay between button press and on-screen response, causing disconnected "mushy" controls. Input lag compounds through the entire rendering pipeline: OS input polling (1-8ms) → game logic (16.67ms frame) → rendering (16.67ms) → display (4-16ms) → pixel response (1-5ms) = 40-120ms total. Studies show >80ms lag is noticeable to 75% of players, >100ms is universally perceived as unresponsive. Fighting games and rhythm games become unplayable above 50ms, platformers feel "off" above 67ms.

**Technical Explanation:**
Input lag minimization requires optimization at multiple pipeline stages. Key techniques: (1) **Poll inputs at start of frame** before physics/logic updates, (2) **Reduce frame time** to <16.67ms for consistent 60fps, (3) **Disable VSync** or use triple buffering to prevent GPU stalling, (4) **Minimize render queue depth** (1-2 frames max), (5) **Use game mode** on displays to reduce post-processing lag, (6) **Implement frame interpolation** for visual smoothness without input delay.

The critical insight: input lag is the sum of all delays in the loop. Reducing any single stage compounds: shaving 8ms from rendering + 8ms from input polling + 4ms from display = 20ms total reduction (from 100ms → 80ms = 20% faster response).

**Algorithmic Complexity:**
- Input polling: O(1) per input device
- Frame time optimization: Varies by bottleneck
- Render queue management: O(1) configuration

**Implementation Pattern:**
```gdscript
# Godot implementation
extends Node

# Input lag optimization settings
const TARGET_FPS = 60
const TARGET_FRAME_TIME_MS = 16.67
const VSYNC_MODE = DisplayServer.VSYNC_DISABLED  # Or VSYNC_MAILBOX for tearing prevention
const MAX_PHYSICS_STEPS = 8
const PHYSICS_TICKS_PER_SECOND = 60

var input_state: Dictionary = {}
var last_poll_time: int = 0
var frame_times: Array[float] = []
var avg_frame_time: float = 0.0

func _ready() -> void:
    # Configure for minimum latency
    setup_low_latency_mode()

    # Set fixed physics tickrate
    Engine.physics_ticks_per_second = PHYSICS_TICKS_PER_SECOND
    Engine.max_physics_steps_per_frame = MAX_PHYSICS_STEPS

    # Optimize process priority
    if OS.has_feature("windows"):
        OS.set_thread_name("MainThread_HighPriority")

func setup_low_latency_mode() -> void:
    # Disable VSync for minimum input lag
    DisplayServer.window_set_vsync_mode(VSYNC_MODE)

    # Set target FPS
    Engine.max_fps = TARGET_FPS

    # Reduce audio buffer size (trade-off: may crackle on slow systems)
    var audio_driver = AudioServer.get_driver_name()
    # Note: Godot doesn't expose buffer size directly, use project settings
    # ProjectSettings.set_setting("audio/driver/output_latency", 15)  # 15ms

    # Enable low processor usage mode = false for minimum latency
    OS.low_processor_usage_mode = false

func _process(delta: float) -> void:
    # Track frame time for monitoring
    track_frame_performance(delta)

    # Poll inputs at very start of frame
    poll_inputs_early()

func _physics_process(delta: float) -> void:
    # Use cached input state from early polling
    process_game_logic_with_cached_inputs()

func poll_inputs_early() -> void:
    # Capture input state at frame start
    var poll_time = Time.get_ticks_usec()

    # Cache all relevant inputs for this frame
    input_state = {
        "move_left": Input.is_action_pressed("move_left"),
        "move_right": Input.is_action_pressed("move_right"),
        "jump": Input.is_action_just_pressed("jump"),
        "attack": Input.is_action_just_pressed("attack"),
        "mouse_position": get_viewport().get_mouse_position(),
    }

    # Track polling time
    var poll_duration = Time.get_ticks_usec() - poll_time
    if poll_duration > 1000:  # >1ms is unusually slow
        push_warning("Input polling took %.2fms" % (poll_duration / 1000.0))

    last_poll_time = poll_time

func process_game_logic_with_cached_inputs() -> void:
    # Use input_state instead of calling Input.is_action_* directly
    # This ensures consistent input across frame even with multiple queries
    if input_state.jump:
        execute_jump()

    if input_state.attack:
        execute_attack()

func track_frame_performance(delta: float) -> void:
    var frame_time_ms = delta * 1000.0
    frame_times.append(frame_time_ms)

    # Keep rolling average of last 60 frames
    if frame_times.size() > 60:
        frame_times.pop_front()

    avg_frame_time = frame_times.reduce(func(acc, val): return acc + val, 0.0) / frame_times.size()

    # Warn if consistently over budget
    if avg_frame_time > TARGET_FRAME_TIME_MS * 1.1:  # 10% over budget
        push_warning("Frame time over budget: %.2fms (target: %.2fms)" %
                     [avg_frame_time, TARGET_FRAME_TIME_MS])

func execute_jump():
    # Jump logic here
    pass

func execute_attack():
    # Attack logic here
    pass
```

**Advanced Low-Latency System:**
```gdscript
class_name LowLatencyInputSystem

# Multi-stage latency tracking
class LatencyBreakdown:
    var input_poll_us: int = 0      # Microseconds to poll inputs
    var logic_update_us: int = 0    # Microseconds for game logic
    var render_us: int = 0          # Microseconds for rendering
    var total_frame_us: int = 0     # Total frame time

    func get_total_ms() -> float:
        return total_frame_us / 1000.0

    func print_breakdown():
        print("=== Latency Breakdown ===")
        print("  Input Poll: %.2fms" % (input_poll_us / 1000.0))
        print("  Logic: %.2fms" % (logic_update_us / 1000.0))
        print("  Render: %.2fms" % (render_us / 1000.0))
        print("  Total: %.2fms" % get_total_ms())

var latency_samples: Array[LatencyBreakdown] = []
var current_breakdown: LatencyBreakdown = null

# Predictive input (1-frame lookahead)
var predicted_inputs: Dictionary = {}
var input_history: Array[Dictionary] = []

func enable_input_prediction() -> void:
    # Predict next frame's input based on current velocity
    # Useful for smoothing at lower framerates
    pass

func predict_next_input(current_input: Dictionary) -> Dictionary:
    # Simple prediction: assume input continues
    # Advanced: Use ML or pattern detection
    var prediction = current_input.duplicate()

    # Track input sequences for pattern detection
    input_history.append(current_input)
    if input_history.size() > 10:
        input_history.pop_front()

    # Detect repeating patterns (e.g., combo inputs)
    if input_history.size() >= 3:
        # Check if last 3 inputs form a pattern
        pass

    return prediction

# Direct input reading (bypass Input singleton)
func read_raw_input_state() -> Dictionary:
    # On platforms where available, read input device state directly
    # Bypasses Godot's input processing for absolute minimum latency
    var raw_state = {}

    # Godot doesn't expose raw device state easily
    # This would require GDNative/GDExtension for direct OS calls
    # On Windows: Use Raw Input API
    # On Linux: Read /dev/input directly
    # On console: Use platform SDK

    return raw_state

# Variable refresh rate synchronization
func setup_vrr_sync() -> void:
    # On displays with VRR (G-Sync/FreeSync)
    # Use VSYNC_ADAPTIVE or VSYNC_MAILBOX
    if DisplayServer.screen_get_refresh_rate() > 60:
        DisplayServer.window_set_vsync_mode(DisplayServer.VSYNC_ADAPTIVE)
```

**Key Parameters:**
- **Target framerate:** 60fps minimum (16.67ms budget)
  - Competitive games: 120-240fps (8.33-4.17ms)
  - VR: 90-120fps (11.11-8.33ms)
- **VSync mode:**
  - Disabled: Lowest latency, screen tearing
  - Mailbox: Low latency, no tearing (best compromise)
  - Adaptive: Good for VRR displays
- **Physics tick rate:** Match or exceed render framerate
- **Audio buffer:** 10-20ms (trade-off with stability)
- **Max buffered frames:** 1-2 (reduce render queue depth)

**Edge Cases:**
- **Variable framerate:** Input lag varies per frame; lock to target FPS
- **Background processing:** Pause or throttle when window loses focus
- **Multiple input devices:** Poll all devices, prioritize most recent
- **Wireless controllers:** ~5-15ms additional latency, unavoidable
- **Network games:** Client-side prediction required (separate concern)
- **Loading/streaming:** Maintain input polling even during loads
- **Alt-tab/minimize:** Reset input state to prevent stuck keys
- **Display lag varies:** Different monitors have 4-40ms pixel response

**When NOT to Use:**
- Turn-based games (no real-time input)
- Slow-paced strategy games
- Visual novels / dialogue systems
- When battery life is priority (mobile)
- Systems with severe performance constraints
- If VSync is required for other reasons (e.g., prevent GPU overheating)

**Examples from Shipped Games:**

1. **Street Fighter V (2016):** Infamous at launch for 8 frames (133ms) input lag, patched down to 4 frames (67ms). Community demanded <5 frames for competitive play. Achieved through: VSync disable option, direct input polling, reduced render queue. Demonstrates measurable impact on competitive viability.

2. **Rocket League (2015):** 2-3 frames (33-50ms) input lag on PC, achieved through aggressive optimization. Polls input at frame start, uses mailbox VSync, optimized physics tick rate. 120fps mode further reduces to 1-2 frames (16-33ms). Essential for high-level play requiring frame-perfect inputs.

3. **Celeste (2018):** Consistent 2-frame latency (33ms) at 60fps. Input polling at start of _process(), minimal rendering pipeline. Developer stated "we profile every frame to stay under 16ms." Critical for precision platforming—players can "feel" extra frames of lag.

4. **Doom Eternal (2020):** <40ms total latency on PC (game + display). Achieved through: id Tech 7 engine optimization, direct input API, reduced buffering, uncapped framerate option. At 200fps, input lag drops to ~20ms. Essential for fast-paced combat feeling "connected."

5. **Guitar Hero III (2007):** Implemented automatic audio/video calibration to compensate for display lag. Manual offset adjustment ±150ms. Without compensation, rhythm games are unplayable on modern TVs (40-80ms lag). Shows importance of accounting for display latency.

**Platform Considerations:**
- **PC (high-end):**
  - 240Hz monitors: Can achieve <20ms total latency
  - Uncapped framerate reduces lag proportionally
  - Direct input APIs (Raw Input on Windows)
- **Console (60fps locked):**
  - ~67ms minimum (2 frames rendering + display lag)
  - Optimize within fixed framerate constraint
  - Use mailbox VSync when available
- **Mobile:**
  - Touch latency: 50-100ms unavoidable
  - Target 60fps on high-end, 30fps acceptable on low-end
  - Disable low power mode for minimum lag
- **VR:**
  - <20ms critical for presence (nausea prevention)
  - 90-120fps minimum
  - Asynchronous Time Warp compensates for frame drops

**Godot-Specific Notes:**
- `Engine.max_fps = 0` for uncapped framerate (minimum lag)
- `DisplayServer.VSYNC_MAILBOX` best compromise for most games
- `Input.is_action_just_pressed()` uses event queue; poll at frame start
- `Input.get_vector()` for combined directional input
- Godot 4.x: Improved input system with lower latency than 3.x
- `OS.low_processor_usage_mode = false` disables frame throttling
- Profile with built-in profiler: target <16ms per frame
- `Time.get_ticks_usec()` for microsecond precision timing

**Synergies:**
- **Input Buffering (#72):** Compensates for small lag variations
- **Coyote Time (#71):** Reduces perceived impact of lag on jumps
- **Fixed Timestep:** Consistent physics simulation prevents lag spikes
- **Object Pooling:** Reduces GC pauses that cause lag spikes
- **Spatial Partitioning:** Faster collision = faster frame time = less lag

**Measurement/Profiling:**
- **LDAT (Latency Display Analysis Tool):** Nvidia hardware tool, measures actual end-to-end latency with photo sensor
- **High-speed camera:** Record screen + LED triggered by button press, count frames
- **Oscilloscope method:** Measure electrical signal to pixel change
- **Software timing:**
  - Track time from `Input.is_action_just_pressed()` to frame completion
  - Add display lag estimate (query monitor specs)
- **Subjective testing:**
  - Blind test different latency values
  - Measure player performance correlation with latency
- **Target metrics:**
  - <50ms: Excellent (fighting games, competitive)
  - 50-67ms: Good (action games, platformers)
  - 67-100ms: Acceptable (casual games)
  - >100ms: Poor (feels laggy to most players)

**Latency Budget Breakdown:**
```
Example 60fps game on modern PC:
- Input polling: 1-2ms (OS + USB)
- Game logic: 10-14ms (physics, AI, gameplay)
- Rendering: 8-12ms (draw calls, GPU work)
- Display lag: 10-20ms (monitor processing)
- Pixel response: 1-5ms (LCD/OLED transition)
= Total: 30-53ms (2-3 frames)

Optimization targets:
- Game logic: <12ms (stay under frame budget)
- Rendering: <10ms (60% of frame budget)
- VSync: Mailbox mode (1 frame buffering)
- Result: ~45ms total (acceptable for action games)
```

**Debug Monitoring:**
```gdscript
# On-screen latency display
@onready var latency_label = $UI/LatencyLabel

func _process(delta):
    # Update every 60 frames (~1 second)
    if Engine.get_frames_drawn() % 60 == 0:
        update_latency_display()

func update_latency_display():
    var fps = Engine.get_frames_per_second()
    var frame_time = 1000.0 / fps if fps > 0 else 0

    # Estimate total latency (game + typical display)
    var estimated_display_lag = 15.0  # Assume 15ms monitor lag
    var render_latency = frame_time * 2  # 2 frames buffered
    var total_latency = render_latency + estimated_display_lag

    latency_label.text = "Latency: ~%.0fms (%.0f FPS)" % [total_latency, fps]

    # Color code by quality
    if total_latency < 50:
        latency_label.modulate = Color.GREEN
    elif total_latency < 67:
        latency_label.modulate = Color.YELLOW
    else:
        latency_label.modulate = Color.RED

# Input timing test
var input_times: Array[int] = []

func _input(event):
    if event is InputEventKey and event.pressed:
        input_times.append(Time.get_ticks_usec())

        # Measure time to visual response (next frame rendered)
        # This requires frame completion callback
        get_viewport().render_target_update_mode = SubViewport.UPDATE_ONCE

func measure_input_to_render():
    if input_times.size() > 0:
        var input_time = input_times.pop_front()
        var render_time = Time.get_ticks_usec()
        var latency = (render_time - input_time) / 1000.0
        print("Input-to-render latency: %.2fms" % latency)
```

**Performance Impact:**
- Disabling VSync: Potentially higher GPU load (uncapped fps)
- Input polling: <0.1ms per frame
- Frame time tracking: Negligible (<0.01ms)
- Low processor mode disabled: Higher CPU/power consumption
- Trade-off: Lower latency vs. higher power usage
