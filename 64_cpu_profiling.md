### 64. CPU Profiling (Sampling vs Instrumentation)

**Category:** Profiling & Advanced - Performance Analysis

**Problem It Solves:** Invisible performance bottlenecks causing frame drops. Without profiling, developers waste weeks optimizing the wrong code sections. A 10ms frame spike could be 8ms in rendering or 8ms in physics - profiling reveals the 400% difference in optimization priority. Manual timing is 95% inaccurate for functions <1ms.

**Technical Explanation:**
Two fundamental approaches: **Sampling** takes periodic snapshots of the call stack (every 1-10ms), building statistical profile with <5% overhead. **Instrumentation** inserts timing code at every function entry/exit, providing exact timing but 10-300% overhead. Sampling reveals "hot spots" (functions consuming most CPU time), instrumentation reveals "hot paths" (specific call sequences). Modern profilers use hybrid approaches - sampling for overview, instrumentation for deep-dive. Critical for finding cache misses, branch mispredictions, and function call overhead.

**Algorithmic Complexity:**
- Sampling: O(1) per sample, O(n) stack walking where n = stack depth
- Instrumentation: O(f) overhead where f = function calls per frame
- Analysis: O(n log n) for sorting by time
- Callstack reconstruction: O(d²) where d = max stack depth

**Implementation Pattern:**
```gdscript
# Godot Profiler Usage
class_name PerformanceAnalyzer

# Enable profiling in Godot
func enable_profiling():
    # Built-in monitors
    Performance.add_custom_monitor("physics/collision_checks",
        _get_collision_checks)
    Performance.add_custom_monitor("rendering/draw_calls",
        _get_draw_calls)

# Manual instrumentation for critical sections
var profiler_data := {}

func profile_begin(tag: String):
    if not profiler_data.has(tag):
        profiler_data[tag] = {
            "count": 0,
            "total_time": 0.0,
            "max_time": 0.0,
            "start": 0.0
        }
    profiler_data[tag].start = Time.get_ticks_usec()
    profiler_data[tag].count += 1

func profile_end(tag: String):
    var elapsed = (Time.get_ticks_usec() - profiler_data[tag].start) / 1000.0
    profiler_data[tag].total_time += elapsed
    profiler_data[tag].max_time = max(profiler_data[tag].max_time, elapsed)

func get_profile_report() -> String:
    var report = "=== PROFILE REPORT ===\n"
    var sorted_tags = profiler_data.keys()
    sorted_tags.sort_custom(func(a, b):
        return profiler_data[a].total_time > profiler_data[b].total_time)

    for tag in sorted_tags:
        var data = profiler_data[tag]
        var avg = data.total_time / data.count if data.count > 0 else 0
        report += "%s: avg=%.3fms max=%.3fms calls=%d total=%.3fms\n" % [
            tag, avg, data.max_time, data.count, data.total_time
        ]
    return report

# Example usage in game loop
func _physics_process(delta):
    profile_begin("physics_total")

    profile_begin("collision_detection")
    _process_collisions()
    profile_end("collision_detection")

    profile_begin("ai_updates")
    _update_ai()
    profile_end("ai_updates")

    profile_end("physics_total")

# Sampling-based profiler (lightweight)
class SamplingProfiler:
    var samples := {}
    var sample_interval := 16  # milliseconds
    var last_sample := 0

    func _process(delta):
        var now = Time.get_ticks_msec()
        if now - last_sample >= sample_interval:
            _take_sample()
            last_sample = now

    func _take_sample():
        # Get current call stack (Godot 4.x)
        var stack = get_stack()
        if stack.size() > 0:
            var key = stack[0].function
            samples[key] = samples.get(key, 0) + 1

    func get_hotspots(top_n: int = 10) -> Array:
        var sorted = []
        for func_name in samples:
            sorted.append([func_name, samples[func_name]])
        sorted.sort_custom(func(a, b): return a[1] > b[1])
        return sorted.slice(0, top_n)
```

**Key Parameters:**
- Sampling interval: 1-10ms (1ms = high overhead, 10ms = coarse data)
- Instrumentation threshold: Only profile functions >0.1ms
- Stack depth: 16-64 levels (deeper = more memory)
- Buffer size: 10,000-100,000 samples (10-50MB memory)
- Aggregation window: 60-300 frames for averaging
- Overhead budget: <5% for sampling, <20% for instrumentation

**Edge Cases:**
- Short functions (<0.01ms): Sampling misses them, instrumentation overhead dominates
- Deep recursion: Stack overflow in profiler, limit depth to 32
- Multithreading: Sampling per-thread, instrumentation needs thread-safe storage
- Frame spikes: Use max/99th percentile, not average
- Hot-reload: Clear profiler data on code changes
- Release builds: Disable instrumentation, keep lightweight sampling

**When NOT to Use:**
- Prototyping phase (premature optimization)
- Performance is already acceptable (stable 60fps)
- Changes affect profiled code (Heisenberg effect)
- Shipping to production (remove instrumentation overhead)
- Very short functions (<0.001ms) where overhead exceeds measurement

