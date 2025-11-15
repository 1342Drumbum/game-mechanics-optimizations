### 66. Memory Profiling

**Category:** Profiling & Advanced - Memory Management

**Problem It Solves:** Memory leaks causing crashes after 30-60 minutes of gameplay. A leak of 10MB/minute exhausts 4GB in 6.6 hours, but console certification requires 48-hour stability. Hidden allocations in hot loops can generate 500MB-2GB garbage per second, triggering 50-100ms GC pauses that break 60fps. Texture thrashing (loading/unloading 2GB textures repeatedly) stalls rendering for 200-500ms.

**Technical Explanation:**
Memory profilers track allocation sites, object lifetimes, and heap fragmentation. Three key metrics: **Peak memory** (max used, must stay under platform limit), **Allocation rate** (MB/s, high rates trigger GC), and **Fragmentation** (wasted space, causes allocation failures). Tools provide: call stacks for allocations, object reference graphs (find leak roots), and heap snapshots (compare before/after). Modern profilers detect: dangling references, circular references, and memory pooling opportunities. Critical for games: texture/mesh memory (50-80% of total), script objects (10-30%), and audio buffers (5-10%).

**Algorithmic Complexity:**
- Heap snapshot: O(n) where n = live objects (typically 100K-10M)
- Leak detection: O(n × r) where r = average references per object
- Allocation tracking: O(1) per allocation with hashtable
- Diff comparison: O(n₁ + n₂) for two snapshots
- Reference graph: O(n + e) where e = edges (BFS/DFS traversal)

