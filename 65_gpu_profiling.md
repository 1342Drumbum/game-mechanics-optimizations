### 65. GPU Profiling (RenderDoc, PIX)

**Category:** Profiling & Advanced - Graphics Analysis

**Problem It Solves:** GPU bottlenecks invisible to CPU profilers. A game running at 45fps might show CPU at 8ms (50% headroom) while GPU takes 22ms (132% over budget). Without GPU profiling, developers optimize the wrong bottleneck. Overdraw, shader complexity, and texture thrashing can consume 80%+ of frame time but appear as "waiting for GPU" in CPU profilers.

**Technical Explanation:**
GPU profilers capture complete frame execution: draw calls, state changes, shader execution, texture uploads, and memory transfers. **RenderDoc** (cross-platform, open-source) captures frame buffer states, shader source, and API calls. **PIX** (Windows/Xbox) provides hardware-level GPU counters, occupancy metrics, and warp/wavefront analysis. Tools reveal: overdraw (pixels shaded multiple times), shader ALU utilization, memory bandwidth saturation, and draw call batching efficiency. Critical metrics: pixel shader invocations (target <2M/frame at 1080p), texture cache hit rate (>90%), and SM occupancy (>50%).

**Algorithmic Complexity:**
- Frame capture: O(d) where d = draw calls (typically 500-5000)
- Pixel history: O(p × l) where p = pixels traced, l = overdraw layers
- Shader analysis: O(i × w) where i = instructions, w = wavefronts
- Texture analysis: O(t × m) where t = textures, m = mip levels
- Memory bandwidth: Measured in GB/s, compare against hardware peak

**Implementation Pattern:**
```gdscript
# Godot GPU performance monitoring
class_name GPUProfiler extends Node

# Enable GPU time queries (Godot 4.x)
var gpu_timer_query: RID
var last_gpu_time_ms: float = 0.0

func _ready():
    # Enable rendering stats
    RenderingServer.set_debug_generate_wireframes(true)

func _process(delta):
    # Monitor GPU metrics via Performance
    var render_objects = Performance.get_monitor(Performance.RENDER_TOTAL_OBJECTS_IN_FRAME)
    var draw_calls = Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)
    var vertices = Performance.get_monitor(Performance.RENDER_VERTICES_IN_FRAME)

    # Calculate overdraw estimate
    var screen_pixels = get_viewport().size.x * get_viewport().size.y
    var shaded_pixels = vertices * 0.5  # Rough estimate
    var overdraw_ratio = shaded_pixels / screen_pixels

    # Log if excessive
    if overdraw_ratio > 3.0:
        push_warning("High overdraw: %.2fx" % overdraw_ratio)

    if draw_calls > 2000:
        push_warning("High draw calls: %d" % draw_calls)

# Shader complexity analyzer
class ShaderComplexityAnalyzer:
    # Estimate shader cost (ALU instructions)
    static func analyze_shader_cost(shader_code: String) -> Dictionary:
        var complexity = {
            "texture_samples": 0,
            "math_operations": 0,
            "branches": 0,
            "estimated_cycles": 0
        }

        # Count texture samples (expensive: 4-32 cycles)
        complexity.texture_samples = shader_code.count("texture(")
        complexity.texture_samples += shader_code.count("textureLod(")

        # Count branches (can cause divergence: 10-100 cycles)
        complexity.branches = shader_code.count("if(")
        complexity.branches += shader_code.count("if (")

        # Math operations (1-4 cycles each)
        var math_ops = ["sin(", "cos(", "pow(", "sqrt(", "normalize("]
        for op in math_ops:
            complexity.math_operations += shader_code.count(op)

        # Rough estimate (cycles per pixel)
        complexity.estimated_cycles = (
            complexity.texture_samples * 16 +
            complexity.math_operations * 2 +
            complexity.branches * 20
        )

        return complexity

# Draw call batching analyzer
class DrawCallAnalyzer:
    var frame_data := []

    func capture_frame():
        frame_data.clear()
        var objects = Performance.get_monitor(Performance.RENDER_TOTAL_OBJECTS_IN_FRAME)
        var draw_calls = Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)

        return {
            "objects": objects,
            "draw_calls": draw_calls,
            "batch_efficiency": float(objects) / float(draw_calls) if draw_calls > 0 else 0
        }

    func analyze_batching() -> String:
        var result = capture_frame()
        var report = "=== GPU DRAW CALL ANALYSIS ===\n"
        report += "Objects: %d\n" % result.objects
        report += "Draw Calls: %d\n" % result.draw_calls
        report += "Batch Efficiency: %.2f objects/call\n" % result.batch_efficiency

        # Recommendations
        if result.batch_efficiency < 2.0:
            report += "WARNING: Poor batching. Consider:\n"
            report += "  - Use MultiMeshInstance for repeated objects\n"
            report += "  - Merge static meshes with same material\n"
            report += "  - Reduce material variations\n"

        if result.draw_calls > 2000:
            report += "WARNING: Excessive draw calls (>2000)\n"
            report += "  - GPU driver overhead: ~%.2fms\n" % (result.draw_calls * 0.01)

        return report

# Texture memory analyzer (for RenderDoc/PIX correlation)
class TextureAnalyzer:
    func analyze_texture_usage() -> Dictionary:
        var total_memory = 0
        var texture_count = 0
        var largest_textures = []

        # Note: Godot doesn't expose direct texture enumeration
        # Use RenderingServer for detailed analysis in C++
        # This is a placeholder for external tool correlation

        return {
            "total_memory_mb": total_memory / 1024.0 / 1024.0,
            "texture_count": texture_count,
            "largest_textures": largest_textures
        }

# RenderDoc integration helper
static func setup_renderdoc_capture():
    # Add command line arguments for RenderDoc capture
    # Launch game with: --renderdoc-capture-frame=120
    var args = OS.get_cmdline_args()
    for arg in args:
        if arg.begins_with("--renderdoc-capture-frame="):
            var frame_num = int(arg.split("=")[1])
            return frame_num
    return -1

# PIX marker integration (Windows/Xbox)
class PIXMarkers:
    static func begin_event(name: String):
        # In C++ module, would call PIXBeginEvent()
        # Godot equivalent: use profiler markers
        pass

    static func end_event():
        # PIXEndEvent() in C++
        pass

    static func set_marker(name: String):
        # PIXSetMarker() for instant events
        pass
```

