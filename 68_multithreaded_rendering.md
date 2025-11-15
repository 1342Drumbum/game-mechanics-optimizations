### 68. Multithreaded Rendering

**Category:** Profiling & Advanced - Threading

**Problem It Solves:** CPU-bound rendering bottlenecks on single core. Modern CPUs have 8-16 cores, but traditional rendering uses only 1, wasting 87-93% of CPU capacity. A game with 3000 draw calls at 0.005ms each = 15ms on single core (90% of 16.67ms budget). Spreading across 4 cores = 3.75ms (22% of budget), enabling 4x more objects or 75% faster framerate.

**Technical Explanation:**
Multithreaded rendering parallelizes CPU-side rendering work across multiple threads. Two architectures: **Command buffer generation** (each thread builds command lists, main thread submits to GPU) and **Full render thread** (separate thread for entire rendering pipeline). Modern approach: **Job system** - divide scene into spatial chunks, parallel threads cull/sort/build commands for each chunk, then merge. Critical challenges: Thread synchronization (mutexes = 0.01-0.1ms overhead), load balancing (uneven work = wasted cores), and data races (shared state). Vulkan/DX12 designed for this (multi-threaded command buffer recording), OpenGL requires workarounds (separate contexts per thread).

**Algorithmic Complexity:**
- Single-threaded: O(n) for n draw calls, sequential processing
- Multi-threaded (ideal): O(n/p) where p = threads (cores)
- Load balancing: O(n/p + sync) where sync = 0.1-1ms overhead
- Work stealing: O(log p) per steal operation (dynamic balancing)
- Command buffer merge: O(p) to combine thread results

