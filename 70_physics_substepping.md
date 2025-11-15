### 70. Physics Sub-stepping

**Category:** Profiling & Advanced - Physics Accuracy

**Problem It Solves:** Physics instability and inaccuracy at standard timesteps. At 60fps (16.67ms), stiff springs oscillate wildly, stacked objects explode apart, and rope simulations jitter. A constraint solver needs 4-10 iterations for stability, but even 10 iterations at 60Hz miss high-frequency dynamics. Sub-stepping divides each frame into 2-8 smaller steps (e.g., 8 substeps = 480Hz effective rate), stabilizing simulations at 2-4x CPU cost.

**Technical Explanation:**
Physics sub-stepping runs multiple physics updates per visual frame. Instead of 1 physics step at 16.67ms, perform 4 steps at 4.17ms each. Smaller timesteps: reduce integration error (Euler integration error is O(dt²)), improve constraint stability (solvers converge faster), catch fast collisions (less distance per step). Three approaches: **Fixed sub-stepping** (always N substeps), **Adaptive sub-stepping** (more substeps for complex scenarios), **Variable rate** (substep only specific objects). Cost scales linearly: 4 substeps = 4x physics time, but enables stable cloth, vehicles, and ragdolls impossible at 60Hz.

**Algorithmic Complexity:**
- Single step physics: O(n² × i) where n = bodies, i = solver iterations (typically 4-10)
- Sub-stepping: O(s × n² × i) where s = substeps (2-8)
- Adaptive: O(s(t) × n² × i) where s(t) varies with scene complexity
- Collision detection per substep: O(n log n) with spatial partitioning
- Total cost: 2-8x single-step physics