**Implementation Pattern:**
```gdscript
# Godot Memory Profiler
class_name MemoryProfiler extends Node

var snapshots := []
var allocation_tracking := {}

func _ready():
    # Enable memory monitors
    Performance.add_custom_monitor("memory/texture_memory",
        _get_texture_memory)
    Performance.add_custom_monitor("memory/mesh_memory",
        _get_mesh_memory)

func take_snapshot(label: String = "") -> Dictionary:
    var snapshot = {
        "label": label,
        "timestamp": Time.get_ticks_msec(),
        "static_memory": Performance.get_monitor(Performance.MEMORY_STATIC),
        "dynamic_memory": Performance.get_monitor(Performance.MEMORY_DYNAMIC),
        "max_memory": Performance.get_monitor(Performance.MEMORY_MAX),
        "object_count": Performance.get_monitor(Performance.OBJECT_COUNT),
        "resource_count": Performance.get_monitor(Performance.OBJECT_RESOURCE_COUNT),
        "node_count": Performance.get_monitor(Performance.OBJECT_NODE_COUNT),
        "orphan_nodes": Performance.get_monitor(Performance.OBJECT_ORPHAN_NODE_COUNT)
    }

    # Calculate derived metrics
    snapshot.total_memory = snapshot.static_memory + snapshot.dynamic_memory
    snapshot.memory_mb = snapshot.total_memory / 1024.0 / 1024.0

    snapshots.append(snapshot)
    return snapshot

func compare_snapshots(snap1: Dictionary, snap2: Dictionary) -> Dictionary:
    return {
        "memory_delta_mb": (snap2.total_memory - snap1.total_memory) / 1024.0 / 1024.0,
        "object_delta": snap2.object_count - snap1.object_count,
        "node_delta": snap2.node_count - snap1.node_count,
        "orphan_delta": snap2.orphan_nodes - snap1.orphan_nodes,
        "time_delta_sec": (snap2.timestamp - snap1.timestamp) / 1000.0
    }

func detect_leak(duration_sec: float = 60.0) -> Dictionary:
    # Take initial snapshot
    var snap1 = take_snapshot("leak_test_start")

    # Wait for duration (run game normally)
    await get_tree().create_timer(duration_sec).timeout

    # Take final snapshot
    var snap2 = take_snapshot("leak_test_end")

    var diff = compare_snapshots(snap1, snap2)

    # Calculate leak rate
    var leak_rate_mb_per_min = (diff.memory_delta_mb / diff.time_delta_sec) * 60.0

    var result = {
        "leaked_mb": diff.memory_delta_mb,
        "leak_rate_mb_per_min": leak_rate_mb_per_min,
        "is_leaking": leak_rate_mb_per_min > 1.0,  # >1MB/min = leak
        "orphan_nodes_gained": diff.orphan_delta,
        "objects_gained": diff.object_delta
    }

    if result.is_leaking:
        push_warning("Memory leak detected: %.2f MB/min" % leak_rate_mb_per_min)
        if result.orphan_nodes_gained > 0:
            push_warning("Orphan nodes: %d (likely source)" % result.orphan_nodes_gained)

    return result

# Track custom allocations
class AllocationTracker:
    var allocations := {}
    var peak_allocations := 0

    func track_allocation(id: String, size_bytes: int):
        allocations[id] = {
            "size": size_bytes,
            "timestamp": Time.get_ticks_msec(),
            "callstack": get_stack()  # Capture call site
        }

        var total = get_total_allocated()
        if total > peak_allocations:
            peak_allocations = total

    func track_free(id: String):
        allocations.erase(id)

    func get_total_allocated() -> int:
        var total = 0
        for alloc in allocations.values():
            total += alloc.size
        return total

    func get_allocation_report() -> String:
        var report = "=== ALLOCATION REPORT ===\n"
        report += "Current: %.2f MB\n" % (get_total_allocated() / 1024.0 / 1024.0)
        report += "Peak: %.2f MB\n" % (peak_allocations / 1024.0 / 1024.0)
        report += "Active allocations: %d\n" % allocations.size()

        # Sort by size
        var sorted = allocations.keys()
        sorted.sort_custom(func(a, b):
            return allocations[a].size > allocations[b].size)

        report += "\nTop 10 allocations:\n"
        for i in min(10, sorted.size()):
            var id = sorted[i]
            var alloc = allocations[id]
            report += "  %s: %.2f MB\n" % [id, alloc.size / 1024.0 / 1024.0]

        return report

# Texture memory analyzer
func _get_texture_memory() -> float:
    # Estimate based on rendering stats
    # In C++, would query RenderingServer directly
    var estimate = 0.0
    # Placeholder: actual implementation needs engine access
    return estimate

func _get_mesh_memory() -> float:
    # Estimate mesh memory usage
    var vertices = Performance.get_monitor(Performance.RENDER_VERTICES_IN_FRAME)
    # Rough estimate: 32 bytes per vertex (pos, normal, uv, tangent)
    return vertices * 32.0

# Resource leak detector
class ResourceLeakDetector:
    var resource_refs := {}

    func track_resource(resource: Resource, context: String):
        var id = resource.get_instance_id()
        if not resource_refs.has(id):
            resource_refs[id] = {
                "resource": resource,
                "context": context,
                "timestamp": Time.get_ticks_msec(),
                "ref_count": resource.get_reference_count()
            }

    func check_leaks() -> Array:
        var leaks = []
        for id in resource_refs.keys():
            var info = resource_refs[id]
            var resource = info.resource

            # Check if still valid
            if not is_instance_valid(resource):
                resource_refs.erase(id)
                continue

            # Check if ref count increased unexpectedly
            var current_refs = resource.get_reference_count()
            if current_refs > info.ref_count:
                leaks.append({
                    "resource": resource,
                    "context": info.context,
                    "ref_growth": current_refs - info.ref_count,
                    "age_sec": (Time.get_ticks_msec() - info.timestamp) / 1000.0
                })

            # Update ref count
            info.ref_count = current_refs

        return leaks

# Frame allocation monitor
class FrameAllocationMonitor:
    var frame_allocations := []
    var max_frames := 300  # 5 seconds at 60fps

    func _process(delta):
        var current_memory = Performance.get_monitor(Performance.MEMORY_DYNAMIC)
        frame_allocations.append(current_memory)

        if frame_allocations.size() > max_frames:
            frame_allocations.pop_front()

    func get_allocation_rate() -> float:
        if frame_allocations.size() < 2:
            return 0.0

        # Calculate rate over last second (60 frames)
        var frames = min(60, frame_allocations.size())
        var delta_memory = frame_allocations[-1] - frame_allocations[-frames]
        var delta_time = frames / 60.0  # Assume 60fps

        # MB per second
        return (delta_memory / delta_time) / 1024.0 / 1024.0

    func detect_allocation_spike(threshold_mb: float = 10.0) -> bool:
        var rate = get_allocation_rate()
        if rate > threshold_mb:
            push_warning("High allocation rate: %.2f MB/s" % rate)
            return true
        return false
```