**Implementation Pattern:**
```gdscript
# Godot Multithreaded Rendering System
class_name MultithreadedRenderer extends Node

# Godot 4.x uses WorkerThreadPool for parallel tasks
var thread_pool_size := 4
var render_jobs := []

class RenderChunk:
    var bounds: AABB
    var objects: Array[Node3D] = []
    var draw_commands: Array = []
    var thread_id := -1

    func cull_and_sort(camera: Camera3D) -> Array:
        var visible_objects = []
        var frustum = camera.get_frustum()

        # Frustum culling (parallel-safe per chunk)
        for obj in objects:
            if _is_in_frustum(obj, frustum):
                visible_objects.append(obj)

        # Depth sorting for transparency
        visible_objects.sort_custom(func(a, b):
            var dist_a = camera.global_position.distance_squared_to(a.global_position)
            var dist_b = camera.global_position.distance_squared_to(b.global_position)
            return dist_a > dist_b  # Back to front
        )

        return visible_objects

    func _is_in_frustum(obj: Node3D, frustum: Array) -> bool:
        # Simplified frustum test
        var aabb = _get_object_aabb(obj)
        for plane in frustum:
            if plane.is_point_over(aabb.position):
                return false
        return true

    func _get_object_aabb(obj: Node3D) -> AABB:
        if obj is MeshInstance3D:
            return obj.get_aabb()
        return AABB(obj.global_position, Vector3(1, 1, 1))

# Spatial partitioning for parallel rendering
class RenderJobSystem:
    var chunks: Array[RenderChunk] = []
    var world_bounds: AABB

    func _init(bounds: AABB, chunk_size: float = 50.0):
        world_bounds = bounds
        _subdivide_world(chunk_size)

    func _subdivide_world(chunk_size: float):
        # Divide world into grid chunks for parallel processing
        var x_chunks = int(world_bounds.size.x / chunk_size) + 1
        var z_chunks = int(world_bounds.size.z / chunk_size) + 1

        for x in x_chunks:
            for z in z_chunks:
                var chunk = RenderChunk.new()
                chunk.bounds = AABB(
                    world_bounds.position + Vector3(x * chunk_size, 0, z * chunk_size),
                    Vector3(chunk_size, world_bounds.size.y, chunk_size)
                )
                chunks.append(chunk)

    func assign_object(obj: Node3D):
        # Assign object to appropriate chunk(s)
        var obj_pos = obj.global_position

        for chunk in chunks:
            if chunk.bounds.has_point(obj_pos):
                chunk.objects.append(obj)
                break

    func process_chunks_parallel(camera: Camera3D) -> Array:
        var visible_objects = []
        var jobs = []

        # Create job for each chunk
        for chunk in chunks:
            jobs.append(func(): return chunk.cull_and_sort(camera))

        # Execute jobs in parallel using WorkerThreadPool (Godot 4.x)
        var results = []
        for i in jobs.size():
            var job_id = WorkerThreadPool.add_task(jobs[i])
            results.append(job_id)

        # Wait for all jobs and collect results
        for job_id in results:
            var chunk_result = WorkerThreadPool.wait_for_task_completion(job_id)
            visible_objects.append_array(chunk_result)

        return visible_objects

# Thread-safe render command buffer
class RenderCommandBuffer:
    var commands := []
    var mutex := Mutex.new()

    func add_command(cmd: Dictionary):
        mutex.lock()
        commands.append(cmd)
        mutex.unlock()

    func get_commands() -> Array:
        mutex.lock()
        var result = commands.duplicate()
        mutex.unlock()
        return result

    func clear():
        mutex.lock()
        commands.clear()
        mutex.unlock()

# Main multithreaded renderer
var job_system: RenderJobSystem
var command_buffer: RenderCommandBuffer

func _ready():
    # Initialize with world bounds
    var world_size = 1000.0
    job_system = RenderJobSystem.new(
        AABB(Vector3(-world_size/2, -100, -world_size/2),
             Vector3(world_size, 200, world_size)),
        50.0  # Chunk size
    )
    command_buffer = RenderCommandBuffer.new()

    # Populate chunks with scene objects
    _populate_chunks()

func _populate_chunks():
    var objects = get_tree().get_nodes_in_group("renderable")
    for obj in objects:
        if obj is Node3D:
            job_system.assign_object(obj)

func _process(delta):
    # Get camera
    var camera = get_viewport().get_camera_3d()
    if not camera:
        return

    # Parallel culling and sorting
    var start_time = Time.get_ticks_usec()
    var visible_objects = job_system.process_chunks_parallel(camera)
    var cull_time = (Time.get_ticks_usec() - start_time) / 1000.0

    # Build render commands (could also be parallelized)
    start_time = Time.get_ticks_usec()
    _build_render_commands(visible_objects)
    var build_time = (Time.get_ticks_usec() - start_time) / 1000.0

    # Submit to GPU (main thread only)
    start_time = Time.get_ticks_usec()
    _submit_commands()
    var submit_time = (Time.get_ticks_usec() - start_time) / 1000.0

    # Debug output
    if Engine.get_frames_drawn() % 60 == 0:
        print("MT Render: cull=%.2fms build=%.2fms submit=%.2fms objects=%d" % [
            cull_time, build_time, submit_time, visible_objects.size()
        ])

func _build_render_commands(objects: Array):
    command_buffer.clear()

    for obj in objects:
        # Build command for each object
        var cmd = {
            "type": "draw_mesh",
            "mesh": obj.mesh if obj is MeshInstance3D else null,
            "transform": obj.global_transform,
            "material": obj.get_surface_override_material(0) if obj is MeshInstance3D else null
        }
        command_buffer.add_command(cmd)

func _submit_commands():
    # In real implementation, would submit to RenderingServer
    # This is simplified for demonstration
    var commands = command_buffer.get_commands()

    # Batch commands by material (reduce state changes)
    var batched = _batch_by_material(commands)

    # Submit to GPU (single-threaded, GPU API requirement)
    for batch in batched:
        _submit_batch(batch)

func _batch_by_material(commands: Array) -> Array:
    var batches := {}

    for cmd in commands:
        var material_id = cmd.material.get_instance_id() if cmd.material else 0
        if not batches.has(material_id):
            batches[material_id] = []
        batches[material_id].append(cmd)

    return batches.values()

func _submit_batch(batch: Array):
    # Actual rendering would happen here
    # In Godot, this is handled by RenderingServer
    pass

# Advanced: Work-stealing job scheduler
class WorkStealingScheduler:
    var thread_count: int
    var job_queues: Array = []  # Per-thread queue
    var mutexes: Array = []

    func _init(threads: int):
        thread_count = threads
        for i in threads:
            job_queues.append([])
            mutexes.append(Mutex.new())

    func add_job(thread_id: int, job: Callable):
        mutexes[thread_id].lock()
        job_queues[thread_id].append(job)
        mutexes[thread_id].unlock()

    func steal_job(thread_id: int) -> Callable:
        # Try to steal from another thread's queue
        for i in thread_count:
            if i == thread_id:
                continue

            mutexes[i].lock()
            if not job_queues[i].is_empty():
                var job = job_queues[i].pop_back()
                mutexes[i].unlock()
                return job
            mutexes[i].unlock()

        return Callable()  # No job available

    func get_job(thread_id: int) -> Callable:
        # Try own queue first
        mutexes[thread_id].lock()
        if not job_queues[thread_id].is_empty():
            var job = job_queues[thread_id].pop_front()
            mutexes[thread_id].unlock()
            return job
        mutexes[thread_id].unlock()

        # Steal from other threads
        return steal_job(thread_id)

# Performance monitoring
class MTRenderingStats:
    var frame_times := []
    var thread_utilization := []

    func record_frame(cull_time: float, build_time: float, submit_time: float):
        frame_times.append({
            "cull": cull_time,
            "build": build_time,
            "submit": submit_time,
            "total": cull_time + build_time + submit_time
        })

        if frame_times.size() > 300:
            frame_times.pop_front()

    func get_average_times() -> Dictionary:
        if frame_times.is_empty():
            return {}

        var totals = {"cull": 0.0, "build": 0.0, "submit": 0.0, "total": 0.0}

        for frame in frame_times:
            totals.cull += frame.cull
            totals.build += frame.build
            totals.submit += frame.submit
            totals.total += frame.total

        var count = frame_times.size()
        return {
            "cull_avg": totals.cull / count,
            "build_avg": totals.build / count,
            "submit_avg": totals.submit / count,
            "total_avg": totals.total / count
        }

    func calculate_speedup(single_threaded_time: float, multi_threaded_time: float) -> float:
        return single_threaded_time / multi_threaded_time

    func calculate_efficiency(speedup: float, thread_count: int) -> float:
        # Ideal efficiency = 1.0 (perfect scaling)
        return speedup / thread_count
```