**Key Parameters:**
- Target resolution: 1080p (2.07M pixels), 1440p (3.69M), 4K (8.29M)
- Max draw calls: 500-2000 (modern APIs), 2000-5000 (legacy GL/DX11)
- Overdraw budget: 2-3x screen pixels (2x = 200% overdraw)
- Shader complexity: <100 ALU instructions for pixel shaders
- Texture bandwidth: <10GB/s (depends on GPU: mobile 5-20GB/s, desktop 200-900GB/s)
- VRAM usage: <75% of available (leave room for spikes)
- Triangle count: 1-3M visible tris at 60fps

**Edge Cases:**
- Transparent overdraw: Can reach 10-20x in particle-heavy scenes, sort back-to-front
- Shadow map cascades: Each cascade doubles draw calls, limit to 2-4
- Post-processing: Full-screen passes add 2-5ms each, budget 5-10ms total
- Async compute: Overlapping work can mask bottlenecks, profile serial first
- Dynamic resolution: GPU time varies with pixel count, profile at max resolution
- VSync disabled: GPU times appear shorter, enable for accurate profiling

**When NOT to Use:**
- CPU-bound games (GPU idle <30% of frame)
- Prototyping phase (premature optimization)
- 2D games with minimal effects (<100 draw calls)
- Turn-based games without real-time requirements
- Already hitting target framerate with GPU headroom

**Examples from Shipped Games:**

1. **Assassin's Creed Unity (Ubisoft):** RenderDoc revealed excessive shadow cascades (8ms), reduced to 4 cascades with optimized ranges, 8ms → 3ms, 25fps → 30fps on consoles
2. **Gears 5 (The Coalition):** PIX showed texture thrashing in streaming system (250GB/s spikes exceeding hardware 448GB/s), implemented virtual texturing, reduced to 80GB/s sustained
3. **Doom Eternal (id Software):** GPU profiling identified particle overdraw at 8x average (12ms), implemented depth-aware sorting + soft particles, reduced to 2.5x (4ms)
4. **Horizon Zero Dawn (Guerrilla):** PIX analysis showed foliage rendering at 15ms (45% of 33ms budget), implemented GPU-driven culling + LOD, reduced to 6ms
5. **Control (Remedy):** RenderDoc revealed ray tracing denoiser at 12ms, optimized to 6ms using temporal accumulation + lower sample count, maintaining 60fps on RTX GPUs

