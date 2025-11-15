### 77. Stick Deadzone Optimization

**Category:** Game Feel - Input Responsiveness

**Problem It Solves:** Analog stick drift causing unwanted movement, or overly large deadzones making small adjustments impossible. Without proper deadzone tuning, controllers either: (1) drift constantly (0% deadzone, worn controllers), or (2) feel unresponsive for precise aiming (25%+ deadzone, over-corrected). Studies show default deadzones (15-25%) cause 30-50% of stick input range to be wasted, particularly problematic for aiming and precise platforming. Player frustration manifests as "can't make small adjustments" or "character moves when I'm not touching stick."

**Technical Explanation:**
Deadzones define the input threshold before stick movement registers. Types:

1. **Axial Deadzone:** Separate X/Y thresholds. Simple but creates diamond-shaped dead area, inconsistent at diagonals.

2. **Radial Deadzone:** Circular threshold based on stick magnitude. More intuitive, consistent in all directions. Standard in modern games.

3. **Scaled Radial:** Circular deadzone with remapped output range (0.15 deadzone → remap 0.15-1.0 to 0.0-1.0). Preserves full output range despite deadzone.

4. **Dual-Zone:** Separate inner deadzone (prevent drift) and outer deadzone (ensure full range). Remap between them.

Implementation uses `sqrt(x² + y²)` for magnitude, then remaps values outside deadzone to 0.0-1.0 range. Key insight: deadzone should eliminate drift while preserving maximum usable input range. Typical: 12-18% inner, 95-98% outer (accounts for stick not reaching perfect 1.0).

**Algorithmic Complexity:** O(1) - simple math per stick per frame

**Implementation Pattern:**
```gdscript
# Godot implementation
extends Node

# Deadzone configuration
@export_group("Deadzone Settings")
@export_range(0.0, 0.5, 0.01) var inner_deadzone: float = 0.15
@export_range(0.5, 1.0, 0.01) var outer_deadzone: float = 0.98
@export var deadzone_type: DeadzoneType = DeadzoneType.SCALED_RADIAL

enum DeadzoneType {
    NONE,           # No deadzone (for testing)
    AXIAL,          # Separate X/Y (legacy)
    RADIAL,         # Circular, simple
    SCALED_RADIAL,  # Circular, remapped (recommended)
    DUAL_ZONE       # Inner + outer remap
}

func _physics_process(delta: float) -> void:
    # Get raw stick input
    var raw_input = Input.get_vector("move_left", "move_right", "move_forward", "move_back")

    # Apply deadzone processing
    var processed_input = apply_deadzone(raw_input)

    # Use processed input for movement
    apply_movement(processed_input, delta)

func apply_deadzone(raw: Vector2) -> Vector2:
    match deadzone_type:
        DeadzoneType.NONE:
            return raw
        DeadzoneType.AXIAL:
            return apply_axial_deadzone(raw)
        DeadzoneType.RADIAL:
            return apply_radial_deadzone(raw)
        DeadzoneType.SCALED_RADIAL:
            return apply_scaled_radial_deadzone(raw)
        DeadzoneType.DUAL_ZONE:
            return apply_dual_zone_deadzone(raw)

    return raw

func apply_axial_deadzone(raw: Vector2) -> Vector2:
    # Simple per-axis threshold (creates diamond shape)
    var result = Vector2.ZERO

    if abs(raw.x) > inner_deadzone:
        result.x = raw.x
    if abs(raw.y) > inner_deadzone:
        result.y = raw.y

    return result

func apply_radial_deadzone(raw: Vector2) -> Vector2:
    # Circular deadzone without remapping
    var magnitude = raw.length()

    if magnitude < inner_deadzone:
        return Vector2.ZERO

    return raw

func apply_scaled_radial_deadzone(raw: Vector2) -> Vector2:
    # Circular deadzone with range remapping (RECOMMENDED)
    var magnitude = raw.length()

    # Below inner deadzone: return zero
    if magnitude < inner_deadzone:
        return Vector2.ZERO

    # Above outer deadzone: clamp to 1.0
    if magnitude > outer_deadzone:
        return raw.normalized()

    # Between deadzones: remap to 0.0-1.0 range
    var normalized_magnitude = (magnitude - inner_deadzone) / (outer_deadzone - inner_deadzone)
    normalized_magnitude = clamp(normalized_magnitude, 0.0, 1.0)

    # Preserve direction, scale magnitude
    return raw.normalized() * normalized_magnitude

func apply_dual_zone_deadzone(raw: Vector2) -> Vector2:
    # Advanced: smooth curve between inner and outer
    var magnitude = raw.length()

    if magnitude < inner_deadzone:
        return Vector2.ZERO

    # Exponential curve for smoother transition
    var normalized_magnitude = (magnitude - inner_deadzone) / (1.0 - inner_deadzone)

    # Apply response curve (optional: square for more precision at low end)
    # normalized_magnitude = normalized_magnitude * normalized_magnitude

    return raw.normalized() * clamp(normalized_magnitude, 0.0, 1.0)

func apply_movement(input: Vector2, delta: float):
    # Example movement application
    var speed = 300.0
    var velocity = input * speed
    # ... use velocity for character movement
```

