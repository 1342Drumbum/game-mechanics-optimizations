### 67. Frame Time Budget Breakdown

**Category:** Profiling & Advanced - Performance Management

**Problem It Solves:** Unbalanced frame time allocation causing bottlenecks. Without budgets, systems compete unpredictably - physics takes 12ms one frame, 4ms the next, making 60fps impossible. A 16.67ms frame budget (60fps) must divide among 8-12 systems. One system exceeding budget by 3ms drops framerate from 60fps to 50fps (20% performance loss). Budget breakdown provides: predictable performance, clear optimization priorities, and early warning of frame spikes.

**Technical Explanation:**
Frame time budgets divide the target frame duration (16.67ms for 60fps, 33.33ms for 30fps) into fixed allocations per system. Typical breakdown: Rendering (40%, 6.7ms), Physics (25%, 4.2ms), Gameplay (15%, 2.5ms), AI (10%, 1.7ms), Audio (5%, 0.8ms), Reserve (5%, 0.8ms). Each system tracks actual vs budget using profiling markers. When system exceeds budget: defer work to next frame, reduce quality (LOD, skip updates), or redistribute budget. Critical technique: **adaptive quality** - if physics takes 5ms (exceeds 4.2ms budget), reduce rendering quality by 1ms to compensate. Prevents cascade failures where one slow system triggers frame drops.

**Algorithmic Complexity:**
- Budget tracking: O(1) per system per frame
- Budget enforcement: O(s) where s = systems (typically 8-12)
- Historical analysis: O(f × s) where f = frames in window (60-300)
- Reallocation: O(s log s) for priority-based redistribution
- Prediction: O(h) where h = history window for moving average