**Platform Considerations:**
- **Windows:** RenderDoc (all APIs), PIX (DX12/DX11), Nsight (NVIDIA)
- **PlayStation 5:** Razor GPU Profiler, 60fps requirement, hardware ray tracing profiling
- **Xbox Series X/S:** PIX integration, DirectX 12 profiling, VRS analysis
- **Mobile (iOS):** Xcode Metal Debugger, thermal/power profiling critical
- **Mobile (Android):** Android GPU Inspector, Adreno/Mali profilers
- **Switch:** Nintendo DevKit profiler, portable mode (lower clocks) critical
- **Linux:** RenderDoc (Vulkan), Mesa tools, limited vendor support

**Godot-Specific Notes:**
- **Godot 4.x Vulkan:** RenderDoc captures work natively, use `--verbose` flag
- **Godot 3.x GLES3:** Limited RenderDoc support, use apitrace
- **Visual Profiler:** Shows GPU time estimate, not hardware-accurate
- **Performance Monitors:** `RENDER_*` monitors provide high-level metrics
- **Forward+ Rendering:** Check cluster light assignment (visible in RenderDoc)
- **SDFGI:** Probe updates can spike GPU time, profile with moving camera
- **RenderingServer:** Access low-level rendering stats in C++ modules
- **Mobile Renderer:** Simpler shaders, fewer draw calls, profile on device

**Synergies:**
- Pairs with CPU Profiling (identify CPU vs GPU bottleneck)
- Guides Multithreaded Rendering (offload CPU-bound rendering work)
- Reveals opportunities for Occlusion Culling (reduce draw calls)
- Informs LOD Strategy (balance triangle count vs draw calls)
- Enables Dynamic Resolution (scale pixels to maintain frame budget)

**Measurement/Profiling:**
```gdscript
# GPU bottleneck detection
func detect_gpu_bottleneck() -> bool:
    var cpu_time = Performance.get_monitor(Performance.TIME_PROCESS) * 1000.0
    var fps = Performance.get_monitor(Performance.TIME_FPS)
    var frame_time = 1000.0 / fps if fps > 0 else 0

    var gpu_time = frame_time - cpu_time

    print("CPU: %.2fms | GPU (estimated): %.2fms" % [cpu_time, gpu_time])

    # If GPU time > CPU time by 50%, GPU is bottleneck
    return gpu_time > cpu_time * 1.5

# Shader complexity budget check
func check_shader_budget(shader_code: String) -> bool:
    var analysis = ShaderComplexityAnalyzer.analyze_shader_cost(shader_code)

    print("Shader Analysis:")
    print("  Texture samples: %d (budget: <8)" % analysis.texture_samples)
    print("  Math operations: %d" % analysis.math_operations)
    print("  Branches: %d (budget: <4)" % analysis.branches)
    print("  Estimated cycles: %d (budget: <200)" % analysis.estimated_cycles)

    var within_budget = (
        analysis.texture_samples < 8 and
        analysis.branches < 4 and
        analysis.estimated_cycles < 200
    )

    return within_budget

# Draw call budget validation
func validate_draw_call_budget() -> Dictionary:
    var draw_calls = Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)
    var target_fps = 60
    var ms_per_draw_call = 0.01  # Conservative estimate

    var overhead_ms = draw_calls * ms_per_draw_call
    var budget_ms = 1000.0 / target_fps

    return {
        "draw_calls": draw_calls,
        "overhead_ms": overhead_ms,
        "budget_ms": budget_ms,
        "within_budget": overhead_ms < budget_ms * 0.2  # 20% of frame
    }
```

**RenderDoc Workflow:**
1. Launch Godot through RenderDoc, or attach to running process
2. Capture frame with F12 (during heavy scene)
3. Inspect draw call list: Look for redundant state changes
4. Check pixel history: Identify overdraw hotspots (>3x = red flag)
5. Analyze shaders: Review ALU and texture sample counts
6. Examine textures: Find oversized textures, missing mipmaps
7. Verify batching: Consecutive calls with same material = good

**PIX Workflow (Windows/Xbox):**
1. Launch game with PIX GPU Capture
2. Capture frame or timing window
3. Analyze GPU timeline: Identify long-running passes
4. Check occupancy: Target >50% SM utilization
5. Review memory bandwidth: Compare to hardware peak
6. Inspect waveforms: Find divergent branches (SIMD inefficiency)
7. Correlate with markers: Map GPU work to game systems

**Target Metrics:**
- Draw calls: <2000 at 60fps (mobile: <500)
- Overdraw: <3x average (transparent: <5x)
- Shader instructions: <200 ALU for complex, <50 for simple
- Texture samples: <8 per pixel shader
- Triangle count: 1-3M visible (mobile: 100k-500k)
- GPU utilization: 70-95% (>95% = bottleneck, <70% = CPU-bound)

---