**Advanced Adaptive Deadzone:**
```gdscript
class_name AdaptiveDeadzoneSystem

# Auto-calibration for drift detection
var drift_samples: Array[Vector2] = []
var calibrated_center: Vector2 = Vector2.ZERO
var recommended_deadzone: float = 0.15

# Per-controller deadzone profiles
var controller_profiles: Dictionary = {}  # device_id -> DeadzoneProfile

class DeadzoneProfile:
    var inner_deadzone: float = 0.15
    var outer_deadzone: float = 0.98
    var drift_offset: Vector2 = Vector2.ZERO
    var response_curve: Curve  # Optional for fine-tuning
    var last_calibration: int = 0  # Timestamp

func calibrate_deadzone(device_id: int, duration: float = 3.0):
    # Auto-detect drift by sampling stick at rest
    print("Calibrating controller %d... Keep sticks centered!" % device_id)

    drift_samples.clear()
    var start_time = Time.get_ticks_msec() / 1000.0

    while Time.get_ticks_msec() / 1000.0 - start_time < duration:
        var sample = Vector2(
            Input.get_joy_axis(device_id, JOY_AXIS_LEFT_X),
            Input.get_joy_axis(device_id, JOY_AXIS_LEFT_Y)
        )
        drift_samples.append(sample)
        await get_tree().process_frame

    # Calculate average drift
    var total = Vector2.ZERO
    for sample in drift_samples:
        total += sample

    calibrated_center = total / drift_samples.size()

    # Recommend deadzone as 2x max drift + safety margin
    var max_drift = 0.0
    for sample in drift_samples:
        var drift = sample.length()
        max_drift = max(max_drift, drift)

    recommended_deadzone = min(max_drift * 2.5 + 0.05, 0.35)

    print("Calibration complete!")
    print("  Drift offset: %.3f, %.3f" % [calibrated_center.x, calibrated_center.y])
    print("  Recommended deadzone: %.2f" % recommended_deadzone)

    # Save profile
    var profile = DeadzoneProfile.new()
    profile.inner_deadzone = recommended_deadzone
    profile.drift_offset = calibrated_center
    profile.last_calibration = Time.get_ticks_msec()
    controller_profiles[device_id] = profile

func apply_calibrated_deadzone(raw: Vector2, device_id: int) -> Vector2:
    # Apply drift correction first
    var corrected = raw

    if controller_profiles.has(device_id):
        var profile = controller_profiles[device_id]
        corrected -= profile.drift_offset

    # Then apply standard deadzone
    return apply_scaled_radial_deadzone(corrected)

# Response curve application
func apply_response_curve(input: Vector2, curve_type: String = "linear") -> Vector2:
    var magnitude = input.length()
    if magnitude == 0:
        return Vector2.ZERO

    var curved_magnitude = magnitude

    match curve_type:
        "linear":
            curved_magnitude = magnitude
        "squared":
            # More precision at low end, faster at high end
            curved_magnitude = magnitude * magnitude
        "smooth":
            # Smoother acceleration
            curved_magnitude = smoothstep(0.0, 1.0, magnitude)
        "custom":
            # Use Curve resource for designer control
            if has_meta("response_curve"):
                var curve: Curve = get_meta("response_curve")
                curved_magnitude = curve.sample(magnitude)

    return input.normalized() * curved_magnitude
```

**Key Parameters:**
- **Inner deadzone:** 0.10-0.20 (typical: 0.15 or 15%)
  - New controllers: 0.10-0.12
  - Average wear: 0.15-0.18
  - Heavy wear: 0.20-0.25