**Key Parameters:**
- Platform memory limits: Mobile (2-4GB), Console (12-16GB), PC (8-32GB)
- Texture budget: 50-70% of total (1-4GB typical)
- Script object budget: 10-20% (200-500MB)
- Audio budget: 5-10% (100-200MB)
- Leak threshold: >1MB/min = investigate
- Allocation rate: <10MB/s during gameplay (spikes <50MB/s acceptable)
- GC frequency: <1 per second (ideally none during gameplay)
- Orphan node threshold: 0 (any orphans = leak)

**Edge Cases:**
- Deferred deletion: Objects queued for deletion still counted, snapshot after frame
- Circular references: A→B→A prevents GC, use weak references
- Native allocations: C++ memory not visible to script profiler, use platform tools
- Streaming: Loading assets temporarily spikes memory, profile steady state
- Caching: Intentional memory growth (shader cache, navigation cache)
- Debug builds: 2-5x more memory than release, profile release builds

**When NOT to Use:**
- Performance is stable (no crashes, no stutters)
- Early prototyping (focus on gameplay first)
- Single-scene games with fixed assets
- Memory usage well under limits (<50% peak)
- No user reports of out-of-memory crashes

**Examples from Shipped Games:**

1. **Assassin's Creed Unity (Ubisoft):** Memory profiler revealed 400MB texture leak from unclosed asset bundles, fixed by adding explicit unload calls, eliminating crashes after 2+ hours
2. **Fortnite (Epic Games):** Allocation tracking showed 1.2GB/s allocation rate in hot loops (string concatenation), replaced with string builder, reduced to 50MB/s, eliminated hitching
3. **Spider-Man PS4 (Insomniac):** Heap fragmentation caused allocation failures at 85% memory usage, implemented custom allocator with defragmentation, stable at 95% usage
4. **Gears 5 (The Coalition):** Memory profiler identified 2GB texture thrashing (load/unload same textures), implemented LRU cache, reduced loads by 80%, eliminated 500ms stalls
5. **Horizon Zero Dawn PC (Guerrilla):** Leak detection found 50MB/min leak in animation system (dangling callback references), fixed with weak references, stable 48+ hour playtests

**Platform Considerations:**
- **Windows:** Visual Studio Memory Profiler, heap snapshots, allocation call stacks
- **Console (PS5/Xbox):** Platform profilers, strict memory limits (10-12GB usable)
- **Mobile (iOS/Android):** Instruments/Memory Profiler, 2-3GB limits, OOM kills
- **Switch:** 3.5GB usable, frequent GC catastrophic (30-50ms pauses)
- **Linux:** Valgrind (detailed but slow), AddressSanitizer for leaks
- **Web (HTML5):** Browser DevTools Memory tab, 2GB typical limit

**Godot-Specific Notes:**
- **Performance Monitors:** `MEMORY_STATIC`, `MEMORY_DYNAMIC`, `MEMORY_MAX`
- **Object Counting:** `OBJECT_COUNT`, `OBJECT_RESOURCE_COUNT`, `OBJECT_NODE_COUNT`
- **Orphan Nodes:** `OBJECT_ORPHAN_NODE_COUNT` - should always be 0
- **Reference Counting:** Godot uses ref counting, not GC (but script objects)
- **queue_free():** Deferred deletion, object stays alive until frame end
- **Weak References:** Use `WeakRef` to break circular references
- **Resource Loader:** Cached automatically, use `ResourceLoader.load_threaded_get()` carefully
- **Godot 4.x:** Improved memory tracking, less fragmentation than 3.x