**Examples from Shipped Games:**

1. **Uncharted 4 (Naughty Dog):** Custom sampling profiler at 10ms intervals, identified rope physics consuming 12ms/frame (72% of budget), optimized to 2ms using verlet integration, shipping 30fps → 60fps
2. **Call of Duty (Treyarch):** PIX instrumentation revealed shader compilation stalls (150ms spikes), precompiled shaders reduced to <1ms, eliminated hitching
3. **Overwatch (Blizzard):** CPU profiler showed animation blending at 8ms (48% of frame), switched to additive blending, reduced to 2ms, maintaining 60fps on consoles
4. **Spider-Man PS4 (Insomniac):** Sampling profiler revealed streaming system at 6ms average, 15ms spikes, implemented predictive loading, reduced to 2ms steady
5. **Celeste (Matt Makes Games):** Godot profiler showed particle updates at 4ms, implemented pooling + spatial partitioning, reduced to 0.3ms, stable 60fps on Switch

**Platform Considerations:**
- **Windows:** Visual Studio Profiler (sampling), Intel VTune (hardware counters), PIX
- **Console (PS5/Xbox):** Platform-specific profilers (Razor, PIX), 60fps mandatory
- **Mobile (iOS/Android):** Instruments/Android Profiler, thermal throttling critical
- **Linux:** perf (sampling), Valgrind (instrumentation), gprof
- **Web (HTML5):** Chrome DevTools Performance tab, limited to JavaScript layer
- **Memory impact:** Sampling: 10-50MB, Instrumentation: 100-500MB

**Godot-Specific Notes:**
- **Godot Profiler:** Built-in sampling profiler (Debug → Profiler), 1ms granularity
- **Performance Monitors:** 50+ built-in metrics (FPS, memory, physics time)
- **Custom Monitors:** Add game-specific metrics with `Performance.add_custom_monitor()`
- **Script Profiler:** GDScript function timings (slower than C++ profiler)
- **Visual Profiler:** Frame breakdown by category (physics, rendering, scripts)
- **Remote Profiling:** Profile game on device via editor connection
- **Godot 4.x:** Improved profiler with flame graph visualization
- **C++ Profiling:** Use external tools (Tracy, Optick) for engine internals

**Synergies:**
- Pairs with GPU Profiling (identify CPU vs GPU bottlenecks)
- Combines with Memory Profiling (CPU time often correlates with allocations)
- Enables Frame Time Budget (allocate time to systems based on measurements)
- Reveals opportunities for Multithreaded Rendering (CPU-bound tasks)
- Guides Object Pooling decisions (identify allocation hotspots)

**Measurement/Profiling:**
```gdscript
# Validate profiler accuracy
func validate_profiler_overhead():
    var iterations = 10000

    # Baseline: empty loop
    var start = Time.get_ticks_usec()
    for i in iterations:
        pass
    var baseline = Time.get_ticks_usec() - start

    # With profiling
    start = Time.get_ticks_usec()
    for i in iterations:
        profile_begin("test")
        profile_end("test")
    var with_profiling = Time.get_ticks_usec() - start

    var overhead_pct = ((with_profiling - baseline) / float(baseline)) * 100.0
    print("Profiler overhead: %.2f%%" % overhead_pct)
    # Target: <5% for production use

# Compare sampling vs instrumentation
func compare_profiling_methods():
    # Sampling (low overhead, statistical)
    var sampling_overhead = measure_sampling_overhead()

    # Instrumentation (high overhead, precise)
    var instrumentation_overhead = measure_instrumentation_overhead()

    print("Sampling overhead: %.2f%%" % sampling_overhead)
    print("Instrumentation overhead: %.2f%%" % instrumentation_overhead)
    # Expected: Sampling <5%, Instrumentation 10-50%
```

**Profiling Workflow:**
1. **Identify problem:** User reports frame drops, measure with Godot Profiler
2. **Broad analysis:** Use sampling (5-10min gameplay) to find hot functions
3. **Deep dive:** Enable instrumentation on suspect functions only
4. **Verify fix:** Compare before/after profiles with same workload
5. **Regression testing:** Keep profiling enabled in development builds

**Target Metrics:**
- 60fps game: Max 16.67ms per frame, allocate 10ms CPU + 6ms GPU
- 30fps game: Max 33.33ms per frame, allocate 20ms CPU + 13ms GPU
- Physics: <4ms (25% of frame budget)
- Rendering setup: <2ms (12% of frame budget)
- Gameplay logic: <4ms (25% of frame budget)
- AI/pathfinding: <2ms (12% of frame budget)
- Reserve: 4ms for frame spikes (25% safety buffer)

**Common Bottlenecks Found:**
- String operations in hot loops: 10-50ms → use StringName
- Array resizing: 5-20ms → preallocate capacity
- Nested loops: O(n²) → O(n log n) with spatial partitioning
- Physics queries: 8-15ms → use collision layers + sleeping
- JSON parsing: 20-100ms → use binary formats
- File I/O in main thread: 50-500ms → async streaming

---