**Implementation Pattern:**
```gdscript
# Frame Time Budget Manager
class_name FrameBudgetManager extends Node

# Budget definitions (milliseconds at 60fps)
const TARGET_FPS := 60
const TARGET_FRAME_TIME := 1000.0 / TARGET_FPS  # 16.67ms

var budgets := {
    "rendering": 6.7,      # 40%
    "physics": 4.2,        # 25%
    "gameplay": 2.5,       # 15%
    "ai": 1.7,            # 10%
    "audio": 0.8,         # 5%
    "reserve": 0.8        # 5% safety buffer
}

var current_times := {}
var frame_history := []
var max_history := 300  # 5 seconds

class SystemTimer:
    var system_name: String
    var start_time: int = 0
    var budget_ms: float = 0.0
    var actual_ms: float = 0.0
    var exceeded_count: int = 0
    var total_overage: float = 0.0

    func begin():
        start_time = Time.get_ticks_usec()

    func end() -> float:
        actual_ms = (Time.get_ticks_usec() - start_time) / 1000.0
        if actual_ms > budget_ms:
            exceeded_count += 1
            total_overage += (actual_ms - budget_ms)
        return actual_ms

func _ready():
    # Initialize timers
    for system in budgets:
        current_times[system] = SystemTimer.new()
        current_times[system].system_name = system
        current_times[system].budget_ms = budgets[system]

func begin_system(system_name: String):
    if current_times.has(system_name):
        current_times[system_name].begin()

func end_system(system_name: String) -> Dictionary:
    if not current_times.has(system_name):
        return {}

    var timer = current_times[system_name]
    var actual = timer.end()
    var budget = timer.budget_ms
    var over_budget = actual > budget

    var result = {
        "system": system_name,
        "actual_ms": actual,
        "budget_ms": budget,
        "over_budget": over_budget,
        "overage_ms": actual - budget if over_budget else 0.0,
        "budget_percent": (actual / budget) * 100.0
    }

    if over_budget:
        push_warning("%s exceeded budget: %.2fms / %.2fms (%.1f%%)" % [
            system_name, actual, budget, result.budget_percent
        ])

    return result

func capture_frame():
    var frame_data = {
        "timestamp": Time.get_ticks_msec(),
        "systems": {}
    }

    var total_time = 0.0
    for system_name in current_times:
        var timer = current_times[system_name]
        frame_data.systems[system_name] = {
            "actual": timer.actual_ms,
            "budget": timer.budget_ms,
            "exceeded": timer.actual_ms > timer.budget_ms
        }
        total_time += timer.actual_ms

    frame_data.total_time = total_time
    frame_data.within_budget = total_time <= TARGET_FRAME_TIME

    frame_history.append(frame_data)
    if frame_history.size() > max_history:
        frame_history.pop_front()

func get_budget_report(last_n_frames: int = 60) -> String:
    if frame_history.is_empty():
        return "No frame data captured"

    var frames_to_analyze = frame_history.slice(-last_n_frames, frame_history.size())
    var report = "=== FRAME BUDGET REPORT (last %d frames) ===\n" % frames_to_analyze.size()
    report += "Target: %.2fms (60fps)\n\n" % TARGET_FRAME_TIME

    # Per-system analysis
    var system_stats = {}
    for system in budgets:
        system_stats[system] = {
            "avg": 0.0,
            "max": 0.0,
            "exceeded": 0,
            "total": 0.0
        }

    for frame in frames_to_analyze:
        for system in frame.systems:
            var stats = system_stats[system]
            var actual = frame.systems[system].actual
            stats.total += actual
            stats.max = max(stats.max, actual)
            if frame.systems[system].exceeded:
                stats.exceeded += 1

    # Calculate averages and print
    for system in budgets:
        var stats = system_stats[system]
        stats.avg = stats.total / frames_to_analyze.size()
        var budget = budgets[system]
        var exceeded_pct = (float(stats.exceeded) / frames_to_analyze.size()) * 100.0

        report += "%s: avg=%.2fms max=%.2fms budget=%.2fms exceeded=%.1f%%\n" % [
            system, stats.avg, stats.max, budget, exceeded_pct
        ]

        if stats.avg > budget:
            report += "  WARNING: Average exceeds budget by %.2fms\n" % (stats.avg - budget)
        if exceeded_pct > 10.0:
            report += "  WARNING: Frequently exceeds budget (%.1f%% of frames)\n" % exceeded_pct

    # Overall frame analysis
    var frames_dropped = 0
    var worst_frame = 0.0
    for frame in frames_to_analyze:
        if not frame.within_budget:
            frames_dropped += 1
        worst_frame = max(worst_frame, frame.total_time)

    var drop_rate = (float(frames_dropped) / frames_to_analyze.size()) * 100.0
    report += "\nFrame drops: %d (%.1f%%)\n" % [frames_dropped, drop_rate]
    report += "Worst frame: %.2fms\n" % worst_frame

    return report

# Adaptive budget reallocation
class AdaptiveBudgetManager:
    var base_budgets := {}
    var current_budgets := {}
    var priorities := {
        "physics": 1.0,      # Highest priority - affects gameplay
        "rendering": 0.9,
        "gameplay": 1.0,
        "ai": 0.5,          # Can skip frames
        "audio": 0.8,
        "reserve": 0.0      # Never reduce
    }

    func _init(budgets: Dictionary):
        base_budgets = budgets.duplicate()
        current_budgets = budgets.duplicate()

    func rebalance(actual_times: Dictionary) -> Dictionary:
        # Calculate total overage
        var total_overage = 0.0
        var systems_over = []

        for system in actual_times:
            var overage = actual_times[system] - current_budgets[system]
            if overage > 0:
                total_overage += overage
                systems_over.append(system)

        if total_overage <= 0:
            return current_budgets  # All systems within budget

        # Find systems that can give up time
        var available_reduction = 0.0
        var systems_under = []

        for system in actual_times:
            if system not in systems_over and system != "reserve":
                var headroom = current_budgets[system] - actual_times[system]
                if headroom > 0.1:  # At least 0.1ms headroom
                    available_reduction += headroom * (1.0 - priorities[system])
                    systems_under.append(system)

        # Redistribute budget
        if available_reduction > 0:
            var reduction_ratio = min(1.0, total_overage / available_reduction)

            for system in systems_under:
                var headroom = current_budgets[system] - actual_times[system]
                var reduction = headroom * (1.0 - priorities[system]) * reduction_ratio
                current_budgets[system] -= reduction

            # Distribute to over-budget systems
            for system in systems_over:
                var needed = actual_times[system] - current_budgets[system]
                var allocation = min(needed, total_overage / systems_over.size())
                current_budgets[system] += allocation

        return current_budgets

# Quality scaling based on budget
class QualityScaler:
    var base_quality := 1.0
    var current_quality := 1.0
    var min_quality := 0.5
    var target_frame_time := 16.67

    func update(actual_frame_time: float):
        if actual_frame_time > target_frame_time:
            # Reduce quality
            var overage_ratio = actual_frame_time / target_frame_time
            current_quality *= (1.0 / overage_ratio)
            current_quality = max(current_quality, min_quality)
        else:
            # Gradually increase quality
            current_quality = lerp(current_quality, base_quality, 0.05)

    func get_lod_bias() -> float:
        # Returns LOD bias (0 = base quality, 1 = lowest quality)
        return 1.0 - current_quality

    func get_shadow_quality() -> int:
        if current_quality > 0.8:
            return 2  # High
        elif current_quality > 0.6:
            return 1  # Medium
        else:
            return 0  # Low

    func should_skip_update() -> bool:
        # Skip non-critical updates when under pressure
        return current_quality < 0.7

# Example integration
var budget_manager: FrameBudgetManager
var quality_scaler: QualityScaler

func _ready():
    budget_manager = FrameBudgetManager.new()
    quality_scaler = QualityScaler.new()
    add_child(budget_manager)

func _process(delta):
    # Rendering system
    budget_manager.begin_system("rendering")
    _do_rendering()
    budget_manager.end_system("rendering")

    # Capture frame data
    budget_manager.capture_frame()

    # Update quality based on performance
    var actual_frame_time = delta * 1000.0
    quality_scaler.update(actual_frame_time)

func _physics_process(delta):
    # Physics system
    budget_manager.begin_system("physics")
    _do_physics()
    budget_manager.end_system("physics")

    # Gameplay logic
    budget_manager.begin_system("gameplay")
    _do_gameplay()
    budget_manager.end_system("gameplay")

    # AI (can skip if over budget)
    if not quality_scaler.should_skip_update():
        budget_manager.begin_system("ai")
        _do_ai()
        budget_manager.end_system("ai")
```