- **Outer deadzone:** 0.95-0.99 (typical: 0.98)
- **Response curve:**
  - Linear: 1:1 mapping (default)
  - Squared: Precision at low, speed at high
  - Cubic: Even more precision focus

**Platform/Genre Variations:**
- **FPS (aiming):** 0.12-0.15 inner, squared curve
- **Platformer (movement):** 0.15-0.18 inner, linear
- **Racing:** 0.08-0.12 inner (precision critical)
- **Fighting games:** 0.10-0.15 inner, often use d-pad instead
- **Flight sims:** 0.05-0.08 inner (need full range)

**Edge Cases:**
- **Controller wear:** Drift increases over time; adaptive deadzone
- **Different stick quality:** Xbox vs PlayStation vs third-party
- **Asymmetric wear:** Left stick (movement) wears faster than right (camera)
- **Hot-swapping controllers:** Detect device change, apply correct profile
- **Multiple controllers:** Per-device deadzone profiles
- **Stick drift during gameplay:** Dynamic adjustment if drift detected
- **Simultaneous keyboard input:** Disable deadzone for keyboard
- **Emulated controllers:** May need different deadzone values

**When NOT to Use:**
- Mouse/keyboard input (no analog stick)
- D-pad input (digital, no deadzone needed)
- Fighting games using d-pad for movement
- Arcade sticks (different hardware characteristics)
- When 0% deadzone is acceptable (brand new controllers in testing)

**Examples from Shipped Games:**

1. **Apex Legends (2019):** Extensive deadzone customization. Separate settings for move/look sticks, inner/outer deadzones, response curves. Default 0.15 inner, but pros often use 0.05-0.08 with new controllers. "Deadzone tuning" became meta knowledge for competitive play.

2. **Rocket League (2015):** 0.10-0.12 default, critical for aerial precision. Community discovered 0.05 deadzone with squared curve optimal for car control. Dodge deadzone separate from steering (0.80 threshold). Pros publish deadzone settings as part of configuration guides.

3. **Call of Duty: Modern Warfare (2019):** Dynamic aim response curve option. Standard (linear), Dynamic (squared), Linear. Also per-stick deadzone adjustment 0.00-0.50. Inner deadzone affects aim assist activation—lower deadzone = more assist triggers.

4. **Forza Horizon 5 (2021):** Default 0.0-10.0 scale (maps to 0.0-0.10). Inside deadzone: 0-10, Outside deadzone: 90-100. Steering linearity separate setting. Essential for precise racing lines; 0.05 common for sim steering.

5. **Celeste (2018):** Fixed 0.25 deadzone initially, causing complaints from speedrunners. Patched to 0.15 after community feedback. Demonstrates impact of deadzone on precision platforming—too large prevents frame-perfect inputs.

**Platform Considerations:**
- **Xbox controllers:** Generally ~0.15 deadzone adequate
- **PlayStation controllers:** DualShock 4/5 slightly less drift, 0.12-0.15
- **Switch Pro Controller:** Notorious drift issues, 0.20-0.25 often needed
- **Switch Joy-Cons:** Severe drift problems, 0.25-0.35 required for worn units
- **Third-party controllers:** Highly variable, 0.10-0.30 range
- **Older controllers:** Increase deadzone over time as wear occurs
- **Steam Deck:** Trackpads + sticks, different handling per input

**Godot-Specific Notes:**
- `Input.get_joy_axis(device, axis)` returns -1.0 to 1.0
- `Input.get_vector()` automatically applies project deadzone setting
- Override with manual `get_joy_axis()` for custom deadzone
- **Project Settings → Input Devices → Pointing:** Default deadzone 0.15
- Export deadzone as `@export_range(0.0, 0.5)` for runtime tuning
- `Input.get_connected_joypads()` for device detection
- `Input.joy_connection_changed` signal for hot-swap handling
- Godot 4.x: Improved input handling, more consistent across platforms