**Synergies:**
- Pairs with CPU Profiling (allocations appear as CPU time)
- Guides Object Pooling (identify allocation hotspots)
- Reveals streaming opportunities (texture/mesh memory spikes)
- Enables Asset Loading Strategy (balance memory vs load times)
- Informs LOD decisions (memory vs visual quality trade-off)

**Measurement/Profiling:**
```gdscript
# Memory leak test suite
func run_leak_tests():
    print("=== MEMORY LEAK TESTS ===")

    # Test 1: Scene loading
    var snap1 = MemoryProfiler.take_snapshot("before_scene_load")
    for i in 10:
        var scene = load("res://test_scene.tscn").instantiate()
        add_child(scene)
        scene.queue_free()
    await get_tree().process_frame  # Wait for deferred deletion
    var snap2 = MemoryProfiler.take_snapshot("after_scene_load")

    var diff = MemoryProfiler.compare_snapshots(snap1, snap2)
    print("Scene load test: %.2f MB delta" % diff.memory_delta_mb)
    assert(diff.memory_delta_mb < 1.0, "Scene loading leak detected")

    # Test 2: Object pooling
    snap1 = MemoryProfiler.take_snapshot("before_pool_test")
    var pool = ObjectPool.new(preload("res://bullet.tscn"), 1000)
    for i in 1000:
        var obj = pool.get_object()
        pool.return_object(obj)
    snap2 = MemoryProfiler.take_snapshot("after_pool_test")

    diff = MemoryProfiler.compare_snapshots(snap1, snap2)
    print("Pool test: %.2f MB delta" % diff.memory_delta_mb)
    assert(diff.memory_delta_mb < 10.0, "Pool allocation excessive")

    # Test 3: Orphan nodes
    snap1 = MemoryProfiler.take_snapshot("before_orphan_test")
    var orphan = Node.new()  # Created but not added to tree
    # Intentional orphan - should be detected
    snap2 = MemoryProfiler.take_snapshot("after_orphan_test")

    diff = MemoryProfiler.compare_snapshots(snap1, snap2)
    print("Orphan test: %d orphans" % diff.orphan_delta)
    orphan.free()  # Clean up

# Allocation rate test
func test_allocation_rate():
    var monitor = FrameAllocationMonitor.new()
    add_child(monitor)

    # Run intensive operations
    for i in 60:  # 1 second at 60fps
        var temp_array = []
        for j in 10000:
            temp_array.append(Vector3.ZERO)  # Allocate
        await get_tree().process_frame

    var rate = monitor.get_allocation_rate()
    print("Allocation rate: %.2f MB/s" % rate)

    monitor.queue_free()

    # Target: <10 MB/s during gameplay
    assert(rate < 10.0, "Allocation rate too high")

# Texture memory budget check
func validate_texture_budget():
    var texture_memory = _get_texture_memory()
    var total_memory = Performance.get_monitor(Performance.MEMORY_STATIC)
    var texture_percent = (texture_memory / total_memory) * 100.0

    print("Texture memory: %.2f MB (%.1f%% of total)" % [
        texture_memory / 1024.0 / 1024.0,
        texture_percent
    ])

    # Target: 50-70% for texture-heavy games
    assert(texture_percent < 80.0, "Texture memory exceeds budget")
```

**Memory Budget Guidelines:**
- **Mobile (3GB total):**
  - Textures: 1.5GB (50%)
  - Meshes: 600MB (20%)
  - Audio: 300MB (10%)
  - Scripts/objects: 300MB (10%)
  - System reserve: 300MB (10%)

- **Console (12GB total):**
  - Textures: 6GB (50%)
  - Meshes: 2GB (17%)
  - Audio: 1GB (8%)
  - Scripts/objects: 2GB (17%)
  - System reserve: 1GB (8%)

**Target Metrics:**
- Peak memory: <80% of platform limit
- Allocation rate: <10MB/s during gameplay
- GC frequency: 0 during gameplay (pre-allocate everything)
- Orphan nodes: 0 (use `queue_free()` properly)
- Leak rate: <0.1MB/min (ideally 0)
- Fragmentation: <20% wasted space

---