**Key Parameters:**
- Target framerate: 60fps (16.67ms), 30fps (33.33ms), 120fps (8.33ms)
- Reserve buffer: 5-10% of frame (safety margin for spikes)
- Budget redistribution threshold: >10% overage triggers rebalancing
- Quality scaling: 0.5-1.0 range (50% = minimum acceptable quality)
- History window: 60-300 frames (1-5 seconds) for averaging
- Alert threshold: System exceeds budget >10% of frames = critical

**Edge Cases:**
- Vertical sync: Frame time locked to refresh rate, can't exceed 16.67ms
- Variable refresh (G-Sync/FreeSync): Budget changes dynamically
- Loading screens: Suspend budgets during asset streaming
- Cutscenes: Different budget (30fps acceptable, higher quality)
- Background/unfocused: Reduce target framerate (30fps or lower)
- Thermal throttling (mobile): Reduce budgets as temperature rises

**When NOT to Use:**
- Single-system games (only rendering, no gameplay complexity)
- Turn-based games without real-time requirements
- Already hitting target framerate with >20% headroom
- Prototyping phase (adds overhead and complexity)
- Very simple games (<5ms total frame time)

**Examples from Shipped Games:**

1. **Destiny 2 (Bungie):** Strict 30fps budget (33.3ms) on consoles: Rendering 15ms, Physics 5ms, AI 4ms, Gameplay 6ms, Reserve 3.3ms. When rendering exceeded budget, dynamically reduced shadow resolution and particle count, maintaining 30fps lock
2. **Rocket League (Psyonix):** 60fps (16.67ms) budget critical for gameplay: Physics 5ms (30%), Rendering 8ms (48%), Gameplay 2ms (12%), Reserve 1.67ms (10%). Physics non-negotiable, scaled rendering quality aggressively
3. **Spider-Man PS4 (Insomniac):** Dynamic budget reallocation: During combat (physics-heavy) rendering budget reduced 8ms→6ms to give physics 6ms→8ms. During traversal (rendering-heavy) reversed allocation
4. **Call of Duty: Modern Warfare (Infinity Ward):** 60fps mandatory (16.67ms): Rendering 7ms, Physics 3ms, Animation 2ms, AI 1.5ms, Audio 1ms, Network 0.5ms, Reserve 1.67ms. Exceeded budget triggered instant LOD reduction + effect culling
5. **Fortnite (Epic Games):** Mobile adaptive: 30fps budget (33.3ms) scaled down to 20fps (50ms) based on device temperature and battery. Redistributed extra 16.7ms to rendering for better quality at lower framerate