**Synergies:**
- **Aim Assist (#76):** Lower deadzone = more precise assist activation
- **Turn-Around Frames (#74):** Deadzone affects direction change detection
- **Camera smoothing:** Deadzones create input for camera lag/lead
- **Input buffering:** Small deadzone inputs still buffer
- **Vibration feedback:** Confirm input outside deadzone with rumble

**Measurement/Profiling:**
- **Stick visualization:** Real-time plot of raw vs. processed input
  - Draw circle showing deadzone boundaries
  - Plot stick position over time
- **Drift detection:**
  - Sample stick at rest (100+ frames)
  - Calculate average offset and max deviation
  - Recommend deadzone = max_deviation × 2.5
- **Input range utilization:**
  - Track min/max stick values during gameplay
  - If never reaching >0.9, outer deadzone too small
  - If <10% of inputs near 1.0, outer deadzone too large
- **Player surveys:**
  - "Do small movements feel responsive?" (target: >85% yes)
  - "Does character drift when idle?" (target: <5% yes)
- **Competitive analysis:**
  - Survey top players for their deadzone settings
  - Typical competitive range: 0.08-0.15

**Debug Visualization:**
```gdscript
@tool
extends Control

var raw_input: Vector2 = Vector2.ZERO
var processed_input: Vector2 = Vector2.ZERO
var inner_deadzone: float = 0.15
var outer_deadzone: float = 0.98

func _process(delta):
    # Get controller input
    raw_input = Vector2(
        Input.get_joy_axis(0, JOY_AXIS_LEFT_X),
        Input.get_joy_axis(0, JOY_AXIS_LEFT_Y)
    )

    # Apply deadzone (use your actual function)
    processed_input = apply_scaled_radial_deadzone(raw_input)

    queue_redraw()

func _draw():
    var center = size / 2
    var radius = min(size.x, size.y) / 2 - 20

    # Draw outer circle (max stick range)
    draw_circle(center, radius, Color(0.2, 0.2, 0.2))

    # Draw outer deadzone circle
    draw_arc(center, radius * outer_deadzone, 0, TAU, 32,
             Color(1, 1, 0, 0.3), 2.0)

    # Draw inner deadzone circle
    draw_circle(center, radius * inner_deadzone, Color(1, 0, 0, 0.2))
    draw_arc(center, radius * inner_deadzone, 0, TAU, 32,
             Color(1, 0, 0, 0.5), 2.0)

    # Draw crosshair
    draw_line(center - Vector2(10, 0), center + Vector2(10, 0), Color.WHITE, 1.0)
    draw_line(center - Vector2(0, 10), center + Vector2(0, 10), Color.WHITE, 1.0)

    # Draw raw input position
    var raw_pos = center + raw_input * radius
    draw_circle(raw_pos, 5, Color.RED)
    draw_line(center, raw_pos, Color(1, 0, 0, 0.5), 2.0)

    # Draw processed input position
    var processed_pos = center + processed_input * radius
    draw_circle(processed_pos, 5, Color.GREEN)
    draw_line(center, processed_pos, Color(0, 1, 0, 0.8), 3.0)

    # Draw info text
    var font = ThemeDB.fallback_font
    draw_string(font, Vector2(10, 20),
                "Raw: (%.2f, %.2f) Mag: %.2f" % [raw_input.x, raw_input.y, raw_input.length()],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.RED)
    draw_string(font, Vector2(10, 40),
                "Processed: (%.2f, %.2f) Mag: %.2f" % [processed_input.x, processed_input.y, processed_input.length()],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.GREEN)
    draw_string(font, Vector2(10, 60),
                "Inner DZ: %.0f%%  Outer DZ: %.0f%%" % [inner_deadzone * 100, outer_deadzone * 100],
                HORIZONTAL_ALIGNMENT_LEFT, -1, 14, Color.YELLOW)

    # Drift indicator
    if raw_input.length() < 0.05:
        var drift_text = "Drift: %.3f (OK)" if raw_input.length() < inner_deadzone else "Drift: %.3f (INCREASE DEADZONE)"
        var drift_color = Color.GREEN if raw_input.length() < inner_deadzone else Color.RED
        draw_string(font, Vector2(10, 80),
                    drift_text % raw_input.length(),
                    HORIZONTAL_ALIGNMENT_LEFT, -1, 14, drift_color)

# Calibration UI
func show_calibration_prompt():
    var dialog = AcceptDialog.new()
    dialog.dialog_text = """Controller Calibration

Please keep the stick centered and press OK.
Calibration will take 3 seconds.

Current drift: %.3f
Recommended deadzone: %.2f
""" % [raw_input.length(), recommended_deadzone]
    add_child(dialog)
    dialog.popup_centered()
```

**Performance Impact:**
- CPU: Negligible (<0.01ms per controller per frame)
- Memory: ~50 bytes per controller profile
- Math operations: 1 sqrt, 2-3 multiplies per stick
- Safe for 8+ simultaneous controllers
- No allocation overhead (pure value types)