**Key Parameters:**
- Thread count: 4-8 threads (match CPU core count, reserve 1-2 for main/physics)
- Chunk size: 25-100 world units (larger = less overhead, smaller = better load balance)
- Job granularity: 0.1-1ms per job (too small = overhead, too large = imbalance)
- Sync overhead budget: <0.5ms total (mutex locks, barriers)
- Work stealing threshold: Steal when queue empty or <10% work remaining
- Command buffer size: 1000-10000 commands per thread (balance memory vs batching)

**Edge Cases:**
- Load imbalance: One chunk with 90% of objects, use work stealing
- Shared state: Material/texture changes require synchronization, batch by material
- GPU submission: Must happen on single thread (OpenGL context affinity)
- Thread creation overhead: Reuse threads via pool, avoid creating per-frame
- Cache coherency: False sharing on mutex-protected data, use padding
- Overdraw with transparency: Must sort globally after parallel culling

**When NOT to Use:**
- <1000 draw calls (overhead exceeds benefit)
- GPU-bound games (CPU idle, threading doesn't help)
- Mobile with <4 cores (insufficient parallelism)
- Simple 2D games (rendering already fast)
- Prototyping phase (adds significant complexity)

**Examples from Shipped Games:**

1. **Destiny 2 (Bungie):** 4-thread rendering: Culling (2 threads, 3ms→1ms), Command gen (1 thread, 4ms→2ms), main thread submission (1ms). Total 8ms→4ms, enabling 2000→4000 draw calls at 30fps
2. **Assassin's Creed Unity (Ubisoft):** 8-thread culling for 10,000+ objects in Paris, single-thread took 25ms, 8-thread reduced to 4ms (6.25x speedup, 78% efficiency)
3. **Battlefield 1 (DICE):** DX12 multi-threaded command buffers, 64 players + destruction: 4 threads recording commands simultaneously, CPU time 12ms→5ms, GPU-bound maintained
4. **Doom Eternal (id Software):** Vulkan multi-threaded recording: 6 worker threads for 3000+ draw calls, culling parallelized across scene tiles, 9ms→3ms (3x speedup)
5. **Forza Horizon 5 (Playground Games):** 8-thread rendering pipeline on Xbox Series X: Parallel culling (4 threads), shadow map generation (2 threads), particle updates (1 thread), main thread (1 thread), total 60fps stable

**Platform Considerations:**
- **PC:** 4-16 cores available, scale thread count with CPU
- **PS5:** 8 cores (7 usable), use 3-4 threads for rendering
- **Xbox Series X/S:** 8 cores, similar to PS5
- **Switch:** 4 cores, use 2 threads max (leave room for main/physics)
- **Mobile:** 4-8 cores, often big.LITTLE architecture, be careful with threading
- **Vulkan/DX12:** Designed for multi-threading, full command buffer parallelism
- **OpenGL:** Limited multi-threading, need separate contexts per thread

**Godot-Specific Notes:**
- **WorkerThreadPool (4.x):** Built-in thread pool for parallel tasks
- **RenderingServer:** Not thread-safe, only call from main thread
- **Godot 4.x:** Automatic multi-threading for some rendering tasks
- **Custom C++ modules:** Required for true parallel command buffer generation
- **Thread class:** Low-level threading, manual management
- **Performance:** GDScript has GIL-like limitations, use C++ for threading
- **Resource loading:** Can be threaded with `ResourceLoader.load_threaded_request()`

**Synergies:**
- Pairs with Frame Time Budget (rendering stays within allocated time)
- Enables higher LOD quality (more CPU budget for detail)
- Combines with Occlusion Culling (parallel culling tests)
- Supports Dynamic Resolution (parallel rendering at multiple resolutions)
- Works with Frustum Culling (parallelized visibility tests)

**Measurement/Profiling:**
```gdscript
# Compare single vs multi-threaded performance
func benchmark_threading():
    var object_counts = [1000, 2000, 5000, 10000]

    for count in object_counts:
        print("\n=== Testing with %d objects ===" % count)

        # Single-threaded
        var start = Time.get_ticks_usec()
        _single_threaded_cull(count)
        var single_time = (Time.get_ticks_usec() - start) / 1000.0

        # Multi-threaded
        start = Time.get_ticks_usec()
        _multi_threaded_cull(count)
        var multi_time = (Time.get_ticks_usec() - start) / 1000.0

        # Calculate metrics
        var speedup = single_time / multi_time
        var efficiency = speedup / thread_pool_size

        print("Single-threaded: %.2fms" % single_time)
        print("Multi-threaded: %.2fms" % multi_time)
        print("Speedup: %.2fx" % speedup)
        print("Efficiency: %.1f%%" % (efficiency * 100.0))

# Measure thread utilization
func measure_thread_utilization(duration_sec: float = 10.0):
    var thread_work_times = []
    for i in thread_pool_size:
        thread_work_times.append(0.0)

    # Run workload
    var start = Time.get_ticks_msec()
    while (Time.get_ticks_msec() - start) < duration_sec * 1000.0:
        # Process frame with timing per thread
        # ... track work time per thread ...
        await get_tree().process_frame

    # Calculate utilization
    var total_time = duration_sec * thread_pool_size
    var total_work = 0.0
    for time in thread_work_times:
        total_work += time

    var utilization = (total_work / total_time) * 100.0
    print("Thread utilization: %.1f%%" % utilization)
    # Target: >80% = good, <60% = poor load balancing

# Validate thread safety
func validate_thread_safety():
    var shared_counter = 0
    var mutex = Mutex.new()
    var threads = []

    # Spawn threads that increment shared counter
    for i in 4:
        var thread = Thread.new()
        thread.start(func():
            for j in 100000:
                mutex.lock()
                shared_counter += 1
                mutex.unlock()
        )
        threads.append(thread)

    # Wait for completion
    for thread in threads:
        thread.wait_to_finish()

    # Validate result
    print("Shared counter: %d (expected: 400000)" % shared_counter)
    assert(shared_counter == 400000, "Thread safety violation detected")
```

**Target Metrics:**
- Speedup: 3-6x on 4-8 cores (75-85% efficiency)
- Thread utilization: >80% (minimal idle time)
- Sync overhead: <0.5ms (5% of rendering budget)
- Load imbalance: <20% difference between busiest/idle threads
- Cache misses: <5% increase vs single-threaded (minimize false sharing)

---
