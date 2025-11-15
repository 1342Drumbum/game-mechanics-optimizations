### 12. Job System Multithreading

**Category:** Memory & Data Management

**Problem It Solves:** Single-threaded game loops waste 75-88% of CPU on modern 8-16 core processors. A game running at 60fps on one core (16.67ms frame time) leaves 7 cores completely idle, wasting available hardware. With 8 cores, theoretical maximum is 8x speedup, but typical games only use 1.5-2 cores effectively without a job system, wasting 75-88% of potential performance.

**Technical Explanation:**
Job systems divide work into small independent tasks (jobs) scheduled across worker threads. Main thread submits jobs to shared queue, worker threads pull and execute jobs concurrently. Jobs must be stateless or use thread-safe data. Uses work-stealing algorithm: idle threads steal jobs from busy threads' queues. Supports dependencies via job handles: Job B waits for Job A completion before starting. Lock-free data structures (ring buffers, atomic counters) minimize synchronization overhead. Fiber-based systems (Naughty Dog, Destiny) use cooperative scheduling for even lower overhead (50-100ns context switch vs 5-10μs for threads).

**Algorithmic Complexity:**
- Job submission: O(1) lock-free queue push
- Job execution: O(1) pop from queue
- Work stealing: O(1) steal from another queue
- Synchronization overhead: 100-500ns per job (lock-free) vs 5-10μs (mutex-based)
- Speedup: Near-linear up to core count (Amdahl's law: serial code limits to 5-6x on 8 cores)

**Implementation Pattern:**
```gdscript
class_name JobSystem
extends Node

# Core job system components
var _worker_threads: Array[Thread] = []
var _job_queue: Array[Callable] = []
var _queue_mutex: Mutex = Mutex.new()
var _queue_semaphore: Semaphore = Semaphore.new()
var _running: bool = true
var _active_jobs: int = 0
var _active_jobs_mutex: Mutex = Mutex.new()

const THREAD_COUNT = 7  # Reserve 1 core for main thread

class JobHandle:
    var completed: bool = false
    var mutex: Mutex = Mutex.new()
    var result: Variant = null

func _ready() -> void:
    # Initialize worker threads
    for i in range(THREAD_COUNT):
        var thread = Thread.new()
        thread.start(_worker_thread_func.bind(i))
        _worker_threads.append(thread)
    print("JobSystem: Started %d worker threads" % THREAD_COUNT)

func _worker_thread_func(thread_id: int) -> void:
    # Worker thread main loop
    while _running:
        _queue_semaphore.wait()  # Block until job available

        # Pop job from queue (critical section)
        _queue_mutex.lock()
        if _job_queue.is_empty():
            _queue_mutex.unlock()
            continue
        var job = _job_queue.pop_front()
        _queue_mutex.unlock()

        # Execute job (outside critical section for parallelism)
        if job:
            job.call()

        # Decrement active job count
        _active_jobs_mutex.lock()
        _active_jobs -= 1
        _active_jobs_mutex.unlock()

func schedule_job(job: Callable) -> void:
    # Add job to queue
    _queue_mutex.lock()
    _job_queue.append(job)
    _active_jobs_mutex.lock()
    _active_jobs += 1
    _active_jobs_mutex.unlock()
    _queue_mutex.unlock()

    # Signal worker thread
    _queue_semaphore.post()

func schedule_job_with_handle(job: Callable) -> JobHandle:
    # Job that returns a handle for waiting/result retrieval
    var handle = JobHandle.new()

    var wrapped_job = func():
        var result = job.call()
        handle.mutex.lock()
        handle.result = result
        handle.completed = true
        handle.mutex.unlock()

    schedule_job(wrapped_job)
    return handle

func wait_for_jobs() -> void:
    # Busy-wait until all jobs complete (not optimal but simple)
    while true:
        _active_jobs_mutex.lock()
        var active = _active_jobs
        _active_jobs_mutex.unlock()

        if active == 0:
            break

        OS.delay_usec(10)  # Small delay to reduce CPU spinning

func wait_for_handle(handle: JobHandle) -> Variant:
    # Wait for specific job to complete and return result
    while true:
        handle.mutex.lock()
        if handle.completed:
            var result = handle.result
            handle.mutex.unlock()
            return result
        handle.mutex.unlock()
        OS.delay_usec(10)

func shutdown() -> void:
    # Graceful shutdown
    _running = false

    # Wake all threads so they can exit
    for i in range(THREAD_COUNT):
        _queue_semaphore.post()

    # Wait for threads to finish
    for thread in _worker_threads:
        thread.wait_to_finish()

    print("JobSystem: Shutdown complete")

# Example usage patterns:

# Pattern 1: Batch processing (e.g., update 1000 enemies)
func update_enemies_parallel(enemies: Array, delta: float) -> void:
    const BATCH_SIZE = 128  # 1000 / 8 threads ≈ 125

    for batch_idx in range(0, enemies.size(), BATCH_SIZE):
        var end_idx = min(batch_idx + BATCH_SIZE, enemies.size())

        # Capture variables for closure
        var batch_start = batch_idx
        var batch_end = end_idx

        schedule_job(func():
            # Process batch on worker thread
            for i in range(batch_start, batch_end):
                enemies[i].update(delta)
        )

    # Wait for all enemy updates to complete
    wait_for_jobs()

# Pattern 2: Parallel for with results
func calculate_pathfinding_parallel(units: Array[Unit], targets: Array[Vector3]) -> Array:
    var handles: Array[JobHandle] = []

    for i in range(units.size()):
        var unit = units[i]
        var target = targets[i]

        var handle = schedule_job_with_handle(func():
            # Expensive pathfinding calculation
            return unit.calculate_path_to(target)
        )
        handles.append(handle)

    # Collect results
    var paths = []
    for handle in handles:
        paths.append(wait_for_handle(handle))

    return paths

# Pattern 3: Pipeline with dependencies
func process_rendering_pipeline(scene_data: Dictionary) -> void:
    # Stage 1: Frustum culling (parallel per octree node)
    var culling_handles = []
    for octree_node in scene_data.octree_nodes:
        var handle = schedule_job_with_handle(func():
            return octree_node.frustum_cull(scene_data.camera)
        )
        culling_handles.append(handle)

    # Wait for stage 1
    var visible_objects = []
    for handle in culling_handles:
        visible_objects.append_array(wait_for_handle(handle))

    # Stage 2: Sort by material (parallel per material type)
    var sorting_handles = []
    for material_type in scene_data.material_types:
        var objs = visible_objects.filter(func(o): return o.material_type == material_type)
        var handle = schedule_job_with_handle(func():
            objs.sort_custom(func(a, b): return a.distance < b.distance)
            return objs
        )
        sorting_handles.append(handle)

    # Collect sorted render batches
    var render_batches = []
    for handle in sorting_handles:
        render_batches.append(wait_for_handle(handle))

# Real-world example: Physics simulation
class PhysicsJobified:
    var job_system: JobSystem
    var bodies: Array[RigidBody3D]

    func simulate_physics(delta: float) -> void:
        const BATCH_SIZE = 64

        # Job 1: Integrate velocities (parallel)
        for batch_idx in range(0, bodies.size(), BATCH_SIZE):
            var start = batch_idx
            var end = min(batch_idx + BATCH_SIZE, bodies.size())

            job_system.schedule_job(func():
                for i in range(start, end):
                    bodies[i].velocity += bodies[i].acceleration * delta
                    bodies[i].position += bodies[i].velocity * delta
            )

        job_system.wait_for_jobs()

        # Job 2: Collision detection (parallel with spatial partitioning)
        var collision_pairs = []
        var collision_mutex = Mutex.new()

        for batch_idx in range(0, bodies.size(), BATCH_SIZE):
            var start = batch_idx
            var end = min(batch_idx + BATCH_SIZE, bodies.size())

            job_system.schedule_job(func():
                var local_pairs = []
                for i in range(start, end):
                    for j in range(i + 1, bodies.size()):
                        if bodies[i].aabb.intersects(bodies[j].aabb):
                            local_pairs.append([i, j])

                # Merge results (critical section)
                collision_mutex.lock()
                collision_pairs.append_array(local_pairs)
                collision_mutex.unlock()
            )

        job_system.wait_for_jobs()

        # Job 3: Collision resolution (parallel per pair)
        for pair in collision_pairs:
            job_system.schedule_job(func():
                _resolve_collision(bodies[pair[0]], bodies[pair[1]])
            )

        job_system.wait_for_jobs()
```

**Key Parameters:**
- Thread count: CPU cores - 1 (reserve main thread)
- Job granularity: 50-500μs per job (too small = overhead, too large = imbalance)
- Batch size: Process 64-256 items per job for cache efficiency
- Queue size: 1024-4096 jobs (ring buffer, power of 2)
- Synchronization: Lock-free queues (100-500ns) vs mutex (5-10μs)

**Edge Cases:**
- Race conditions: Jobs must not write to shared data without synchronization
- Deadlocks: Avoid job waiting for another job on same thread
- Thread starvation: Work-stealing prevents idle threads
- Cache thrashing: Jobs accessing same data on different cores (false sharing)
- Overhead: Very small jobs (<10μs) waste time on scheduling

**When NOT to Use:**
- Single-core platforms (mobile low-end devices)
- Already GPU-bound games (CPU parallelism won't help)
- Prototyping phase (adds complexity)
- Work that's already inherently serial (e.g., UI updates)
- Small workloads (<1ms total) where overhead exceeds benefit

**Examples from Shipped Games:**

1. **Destiny (Bungie):** Fiber-based job system utilizes all CPU cores, enabled 30fps on PS3/Xbox 360 with massive open worlds and complex AI, 6-8x performance improvement over single-threaded
2. **Naughty Dog (The Last of Us, Uncharted):** Custom fiber scheduler across 7 SPU cores (PS3), achieved 30fps with industry-leading graphics and physics
3. **Unity DOTS (Job System + Burst):** Near-linear scaling up to 16 cores, enables 100K+ entities at 60fps, 5-10x speedup for ECS systems compared to single-threaded
4. **Unreal Engine TaskGraph:** Parallel-for across all gameplay systems, used by Fortnite to handle 100 players + complex physics on all platforms
5. **Frostbite (DICE - Battlefield series):** Job system enabled 64-player multiplayer with destruction physics on last-gen consoles, 4-6x CPU efficiency improvement

**Platform Considerations:**
- **PC (Desktop):** 8-16 threads typical (8-16 cores), aggressive parallelism, use all cores
- **Consoles (PS5/Xbox):** 16 threads (8 cores with SMT), reserve 1-2 cores for OS, use 12-14 threads
- **Mobile (High-end):** 4-8 threads (4-8 cores), battery considerations, use 3-6 threads
- **Mobile (Low-end):** 2-4 threads, may not be worth overhead, consider single-threaded
- **Web (WASM):** Web Workers support limited, 2-4 threads typical, SharedArrayBuffer required

**Godot-Specific Notes:**
- Godot lacks built-in job system (use manual Thread management)
- WorkerThreadPool exists in Godot 4.x but limited API
- Consider GDExtension for production job system (C++ implementation)
- Physics/rendering already multi-threaded internally
- GDScript threading has GIL-like limitations (use C++ for true parallelism)
- Avoid heavy GDScript in jobs (use C++ GDExtension for compute-heavy tasks)

**Synergies:**
- Essential for ECS/Data-Oriented Design (parallel component processing)
- Pairs with Struct-of-Arrays (cache-friendly batch processing)
- Enables SIMD Vectorization (process 4-8 elements per job)
- Combines with Async Compute (CPU jobs while GPU works)
- Critical for Physics Simulation (parallel collision detection/resolution)

**Measurement/Profiling:**
```gdscript
func benchmark_job_system():
    const WORK_ITEMS = 10000

    # Single-threaded baseline
    var start = Time.get_ticks_usec()
    for i in range(WORK_ITEMS):
        expensive_computation(i)
    var single_time = Time.get_ticks_usec() - start

    # Multi-threaded with job system
    start = Time.get_ticks_usec()
    var batch_size = WORK_ITEMS / THREAD_COUNT
    for thread_id in range(THREAD_COUNT):
        var batch_start = thread_id * batch_size
        var batch_end = batch_start + batch_size
        job_system.schedule_job(func():
            for i in range(batch_start, batch_end):
                expensive_computation(i)
        )
    job_system.wait_for_jobs()
    var multi_time = Time.get_ticks_usec() - start

    var speedup = float(single_time) / float(multi_time)
    print("Speedup: %.2fx (Single: %.2fms, Multi: %.2fms)" % [
        speedup,
        single_time / 1000.0,
        multi_time / 1000.0
    ])

    # Expected speedup: 5-6x on 8 cores (Amdahl's law overhead)

# Monitor thread utilization
func profile_thread_usage():
    var thread_times = []
    for i in range(THREAD_COUNT):
        # Track how much time each thread spends working vs idle
        thread_times.append({"work": 0.0, "idle": 0.0})

    # Analyze load balance
    var max_work = thread_times.map(func(t): return t.work).max()
    var min_work = thread_times.map(func(t): return t.work).min()
    var imbalance = (max_work - min_work) / max_work * 100.0

    if imbalance > 20.0:
        push_warning("Thread load imbalance: %.1f%%" % imbalance)
```

**Target Metrics:**
- Speedup: 5-7x on 8 cores (theoretical max 8x, Amdahl's law limits)
- Thread utilization: >80% (avoid idle threads)
- Job overhead: <5% of frame time
- Load balance: <20% variance between busiest/quietest thread
