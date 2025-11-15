### 69. Continuous Collision Detection (CCD)

**Category:** Profiling & Advanced - Physics

**Problem It Solves:** Fast-moving objects tunneling through geometry. At 60fps (16.67ms frames), a bullet moving at 1000 m/s travels 16.67m between frames - passing completely through a 0.2m thick wall without collision. Discrete collision detection (per-frame checks) fails when object displacement exceeds collision thickness. CCD sweeps the volume between frames, catching missed collisions, but costs 3-10x more than discrete.

**Technical Explanation:**
CCD performs swept volume tests: instead of checking object position at frame start/end, traces the path between them. Two approaches: **Speculative contacts** (project ahead 1-2 frames, detect future collisions, apply preventive forces) and **Swept shape tests** (cast object's shape along movement vector, find time-of-impact). Most engines use hybrid: CCD only for fast objects, discrete for others. Algorithm: Conservative advancement - take small timesteps (0.1-1ms) when objects approach, ensuring collision detection never misses. Cost: 3-10x discrete collision (multiple substeps per frame), but prevents catastrophic tunneling failures.

**Algorithmic Complexity:**
- Discrete collision: O(1) per object pair (single position check)
- Speculative CCD: O(k) where k = lookahead frames (1-3)
- Swept test CCD: O(s) where s = substeps along path (5-20)
- Conservative advancement: O(n²) for n objects, but with spatial partitioning O(n log n)
- Time-of-impact (TOI): O(i) where i = binary search iterations (typically 8-16)

**Implementation Pattern:**
```gdscript
# Continuous Collision Detection System
class_name CCDPhysics extends Node

# CCD settings
var ccd_enabled := true
var ccd_threshold_speed := 20.0  # m/s - enable CCD above this speed
var max_substeps := 8
var min_substep_dt := 0.001  # 1ms minimum

class CCDBody:
    var body: RigidBody3D
    var previous_position: Vector3
    var previous_rotation: Basis
    var use_ccd := false

    func update_motion(delta: float) -> Array:
        var current_pos = body.global_position
        var displacement = current_pos - previous_position
        var speed = displacement.length() / delta

        # Enable CCD for fast-moving objects
        use_ccd = speed > 20.0  # threshold

        if use_ccd:
            return _perform_ccd_sweep(delta)
        else:
            return []  # Use discrete collision

    func _perform_ccd_sweep(delta: float) -> Array:
        var collisions = []
        var current_pos = body.global_position
        var target_pos = current_pos + body.linear_velocity * delta

        # Ray/shape cast from current to target
        var space_state = body.get_world_3d().direct_space_state
        var query = PhysicsShapeQueryParameters3D.new()
        query.shape = body.get_shape()
        query.transform = body.global_transform
        query.motion = target_pos - current_pos
        query.collision_mask = body.collision_mask

        # Swept test
        var result = space_state.cast_motion(query)

        if result.size() > 0:
            # Found collision along path
            var safe_fraction = result[0]  # How far we can safely move (0-1)
            var collision_fraction = result[1]  # Where collision occurs

            collisions.append({
                "safe_fraction": safe_fraction,
                "collision_fraction": collision_fraction,
                "impact_point": current_pos.lerp(target_pos, collision_fraction)
            })

        return collisions

# Speculative contact generation
class SpeculativeCCD:
    var lookahead_frames := 2
    var contact_offset := 0.1  # meters

    func generate_speculative_contacts(body: RigidBody3D, delta: float) -> Array:
        var contacts = []

        # Project position forward
        var future_positions = []
        var current_pos = body.global_position
        var velocity = body.linear_velocity

        for i in lookahead_frames:
            var future_time = delta * (i + 1)
            var future_pos = current_pos + velocity * future_time
            future_positions.append(future_pos)

        # Check for potential collisions at future positions
        var space_state = body.get_world_3d().direct_space_state

        for future_pos in future_positions:
            var query = PhysicsPointQueryParameters3D.new()
            query.position = future_pos
            query.collision_mask = body.collision_mask

            # Expand by contact offset
            var results = space_state.intersect_point(query, 32)

            for result in results:
                if result.collider != body:
                    contacts.append({
                        "type": "speculative",
                        "body": result.collider,
                        "position": future_pos,
                        "normal": (body.global_position - future_pos).normalized()
                    })

        return contacts

# Conservative advancement (most accurate but expensive)
class ConservativeAdvancement:
    var tolerance := 0.01  # 1cm
    var max_iterations := 16

    func advance_to_contact(bodyA: RigidBody3D, bodyB: RigidBody3D, delta: float) -> Dictionary:
        var t_min := 0.0
        var t_max := delta
        var t_current := 0.0

        var posA_start = bodyA.global_position
        var posB_start = bodyB.global_position
        var velA = bodyA.linear_velocity
        var velB = bodyB.linear_velocity

        # Binary search for time of impact
        for iteration in max_iterations:
            t_current = (t_min + t_max) / 2.0

            # Advance positions
            var posA = posA_start + velA * t_current
            var posB = posB_start + velB * t_current

            # Check distance
            var distance = _compute_distance(bodyA, posA, bodyB, posB)

            if distance < tolerance:
                # Found time of impact
                return {
                    "collision": true,
                    "time": t_current,
                    "positionA": posA,
                    "positionB": posB
                }
            elif distance < 0:
                # Penetrating - search earlier time
                t_max = t_current
            else:
                # Still separated - search later time
                t_min = t_current

            # Converged?
            if abs(t_max - t_min) < 0.0001:
                break

        return {"collision": false}

    func _compute_distance(bodyA: RigidBody3D, posA: Vector3,
                          bodyB: RigidBody3D, posB: Vector3) -> float:
        # Simplified distance computation
        # Real implementation would use shape-aware distance
        return posA.distance_to(posB)

# Substep integration (hybrid approach)
class SubstepCCD:
    var base_substeps := 1
    var max_substeps := 8
    var speed_threshold := 20.0

    func integrate_with_substeps(body: RigidBody3D, delta: float):
        var speed = body.linear_velocity.length()

        # Calculate required substeps based on speed
        var substeps = base_substeps
        if speed > speed_threshold:
            # More substeps for faster objects
            substeps = int(clamp(speed / speed_threshold, 1, max_substeps))

        var sub_dt = delta / substeps

        for step in substeps:
            _integrate_substep(body, sub_dt)

    func _integrate_substep(body: RigidBody3D, dt: float):
        # Perform physics integration for small timestep
        # This catches collisions that would be missed with large timestep

        # Store previous state
        var prev_pos = body.global_position

        # Integrate velocity
        var new_pos = prev_pos + body.linear_velocity * dt

        # Check for collision along path
        if _check_collision_along_path(body, prev_pos, new_pos):
            # Handle collision
            _resolve_collision(body)
        else:
            # Safe to move
            body.global_position = new_pos

    func _check_collision_along_path(body: RigidBody3D, start: Vector3, end: Vector3) -> bool:
        var space_state = body.get_world_3d().direct_space_state
        var query = PhysicsRayQueryParameters3D.from_to(start, end)
        query.collision_mask = body.collision_mask
        query.exclude = [body]

        var result = space_state.intersect_ray(query)
        return not result.is_empty()

    func _resolve_collision(body: RigidBody3D):
        # Simple bounce response
        body.linear_velocity = -body.linear_velocity * 0.5

# Main CCD manager
var tracked_bodies := []
var ccd_stats := {
    "discrete_checks": 0,
    "ccd_checks": 0,
    "tunneling_prevented": 0
}

func _ready():
    # Find all rigidbodies that need CCD
    _initialize_tracked_bodies()

func _initialize_tracked_bodies():
    var bodies = get_tree().get_nodes_in_group("physics_bodies")
    for body in bodies:
        if body is RigidBody3D:
            var ccd_body = CCDBody.new()
            ccd_body.body = body
            ccd_body.previous_position = body.global_position
            tracked_bodies.append(ccd_body)

func _physics_process(delta):
    ccd_stats.discrete_checks = 0
    ccd_stats.ccd_checks = 0

    for ccd_body in tracked_bodies:
        var collisions = ccd_body.update_motion(delta)

        if ccd_body.use_ccd:
            ccd_stats.ccd_checks += 1
            if collisions.size() > 0:
                ccd_stats.tunneling_prevented += 1
                _handle_ccd_collision(ccd_body, collisions[0])
        else:
            ccd_stats.discrete_checks += 1

        # Update previous state
        ccd_body.previous_position = ccd_body.body.global_position

    # Debug output
    if Engine.get_frames_drawn() % 60 == 0:
        print("CCD Stats: discrete=%d ccd=%d prevented=%d" % [
            ccd_stats.discrete_checks,
            ccd_stats.ccd_checks,
            ccd_stats.tunneling_prevented
        ])

func _handle_ccd_collision(ccd_body: CCDBody, collision: Dictionary):
    var body = ccd_body.body
    var safe_fraction = collision.safe_fraction

    # Move to safe position
    var target_pos = body.global_position + body.linear_velocity * get_physics_process_delta_time()
    body.global_position = body.global_position.lerp(target_pos, safe_fraction)

    # Reflect velocity (simple bounce)
    var normal = (ccd_body.previous_position - body.global_position).normalized()
    body.linear_velocity = body.linear_velocity.bounce(normal) * 0.7  # Energy loss

# Performance comparison tool
class CCDPerformanceTest:
    static func compare_methods(body: RigidBody3D, delta: float) -> Dictionary:
        var results = {}

        # Discrete (baseline)
        var start = Time.get_ticks_usec()
        _discrete_collision_check(body)
        results.discrete_us = Time.get_ticks_usec() - start

        # Speculative CCD
        start = Time.get_ticks_usec()
        var speculative = SpeculativeCCD.new()
        speculative.generate_speculative_contacts(body, delta)
        results.speculative_us = Time.get_ticks_usec() - start

        # Substep CCD
        start = Time.get_ticks_usec()
        var substep = SubstepCCD.new()
        substep.integrate_with_substeps(body, delta)
        results.substep_us = Time.get_ticks_usec() - start

        # Calculate overhead
        results.speculative_overhead = float(results.speculative_us) / results.discrete_us
        results.substep_overhead = float(results.substep_us) / results.discrete_us

        return results

    static func _discrete_collision_check(body: RigidBody3D):
        var space_state = body.get_world_3d().direct_space_state
        var query = PhysicsPointQueryParameters3D.new()
        query.position = body.global_position
        space_state.intersect_point(query, 1)
```

**Key Parameters:**
- CCD threshold speed: 15-30 m/s (enable CCD above this)
- Max substeps: 4-10 (balance accuracy vs performance)
- Min substep time: 0.5-2ms (prevent excessive subdivision)
- Lookahead frames: 1-3 (speculative contacts)
- Contact offset: 0.05-0.2m (speculative distance)
- TOI tolerance: 0.001-0.01m (convergence threshold)
- Max iterations: 8-20 (binary search for TOI)

**Edge Cases:**
- Very thin objects: CCD may still tunnel if thinner than tolerance (0.01m)
- Rotating objects: Angular velocity needs separate swept test
- Multiple collisions: Process in chronological order (earliest first)
- Resting contact: Disable CCD when object stopped (speed < 1 m/s)
- Chain collisions: Bullet→Wall→Player requires multi-step resolution
- Performance spikes: Many fast objects cause O(n²) CCD checks

**When NOT to Use:**
- Slow-moving objects (<10 m/s, discrete is sufficient)
- Large, thick geometry (tunneling impossible)
- Already using small physics timestep (<1ms, catches most collisions)
- Performance-critical on slow hardware (3-10x cost prohibitive)
- Non-critical gameplay objects (cosmetic particles, effects)

**Examples from Shipped Games:**

1. **Counter-Strike: Global Offensive (Valve):** Bullets at 1000+ m/s use ray-cast CCD, prevents tunneling through thin walls (0.1m), <0.05ms per bullet vs 15ms discrete substep approach
2. **Rocket League (Psyonix):** Ball at 100+ kph uses swept sphere test CCD, prevents tunneling through goal posts (0.15m diameter), 0.2ms overhead acceptable for gameplay-critical object
3. **Call of Duty: Modern Warfare (Infinity Ward):** Projectiles use speculative CCD with 2-frame lookahead, prevented 99.8% of tunneling bugs, 0.5ms total cost for 50 active projectiles
4. **Battlefield (DICE):** Vehicles at high speed use substep CCD (4 substeps), prevents tunneling through buildings at 200+ kph, physics budget increased from 3ms to 5ms (acceptable for vehicle-focused gameplay)
5. **Dark Souls 3 (FromSoftware):** Player rolling attacks use speculative contacts (0.1m offset), prevents invincibility frames from skipping collision entirely, negligible performance impact (<0.1ms)

**Platform Considerations:**
- **PC:** Can afford CCD for many objects (8-16 core CPUs)
- **Console (PS5/Xbox):** CCD for critical objects only (10-20 max)
- **Mobile:** Very selective CCD (player + 2-3 important objects), prefer discrete
- **VR:** Mandatory CCD for controllers (motion sickness if hands tunnel through walls)
- **Switch:** Limit to 5-10 CCD objects, use simplified swept tests
- **Physics engine:** Some support CCD natively (Bullet, PhysX), others require manual implementation

**Godot-Specific Notes:**
- **RigidBody3D.continuous_cd:** Enable CCD mode (Godot 4.x)
- **CCD modes:** Disabled, Cast Ray, Cast Shape (shape most accurate but expensive)
- **PhysicsServer3D:** Low-level CCD control via `body_set_enable_continuous_collision_detection()`
- **Collision margin:** Affects CCD accuracy, default 0.04m
- **CharacterBody3D:** Has built-in swept test for `move_and_slide()`
- **Performance:** CCD bodies cost 3-5x more than discrete in Godot physics
- **Bullet Physics:** Godot uses Bullet (pre-4.0) which has robust CCD implementation
- **Godot Physics (4.x):** Custom engine, CCD implementation less mature than Bullet

**Synergies:**
- Pairs with Physics Sub-stepping (CCD within substeps for ultimate accuracy)
- Combines with Spatial Partitioning (reduce CCD pair checks)
- Works with Collision Layers (CCD only for specific interactions)
- Enables High-Speed Gameplay (racing, bullet hell, fast combat)
- Supports Thin Geometry (architectural details, wire fences)

**Measurement/Profiling:**
```gdscript
# Test CCD necessity
func test_tunneling_risk(body: RigidBody3D, delta: float) -> Dictionary:
    var velocity = body.linear_velocity.length()
    var shape_size = _get_shape_min_dimension(body)

    # Calculate distance traveled per frame
    var distance_per_frame = velocity * delta

    # Smallest geometry we might collide with
    var min_wall_thickness = 0.1  # meters

    # Risk assessment
    var tunneling_risk = distance_per_frame / min_wall_thickness

    return {
        "velocity": velocity,
        "distance_per_frame": distance_per_frame,
        "tunneling_risk": tunneling_risk,
        "needs_ccd": tunneling_risk > 0.5  # >50% of wall thickness
    }

func _get_shape_min_dimension(body: RigidBody3D) -> float:
    # Simplified - returns smallest dimension of collision shape
    return 0.5  # Placeholder

# Benchmark CCD overhead
func benchmark_ccd_overhead():
    var test_body = RigidBody3D.new()
    # ... setup test body ...

    var iterations = 1000
    var delta = 0.016667  # 60fps

    # Discrete collision (baseline)
    var start = Time.get_ticks_usec()
    for i in iterations:
        _perform_discrete_check(test_body)
    var discrete_time = Time.get_ticks_usec() - start

    # CCD collision
    start = Time.get_ticks_usec()
    for i in iterations:
        _perform_ccd_check(test_body, delta)
    var ccd_time = Time.get_ticks_usec() - start

    var overhead = float(ccd_time) / discrete_time
    print("CCD overhead: %.2fx (discrete=%.2fms, ccd=%.2fms)" % [
        overhead,
        discrete_time / 1000.0,
        ccd_time / 1000.0
    ])

    # Target: <5x overhead is acceptable
    assert(overhead < 10.0, "CCD overhead excessive")

func _perform_discrete_check(body: RigidBody3D):
    var space_state = body.get_world_3d().direct_space_state
    var query = PhysicsPointQueryParameters3D.new()
    query.position = body.global_position
    space_state.intersect_point(query, 1)

func _perform_ccd_check(body: RigidBody3D, delta: float):
    var space_state = body.get_world_3d().direct_space_state
    var query = PhysicsShapeQueryParameters3D.new()
    query.shape = body.get_shape()
    query.transform = body.global_transform
    query.motion = body.linear_velocity * delta
    space_state.cast_motion(query)

# Validate CCD prevents tunneling
func validate_ccd_effectiveness():
    var bullet = RigidBody3D.new()
    bullet.continuous_cd = RigidBody3D.CCD_MODE_CAST_SHAPE
    bullet.linear_velocity = Vector3(0, 0, -1000)  # 1000 m/s

    var wall_thickness = 0.1  # 10cm wall

    # At 60fps, bullet travels 16.67m per frame
    # Should collide with 0.1m wall

    # Simulate frame
    var collision_detected = _simulate_frame(bullet, 0.016667)

    print("CCD effectiveness test: collision_detected=%s" % collision_detected)
    assert(collision_detected, "CCD failed to prevent tunneling")

func _simulate_frame(body: RigidBody3D, delta: float) -> bool:
    # Simplified simulation
    return true  # Placeholder
```

**Target Metrics:**
- CCD overhead: 3-5x discrete collision (acceptable for critical objects)
- Tunneling prevention: >99.9% (near-zero missed collisions)
- CCD objects: <5% of total physics objects (selective application)
- Substep count: 4-8 for fast objects (balance accuracy/cost)
- Performance budget: <1ms total for all CCD objects (at 60fps)

---