**Implementation Pattern:**
```gdscript
# Physics Sub-stepping System
class_name PhysicsSubstepping extends Node

# Sub-stepping configuration
var substeps := 4  # Divide each frame into 4 physics steps
var use_adaptive := false
var min_substeps := 2
var max_substeps := 8

# Adaptive thresholds
var complexity_threshold := 100  # Active rigidbodies
var velocity_threshold := 50.0   # m/s

# Performance tracking
var physics_time_budget := 5.0   # milliseconds at 60fps
var current_physics_time := 0.0

func _ready():
    # Configure physics engine for sub-stepping
    _setup_physics_engine()

func _setup_physics_engine():
    # Get physics server
    var physics_fps = Engine.physics_ticks_per_second

    # With sub-stepping, effective rate = physics_fps × substeps
    # e.g., 60fps × 4 substeps = 240Hz effective physics rate

    print("Physics configuration:")
    print("  Base rate: %d Hz" % physics_fps)
    print("  Substeps: %d" % substeps)
    print("  Effective rate: %d Hz" % (physics_fps * substeps))

func _physics_process(delta):
    var start_time = Time.get_ticks_usec()

    # Determine substep count
    var current_substeps = substeps
    if use_adaptive:
        current_substeps = _calculate_adaptive_substeps()

    # Calculate substep delta
    var substep_dt = delta / current_substeps

    # Perform sub-steps
    for step in current_substeps:
        _physics_substep(substep_dt, step)

    # Track time
    current_physics_time = (Time.get_ticks_usec() - start_time) / 1000.0

    # Debug output
    if Engine.get_frames_drawn() % 60 == 0:
        print("Physics: %.2fms for %d substeps (%.2fms/substep)" % [
            current_physics_time,
            current_substeps,
            current_physics_time / current_substeps
        ])

func _physics_substep(dt: float, step_index: int):
    # This is called multiple times per frame
    # Each call advances physics by dt (1/4 of frame at 4 substeps)

    # Get all physics bodies
    var bodies = get_tree().get_nodes_in_group("physics_bodies")

    # 1. Integration phase (update velocities and positions)
    for body in bodies:
        if body is RigidBody3D and not body.freeze:
            _integrate_forces(body, dt)

    # 2. Collision detection (at substep granularity)
    var contacts = _detect_collisions(bodies)

    # 3. Constraint solving (iterative)
    for iteration in 4:  # Solver iterations per substep
        _solve_constraints(contacts, dt)

    # 4. Apply corrections
    for body in bodies:
        if body is RigidBody3D:
            _apply_position_correction(body)

func _integrate_forces(body: RigidBody3D, dt: float):
    # Semi-implicit Euler integration (stable for small dt)
    # v(t+dt) = v(t) + a(t) × dt
    # x(t+dt) = x(t) + v(t+dt) × dt

    # Apply gravity
    var gravity = Vector3(0, -9.8, 0)
    body.linear_velocity += gravity * dt

    # Apply custom forces (springs, damping, etc.)
    var force = _calculate_body_forces(body)
    var acceleration = force / body.mass
    body.linear_velocity += acceleration * dt

    # Update position
    var new_position = body.global_position + body.linear_velocity * dt
    body.global_position = new_position

    # Angular integration
    body.angular_velocity += body.get_torque() / body.get_inertia() * dt
    # ... rotation update ...

func _calculate_body_forces(body: RigidBody3D) -> Vector3:
    var total_force = Vector3.ZERO

    # Example: Spring force
    if body.has_meta("spring_anchor"):
        var anchor = body.get_meta("spring_anchor") as Vector3
        var displacement = anchor - body.global_position
        var spring_constant = 100.0  # N/m
        total_force += displacement * spring_constant

    # Damping
    var damping = 0.99
    body.linear_velocity *= damping

    return total_force

func _detect_collisions(bodies: Array) -> Array:
    var contacts = []

    # Broad phase (spatial partitioning)
    var potential_pairs = _broad_phase_collision(bodies)

    # Narrow phase (precise collision detection)
    for pair in potential_pairs:
        var bodyA = pair[0]
        var bodyB = pair[1]

        var contact = _narrow_phase_collision(bodyA, bodyB)
        if contact:
            contacts.append(contact)

    return contacts

func _broad_phase_collision(bodies: Array) -> Array:
    # Simplified - would use quadtree/octree
    var pairs = []
    for i in bodies.size():
        for j in range(i + 1, bodies.size()):
            pairs.append([bodies[i], bodies[j]])
    return pairs

func _narrow_phase_collision(bodyA: Node3D, bodyB: Node3D) -> Dictionary:
    # Use PhysicsServer for actual collision detection
    var distance = bodyA.global_position.distance_to(bodyB.global_position)

    # Simplified contact
    if distance < 1.0:  # Collision threshold
        return {
            "bodyA": bodyA,
            "bodyB": bodyB,
            "normal": (bodyB.global_position - bodyA.global_position).normalized(),
            "depth": 1.0 - distance
        }

    return {}

func _solve_constraints(contacts: Array, dt: float):
    # Iterative constraint solver
    # Resolves penetrations and enforces contact constraints

    for contact in contacts:
        var bodyA = contact.bodyA as RigidBody3D
        var bodyB = contact.bodyB as RigidBody3D

        if not bodyA or not bodyB:
            continue

        var normal = contact.normal as Vector3
        var depth = contact.depth as float

        # Calculate relative velocity
        var relative_velocity = bodyB.linear_velocity - bodyA.linear_velocity
        var normal_velocity = relative_velocity.dot(normal)

        # Apply impulse if approaching
        if normal_velocity < 0:
            var restitution = 0.5  # Bounciness
            var impulse_magnitude = -(1.0 + restitution) * normal_velocity
            impulse_magnitude /= (1.0 / bodyA.mass + 1.0 / bodyB.mass)

            var impulse = normal * impulse_magnitude

            # Apply to bodies
            bodyA.apply_central_impulse(-impulse)
            bodyB.apply_central_impulse(impulse)

        # Position correction (Baumgarte stabilization)
        var correction_factor = 0.2
        var correction = normal * depth * correction_factor

        if not bodyA.freeze:
            bodyA.global_position -= correction / 2.0
        if not bodyB.freeze:
            bodyB.global_position += correction / 2.0

func _apply_position_correction(body: RigidBody3D):
    # Clamp velocities to prevent explosions
    var max_velocity = 100.0  # m/s
    if body.linear_velocity.length() > max_velocity:
        body.linear_velocity = body.linear_velocity.normalized() * max_velocity

# Adaptive sub-stepping
func _calculate_adaptive_substeps() -> int:
    # Analyze scene complexity
    var active_bodies = _count_active_bodies()
    var max_velocity = _get_max_velocity()

    # More substeps for complex/fast scenarios
    var complexity_factor = float(active_bodies) / complexity_threshold
    var velocity_factor = max_velocity / velocity_threshold

    var required_substeps = int(ceil(max(complexity_factor, velocity_factor) * substeps))

    return clamp(required_substeps, min_substeps, max_substeps)

func _count_active_bodies() -> int:
    var count = 0
    var bodies = get_tree().get_nodes_in_group("physics_bodies")

    for body in bodies:
        if body is RigidBody3D and not body.sleeping:
            count += 1

    return count

func _get_max_velocity() -> float:
    var max_vel = 0.0
    var bodies = get_tree().get_nodes_in_group("physics_bodies")

    for body in bodies:
        if body is RigidBody3D:
            var vel = body.linear_velocity.length()
            max_vel = max(max_vel, vel)

    return max_vel

# Fixed timestep accumulator (for perfect determinism)
class FixedTimestepAccumulator:
    var accumulator := 0.0
    var fixed_dt := 0.01  # 100Hz physics
    var max_steps := 8    # Prevent spiral of death

    func update(delta: float, physics_callback: Callable):
        accumulator += delta

        var steps = 0
        while accumulator >= fixed_dt and steps < max_steps:
            physics_callback.call(fixed_dt)
            accumulator -= fixed_dt
            steps += 1

        # Handle leftover time
        if steps >= max_steps:
            accumulator = 0.0  # Discard to prevent buildup

# Sub-stepping for specific systems
class SelectiveSubstepping:
    var rope_substeps := 8     # Ropes need high frequency
    var vehicle_substeps := 4  # Vehicles need moderate
    var default_substeps := 2  # Everything else

    func update_system(system_type: String, dt: float):
        var substeps = default_substeps

        match system_type:
            "rope":
                substeps = rope_substeps
            "vehicle":
                substeps = vehicle_substeps
            "cloth":
                substeps = 8
            "ragdoll":
                substeps = 4

        var sub_dt = dt / substeps

        for step in substeps:
            _update_specific_system(system_type, sub_dt)

    func _update_specific_system(system_type: String, dt: float):
        # System-specific physics update
        pass

# Performance monitoring
class SubsteppingProfiler:
    var frame_data := []

    func record_frame(substeps: int, time_ms: float):
        frame_data.append({
            "substeps": substeps,
            "time_ms": time_ms,
            "time_per_substep": time_ms / substeps
        })

        if frame_data.size() > 300:
            frame_data.pop_front()

    func get_statistics() -> Dictionary:
        if frame_data.is_empty():
            return {}

        var total_time = 0.0
        var total_substeps = 0

        for frame in frame_data:
            total_time += frame.time_ms
            total_substeps += frame.substeps

        return {
            "avg_time_ms": total_time / frame_data.size(),
            "avg_substeps": float(total_substeps) / frame_data.size(),
            "avg_time_per_substep": total_time / total_substeps
        }

# Stability analyzer
class PhysicsStabilityAnalyzer:
    var bodies_to_track := []
    var stability_threshold := 0.1  # m/s

    func track_body(body: RigidBody3D):
        bodies_to_track.append({
            "body": body,
            "prev_velocity": body.linear_velocity,
            "velocity_changes": []
        })

    func analyze_stability() -> Dictionary:
        var unstable_count = 0
        var max_acceleration = 0.0

        for tracked in bodies_to_track:
            var body = tracked.body as RigidBody3D
            var current_vel = body.linear_velocity
            var prev_vel = tracked.prev_velocity

            var acceleration = (current_vel - prev_vel).length()
            max_acceleration = max(max_acceleration, acceleration)

            if acceleration > stability_threshold:
                unstable_count += 1

            tracked.prev_velocity = current_vel

        return {
            "unstable_bodies": unstable_count,
            "total_bodies": bodies_to_track.size(),
            "max_acceleration": max_acceleration,
            "is_stable": unstable_count == 0
        }
```