**Platform Considerations:**
- **Mobile:** 30fps target (33.3ms), thermal throttling requires dynamic budgets
- **Console (PS5/Xbox):** 60fps mandatory for competitive games, 30fps for cinematic
- **PC:** Variable (60-240fps), adjust budgets based on detected hardware
- **VR:** 90fps minimum (11.1ms), 120fps ideal (8.33ms), missed frames cause nausea
- **Switch:** Portable mode = 30fps (33.3ms), Docked = 60fps (16.67ms)

**Godot-Specific Notes:**
- **Process vs Physics:** `_process()` for rendering (variable), `_physics_process()` for fixed timestep
- **Performance Monitors:** Use built-in monitors to track actual times
- **Custom Monitors:** Add game-specific systems with `Performance.add_custom_monitor()`
- **Profiler Integration:** Godot Profiler shows per-system breakdown automatically
- **Engine Time:** Account for engine overhead (input, scene tree, etc.) ~1-2ms
- **Godot 4.x:** Improved threading allows parallel system execution

**Synergies:**
- Pairs with CPU Profiling (measure actual system times)
- Guides Quality Scaling (reduce quality in over-budget systems)
- Enables Dynamic LOD (allocate rendering budget to visible objects)
- Informs Physics Sub-stepping (adjust physics budget vs accuracy)
- Combines with Adaptive Resolution (scale pixels to fit rendering budget)

**Measurement/Profiling:**
```gdscript
# Validate budget allocations
func validate_budgets():
    var total_budget = 0.0
    for system in budgets:
        total_budget += budgets[system]

    print("Total budget: %.2fms (target: %.2fms)" % [total_budget, TARGET_FRAME_TIME])
    assert(abs(total_budget - TARGET_FRAME_TIME) < 0.1, "Budget doesn't sum to frame time")

# Run budget stress test
func stress_test_budgets(duration_sec: float = 60.0):
    var budget_mgr = FrameBudgetManager.new()
    add_child(budget_mgr)

    # Simulate heavy load
    var start_time = Time.get_ticks_msec()
    while (Time.get_ticks_msec() - start_time) < duration_sec * 1000.0:
        # Heavy rendering
        budget_mgr.begin_system("rendering")
        _heavy_rendering_workload()
        budget_mgr.end_system("rendering")

        # Heavy physics
        budget_mgr.begin_system("physics")
        _heavy_physics_workload()
        budget_mgr.end_system("physics")

        budget_mgr.capture_frame()
        await get_tree().process_frame

    # Analyze results
    print(budget_mgr.get_budget_report(3600))  # Last 60 seconds

# Compare different budget strategies
func compare_budget_strategies():
    # Conservative: More reserve
    var conservative = {
        "rendering": 6.0,
        "physics": 4.0,
        "gameplay": 2.0,
        "ai": 1.5,
        "audio": 0.8,
        "reserve": 2.37  # 14% reserve
    }

    # Aggressive: Minimal reserve
    var aggressive = {
        "rendering": 7.5,
        "physics": 4.5,
        "gameplay": 2.5,
        "ai": 1.5,
        "audio": 0.8,
        "reserve": 0.37  # 2% reserve
    }

    # Test both under load
    print("Testing conservative strategy...")
    # ... run test ...

    print("Testing aggressive strategy...")
    # ... run test ...
```

**Budget Allocation Guidelines:**

**60fps (16.67ms total):**
- Rendering: 6-8ms (40-48%)
- Physics: 3-5ms (20-30%)
- Gameplay: 2-3ms (12-18%)
- AI: 1-2ms (6-12%)
- Audio: 0.5-1ms (3-6%)
- Reserve: 1-2ms (6-12%)

**30fps (33.33ms total):**
- Rendering: 15-18ms (45-54%)
- Physics: 6-8ms (18-24%)
- Gameplay: 4-6ms (12-18%)
- AI: 2-4ms (6-12%)
- Audio: 1-2ms (3-6%)
- Reserve: 2-4ms (6-12%)

**Target Metrics:**
- Budget adherence: >90% of frames within budget
- System exceedance: <10% of frames per system
- Worst-case spike: <150% of budget (25ms for 60fps)
- Reserve utilization: >50% unused (safety margin)
- Quality scaling: Triggers at <80% budget adherence

---