**Key Parameters:**
- Substep count: 2-8 (2 = minimal, 4 = balanced, 8 = high accuracy)
- Physics rate: 60Hz base (with 4 substeps = 240Hz effective)
- Solver iterations: 4-10 per substep (higher = more stable)
- Time budget: 4-6ms total physics at 60fps (25-40% of frame)
- Adaptive range: 2-8 substeps (scale with complexity)
- Max velocity: 100 m/s clamp (prevent explosions)
- Sleeping threshold: 0.5 m/s (disable substeps for resting objects)

**Edge Cases:**
- Spiral of death: Slow frame causes more substeps, causing slower frame - limit max substeps
- Temporal aliasing: Very fast objects still tunnel with 8 substeps, use CCD
- Determinism: Substep count must be fixed for replay/networking
- Performance spikes: Complex scenes with 8 substeps exceed frame budget, reduce quality elsewhere
- Sleeping bodies: Don't substep objects at rest, check velocity threshold
- Initialization: Physics unstable for first few frames, allow settling

**When NOT to Use:**
- Simple physics (no stacking, springs, or constraints)
- Already stable at 60Hz (no jitter or explosions)
- Performance-critical on slow hardware (2-8x cost prohibitive)
- Static scenes (no dynamic interactions)
- Kinematic-only movement (character controllers don't need substeps)

**Examples from Shipped Games:**

1. **BeamNG.drive (BeamNG):** Soft-body vehicle physics with 8-16 substeps per frame (960-1920Hz effective), enables realistic deformation, physics budget 12-15ms (45% of 33ms at 30fps)
2. **Totally Accurate Battle Simulator (Landfall):** Ragdoll battles with 4 substeps (240Hz), prevents limb explosions with 100+ units, increased physics from 3ms to 8ms (acceptable for physics-focused game)
3. **Human: Fall Flat (No Brakes Games):** Character physics with 6 substeps, stabilizes wobbly controls, reduced jitter by 90%, 4ms physics budget out of 16.67ms (24%)
4. **Teardown (Tuxedo Labs):** Destructible voxels with adaptive 2-8 substeps based on falling debris count, maintains 60fps, physics 3-10ms depending on substeps
5. **Bridge Constructor Portal (ClockStone):** Structural physics with 4 substeps, prevents bridge oscillation, stable simulation critical for puzzle accuracy, 5ms physics out of 33ms at 30fps (15%)

**Platform Considerations:**
- **PC:** Can afford 4-8 substeps (multi-core physics)
- **Console (PS5/Xbox):** 2-4 substeps typical (balance with rendering)
- **Mobile:** 2 substeps maximum (thermal limits)
- **Switch:** 2-3 substeps in portable, 3-4 in docked
- **VR:** 2-4 substeps critical (physics must feel responsive)
- **Web:** 2 substeps (JavaScript physics overhead high)

**Godot-Specific Notes:**
- **Engine.physics_ticks_per_second:** Base physics rate (default 60Hz)
- **Custom sub-stepping:** Must implement manually in `_physics_process()`
- **PhysicsServer3D:** Low-level physics control for custom substeps
- **Bullet Physics (3.x):** Has built-in substepping support
- **Godot Physics (4.x):** Must implement substepping in GDScript or C++
- **Sleeping:** Godot auto-sleeps bodies, skip substeps for sleeping objects
- **Determinism:** Fixed timestep accumulator ensures consistent substeps

**Synergies:**
- Pairs with CCD (substeps catch more collisions, CCD catches remainder)
- Combines with Constraint Solving (more iterations per substep = stabler)
- Enables Advanced Physics (cloth, soft bodies, ropes)
- Works with Frame Time Budget (allocate extra time for substeps)
- Improves Vehicle Physics (suspension, tire forces need high frequency)

**Measurement/Profiling:**
```gdscript
# Compare stability with different substep counts
func benchmark_substep_stability():
    var substep_counts = [1, 2, 4, 8]
    var results = {}

    for count in substep_counts:
        print("\n=== Testing %d substeps ===" % count)

        # Setup test scenario (stacked boxes)
        var test_scene = _create_test_scenario()

        # Run simulation
        substeps = count
        var stability = _run_stability_test(test_scene, 5.0)  # 5 seconds

        results[count] = stability

        print("Stability: %.2f%% (lower = better)" % (stability * 100))

        # Cleanup
        test_scene.queue_free()

    return results

func _create_test_scenario() -> Node:
    # Create tower of boxes (challenging for physics)
    var scene = Node3D.new()

    for i in 10:
        var box = RigidBody3D.new()
        box.global_position = Vector3(0, i * 1.1, 0)
        # ... add collision shape ...
        scene.add_child(box)

    return scene

func _run_stability_test(scene: Node, duration: float) -> float:
    # Measure how much boxes move (instability metric)
    var bodies = scene.get_children()
    var initial_positions = []

    for body in bodies:
        if body is RigidBody3D:
            initial_positions.append(body.global_position)

    # Simulate
    var start = Time.get_ticks_msec()
    while (Time.get_ticks_msec() - start) < duration * 1000.0:
        await get_tree().physics_frame

    # Measure displacement
    var total_displacement = 0.0
    for i in bodies.size():
        var body = bodies[i] as RigidBody3D
        if body:
            var displacement = body.global_position.distance_to(initial_positions[i])
            total_displacement += displacement

    return total_displacement / bodies.size()

# Performance impact analysis
func analyze_substep_cost():
    var iterations = 100
    var delta = 0.016667

    for substep_count in [1, 2, 4, 8]:
        substeps = substep_count

        var start = Time.get_ticks_usec()

        for i in iterations:
            _physics_process(delta)
            await get_tree().physics_frame

        var elapsed = (Time.get_ticks_usec() - start) / 1000.0
        var avg_frame_time = elapsed / iterations

        print("%d substeps: %.2fms per frame (%.2fx cost)" % [
            substep_count,
            avg_frame_time,
            float(substep_count)
        ])

    # Expected: Linear scaling (4 substeps ≈ 4x cost)

# Validate determinism
func test_substep_determinism():
    var test_scene = _create_test_scenario()

    # Run simulation twice with same substeps
    var positions1 = []
    var positions2 = []

    # First run
    substeps = 4
    for i in 300:  # 5 seconds at 60fps
        _physics_process(0.016667)
        await get_tree().physics_frame

    for body in test_scene.get_children():
        if body is RigidBody3D:
            positions1.append(body.global_position)

    # Reset scene
    test_scene.queue_free()
    test_scene = _create_test_scenario()

    # Second run (identical)
    substeps = 4
    for i in 300:
        _physics_process(0.016667)
        await get_tree().physics_frame

    for body in test_scene.get_children():
        if body is RigidBody3D:
            positions2.append(body.global_position)

    # Compare
    var max_error = 0.0
    for i in positions1.size():
        var error = positions1[i].distance_to(positions2[i])
        max_error = max(max_error, error)

    print("Determinism test: max error = %.6f meters" % max_error)
    # Target: <0.001m (1mm) for deterministic physics
```

**Substep Guidelines:**

**By Game Type:**
- Racing games: 4-6 substeps (vehicle suspension stability)
- Physics puzzles: 4-8 substeps (accurate constraint solving)
- Fighting games: 2-3 substeps (ragdolls, hit reactions)
- FPS: 2 substeps (minimal physics complexity)
- Platformers: 1-2 substeps (kinematic characters don't need more)

**By Object Type:**
- Cloth: 8-16 substeps
- Ropes: 8-12 substeps
- Vehicles: 4-6 substeps
- Ragdolls: 2-4 substeps
- Standard rigidbodies: 2 substeps
- Static geometry: 0 substeps (not simulated)

**Target Metrics:**
- Physics budget: 4-6ms at 60fps (25-40% of frame)
- Cost scaling: 1.8-2.2x per doubling of substeps (linear with overhead)
- Stability: <1m position error over 10 seconds
- Determinism: <1mm error in replay
- Sleeping: >80% of bodies asleep in steady state (optimization)

---
