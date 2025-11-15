# Physics Sleeping Bodies

## Category
Memory & Data Management

## Problem It Solves
Physics engines waste massive CPU time simulating static objects that haven't moved in seconds. A typical game scene has **70-90% of physics objects stationary** at any moment, yet naive simulation updates all of them every frame:
- **1,000 rigid bodies**, all simulating: **8-15ms** per frame
- **1,000 rigid bodies**, 900 sleeping: **1-2ms** per frame (**5-10x speedup**)
- **10,000 rigid bodies** without sleeping: **80-150ms** (unplayable)
- **10,000 rigid bodies** with sleeping: **8-15ms** (playable)

**Performance Impact Breakdown**:
- Active rigid body: ~10-20μs per frame (integration + collision checks)
- Sleeping rigid body: ~0.1-0.5μs per frame (just awake checks)
- **50-100x cheaper** to keep object asleep than active

Unity documentation states physics sleeping is "especially important" for mobile games, reducing processing needed for contact calculations as sleeping objects don't require position/velocity updates.

## Technical Explanation

### Sleep/Wake State Machine
```
[AWAKE] → velocity < threshold for N frames → [SLEEPING]
[SLEEPING] → collision/force applied → [AWAKE]
```

**Awake State**:
- Full physics integration (velocity, position, rotation)
- Collision detection and response
- Constraint solving
- Force accumulation
- **Cost**: 10-20μs per object per frame

**Sleeping State**:
- Skip physics integration
- Skip collision broadphase (static objects don't move)
- Skip constraint solving (if all connected objects sleeping)
- Only check for wake conditions
- **Cost**: 0.1-0.5μs per object per frame

### Sleep Criteria
An object can sleep when:
1. **Linear velocity** < sleep_threshold (e.g., 0.5 m/s)
2. **Angular velocity** < sleep_threshold (e.g., 0.5 rad/s)
3. Conditions met for **N consecutive frames** (e.g., 60 frames = 1 second)
4. All **connected objects** (via constraints) also eligible to sleep

### Wake Conditions
An object must wake when:
1. External **force/impulse** applied
2. **Collision** with awake object
3. Attached via **constraint** to awake object (rope pulls sleeping crate)
4. Position/velocity **explicitly set** by code
5. Constraint **breaks** (requires re-simulation)

### Implementation Pseudocode
```
func physics_tick(delta):
    for obj in rigid_bodies:
        if obj.is_sleeping:
            # Check wake conditions (cheap)
            if has_force_applied(obj) or collision_with_awake(obj):
                wake_up(obj)
            continue  # Skip expensive physics

        # Full physics simulation
        integrate_velocity(obj, delta)
        integrate_position(obj, delta)
        solve_constraints(obj)

        # Check sleep conditions
        if obj.velocity.length() < SLEEP_THRESHOLD:
            obj.sleep_timer += delta
            if obj.sleep_timer > SLEEP_TIME:
                put_to_sleep(obj)
        else:
            obj.sleep_timer = 0
```

### Optimization: Island-Based Sleeping
Physics engines group connected objects into "islands":
```
Island 1: [Box A] ←→ [Box B] ←→ [Box C]  (all connected via contacts/constraints)
Island 2: [Sphere D] ←→ [Sphere E]
Island 3: [Static Ground] (infinite mass, always sleeping)
```

**Benefit**: Entire island sleeps/wakes together
- Island 1 at rest → sleep all 3 boxes (one decision, not three)
- External force hits Box A → wake entire island

This reduces wake/sleep checks from O(N) to O(islands), typically **10-50x fewer** checks.

## Algorithmic Complexity

| Operation | Without Sleeping | With Sleeping |
|-----------|------------------|---------------|
| Physics integration (N objects, S sleeping) | O(N) | O(N - S) |
| Collision detection | O(N²) or O(N log N) | O((N-S)²) or O((N-S) log(N-S)) |
| Constraint solving | O(N × iterations) | O((N-S) × iterations) |
| Sleep/wake checks | N/A | O(islands) ≈ O(N/10) |

**Concrete Example** (1,000 objects, 900 sleeping):
- Full physics: 1,000 objects × 15μs = 15,000μs = **15ms**
- With sleeping: 100 objects × 15μs + 900 × 0.3μs = 1,500μs + 270μs = **1.77ms**
- **Speedup: 8.5x**

**Memory Complexity**:
- Additional per-object: 8-16 bytes (sleep timer, flags)
- Island graph: O(N + constraints)
- Total overhead: <1% of physics memory

## Implementation Pattern

### Basic Sleeping Rigid Body
```gdscript
class_name SleepingRigidBody
extends RigidBody2D

const SLEEP_LINEAR_THRESHOLD = 10.0   # pixels/second
const SLEEP_ANGULAR_THRESHOLD = 0.1   # radians/second
const SLEEP_TIME = 1.0                # seconds

var _sleep_timer: float = 0.0
var _is_sleeping: bool = false

func _ready() -> void:
    # Enable Godot's built-in sleeping (recommended)
    sleeping = false  # Start awake
    can_sleep = true  # Allow automatic sleeping

func _integrate_forces(state: PhysicsDirectBodyState2D) -> void:
    if _is_sleeping:
        # Skip custom physics when sleeping
        # Godot still needs to integrate, but we can skip custom logic
        return

    # Custom physics logic here
    apply_custom_forces(state)

    # Manual sleep check (if not using Godot's automatic sleeping)
    _check_sleep_condition(state)

func _check_sleep_condition(state: PhysicsDirectBodyState2D) -> void:
    var lin_vel = state.linear_velocity.length()
    var ang_vel = abs(state.angular_velocity)

    if lin_vel < SLEEP_LINEAR_THRESHOLD and ang_vel < SLEEP_ANGULAR_THRESHOLD:
        _sleep_timer += state.step
        if _sleep_timer >= SLEEP_TIME:
            put_to_sleep()
    else:
        _sleep_timer = 0.0

func put_to_sleep() -> void:
    _is_sleeping = true
    sleeping = true  # Tell Godot physics system
    modulate = Color(0.7, 0.7, 0.7)  # Visual feedback (grayed out)

func wake_up() -> void:
    _is_sleeping = false
    sleeping = false
    _sleep_timer = 0.0
    modulate = Color(1, 1, 1)  # Restore color

func apply_impulse_at_point(impulse: Vector2, point: Vector2) -> void:
    wake_up()  # External force wakes object
    super.apply_impulse(impulse, point)
```

### Sleep Manager System
```gdscript
class_name PhysicsSleepManager
extends Node

var _tracked_bodies: Array[RigidBody2D] = []
var _sleeping_count: int = 0
var _awake_count: int = 0

func register_body(body: RigidBody2D) -> void:
    _tracked_bodies.append(body)
    body.sleeping_state_changed.connect(_on_sleep_state_changed.bind(body))

func _on_sleep_state_changed(body: RigidBody2D) -> void:
    if body.sleeping:
        _sleeping_count += 1
        _awake_count -= 1
    else:
        _sleeping_count -= 1
        _awake_count += 1

func _physics_process(_delta: float) -> void:
    # Only awake bodies need processing
    for body in _tracked_bodies:
        if not body.sleeping:
            # Process awake body
            pass

    # Performance monitoring
    Performance.add_custom_monitor("physics/sleeping_bodies",
        func(): return _sleeping_count)
    Performance.add_custom_monitor("physics/awake_bodies",
        func(): return _awake_count)
    Performance.add_custom_monitor("physics/sleep_ratio",
        func(): return float(_sleeping_count) / max(1, _tracked_bodies.size()))
```

### Island-Based Sleep System
```gdscript
class_name PhysicsIslandManager
extends Node

class PhysicsIsland:
    var bodies: Array[RigidBody2D] = []
    var is_sleeping: bool = false
    var sleep_timer: float = 0.0

var _islands: Array[PhysicsIsland] = []

const ISLAND_SLEEP_THRESHOLD = 1.0  # All bodies calm for 1 second
const VELOCITY_THRESHOLD = 10.0

func _physics_process(delta: float) -> void:
    _rebuild_islands()  # Group connected bodies

    for island in _islands:
        if island.is_sleeping:
            # Check wake conditions
            if _island_has_external_force(island):
                _wake_island(island)
        else:
            # Check sleep conditions
            if _island_is_calm(island):
                island.sleep_timer += delta
                if island.sleep_timer >= ISLAND_SLEEP_THRESHOLD:
                    _sleep_island(island)
            else:
                island.sleep_timer = 0.0

func _island_is_calm(island: PhysicsIsland) -> bool:
    for body in island.bodies:
        if body.linear_velocity.length() > VELOCITY_THRESHOLD:
            return false
        if abs(body.angular_velocity) > 0.1:
            return false
    return true

func _sleep_island(island: PhysicsIsland) -> void:
    island.is_sleeping = true
    for body in island.bodies:
        body.sleeping = true

func _wake_island(island: PhysicsIsland) -> void:
    island.is_sleeping = false
    island.sleep_timer = 0.0
    for body in island.bodies:
        body.sleeping = false

func _rebuild_islands() -> void:
    # Use union-find or graph search to group connected bodies
    # Bodies connected via contacts or constraints are in same island
    _islands.clear()

    var visited = {}
    for body in get_tree().get_nodes_in_group("physics_bodies"):
        if not body in visited:
            var island = PhysicsIsland.new()
            _flood_fill_island(body, island, visited)
            _islands.append(island)

func _flood_fill_island(body: RigidBody2D, island: PhysicsIsland, visited: Dictionary) -> void:
    visited[body] = true
    island.bodies.append(body)

    # Find all bodies in contact with this one
    for contact in body.get_colliding_bodies():
        if contact is RigidBody2D and not contact in visited:
            _flood_fill_island(contact, island, visited)
```

### Selective Wake System
```gdscript
class_name SelectiveWakeSystem
extends Node2D

var _bodies: Array[RigidBody2D] = []

func _ready() -> void:
    # Put all bodies to sleep initially
    for body in _bodies:
        body.sleeping = true

func wake_in_radius(center: Vector2, radius: float) -> void:
    # Only wake bodies near an event (explosion, player, etc.)
    for body in _bodies:
        if body.sleeping:
            var distance = body.global_position.distance_to(center)
            if distance < radius:
                body.sleeping = false

func wake_in_camera_view(camera: Camera2D) -> void:
    # Only simulate visible objects
    var viewport_rect = camera.get_viewport_rect()
    var camera_center = camera.global_position

    for body in _bodies:
        if body.sleeping:
            var in_view = viewport_rect.has_point(
                body.global_position - camera_center
            )
            if in_view:
                body.sleeping = false
            # Otherwise keep sleeping (invisible objects)
```

## Key Parameters
- **Linear velocity threshold**: 0.1-1.0 m/s (game scale dependent)
  - Too low: Objects never sleep (jitter from numerical precision)
  - Too high: Objects sleep while visibly moving
- **Angular velocity threshold**: 0.05-0.2 rad/s
  - Similar trade-offs to linear threshold
- **Sleep time**: 0.5-2.0 seconds
  - Too short: Objects thrash between sleep/wake
  - Too long: Objects stay awake unnecessarily
- **Wake margin**: Wake sleeping objects slightly before collision
  - Prevents "sleeping through" fast collisions

## Edge Cases

### 1. Oscillating Sleep/Wake (Thrashing)
```gdscript
# Object at edge of sleep threshold vibrates
# velocity = 0.49, 0.51, 0.49, 0.51... (thrashing)

# Solution: Hysteresis
const SLEEP_THRESHOLD = 0.5
const WAKE_THRESHOLD = 0.7  # Higher than sleep

func check_sleep():
    if velocity < SLEEP_THRESHOLD:
        go_to_sleep()

func check_wake():
    if velocity > WAKE_THRESHOLD:  # Not SLEEP_THRESHOLD
        wake_up()
```

### 2. Constraint Sleeping Bugs
```gdscript
# Box A (sleeping) connected to Box B (awake) via rope
# Box B moves → should wake Box A, but might not if constraint logic broken

# Solution: Propagate wake through constraints
func wake_up():
    _is_sleeping = false
    for constraint in attached_constraints:
        if constraint.other_body.is_sleeping:
            constraint.other_body.wake_up()  # Recursive wake
```

### 3. Tunneling Through Sleeping Objects
```gdscript
# Fast bullet hits sleeping wall
# Wall doesn't wake up in time → bullet tunnels through

# Solution: Wake objects in projected path
func projectile_check_path(start: Vector2, end: Vector2):
    var space_state = get_world_2d().direct_space_state
    var query = PhysicsRayQueryParameters2D.create(start, end)
    var result = space_state.intersect_ray(query)

    if result and result.collider is RigidBody2D:
        result.collider.sleeping = false  # Pre-wake
```

### 4. Stack Collapse
```gdscript
# Tower of boxes: bottom box sleeps → top boxes sleep
# Remove bottom box → top boxes still asleep, should fall!

# Solution: Wake when constraint breaks
func _on_constraint_broken(other_body):
    wake_up()
    other_body.wake_up()
```

## When NOT to Use

- **Constant motion games**: Platformers where everything always moves
  - Sleep checks waste CPU if nothing ever sleeps
- **Deterministic replay**: Sleeping can introduce non-determinism
  - Floating-point precision differences affect sleep timing
  - Solution: Force all objects awake in replay mode
- **Very few objects**: <10 physics objects
  - Sleep overhead > benefit
- **Real-time multiplayer**: Sleep state must sync across clients
  - Adds network complexity

## Examples from Shipped Games

### 1. **Angry Birds (Rovio)**
- Aggressive sleeping after physics settle
- 100+ blocks in level, only 5-10 awake at a time
- Sleep after **0.5 seconds** of calm
- Wake radius around bird impact point
- Critical for 60 FPS on 2010-era mobile devices

### 2. **World of Goo**
- Physics-based tower building
- 200+ goo balls, 90% sleeping at any moment
- Island-based sleeping (connected goo balls sleep together)
- Measured: **10x physics performance** improvement with sleeping
- Enabled complex towers on Nintendo Wii (limited CPU)

### 3. **Half-Life 2 (Valve)**
- Source engine physics simulation
- Sleep threshold: **36 units/sec** linear, **6 deg/sec** angular
- Sleep after **1 second** below threshold
- Barnacle constraint breaks → wake attached objects
- Hundreds of physics props in levels, mostly sleeping

### 4. **Teardown (Tuxedo Labs)**
- Voxel destruction physics
- **10,000+ rigid bodies** from destruction
- Aggressive sleeping (0.3s threshold)
- Wake radius: 10 meters around player/explosions
- **95% of debris sleeping** at any time
- Without sleeping: unplayable on any hardware

### 5. **BeamNG.drive**
- Vehicle soft-body physics (3,000+ nodes per car)
- Parked cars fully sleep (0 CPU)
- Collision wakes car → full physics for ~2 seconds → sleep again
- Measured: **100x CPU reduction** for parked vehicles
- Enables 10+ cars in scene simultaneously

## Platform Considerations

### Desktop (PC/Mac/Linux)
- Can afford tighter sleep thresholds (more accuracy)
- Recommendation: 0.5m/s threshold, 1.0s sleep time
- More objects can be simulated awake

### Mobile (iOS/Android)
- **Critical for battery life**: Sleeping reduces CPU load
- Recommendation: 1.0m/s threshold, 0.5s sleep time (aggressive)
- Sleep 90%+ of objects
- Unity docs emphasize "especially important for mobile"

### Consoles (PS5/Xbox Series X)
- Good CPU headroom but physics still expensive
- Recommendation: 0.7m/s threshold, 0.8s sleep time
- Enable complex physics scenes (destruction, ragdolls)

### Web (HTML5/WASM)
- Physics in WASM still expensive
- Recommendation: Aggressive sleeping (like mobile)
- Keep 80-90% of objects sleeping

## Godot-Specific Notes

### Built-in Sleeping System
Godot's physics engine automatically manages sleeping:

```gdscript
# RigidBody2D/3D properties
body.can_sleep = true   # Enable automatic sleeping (default: true)
body.sleeping = false   # Current sleep state (read/write)

# Sleep signals
body.sleeping_state_changed.connect(_on_sleep_changed)

# Project Settings
# Physics > 2D/3D > Sleep Threshold Linear: 2.0 (pixels/sec or meters/sec)
# Physics > 2D/3D > Sleep Threshold Angular: 8.0 (degrees/sec)
# Physics > 2D/3D > Time Before Sleep: 0.5 (seconds)
```

### Manual Sleep Control
```gdscript
# Force object awake
body.sleeping = false

# Force object asleep (check conditions first!)
if body.linear_velocity.length() < 1.0:
    body.sleeping = true

# Query sleep state
if body.sleeping:
    print("Object is sleeping")
```

### Performance Monitoring
```gdscript
func _physics_process(_delta):
    var sleeping_count = 0
    var total_count = 0

    for body in get_tree().get_nodes_in_group("physics_objects"):
        if body is RigidBody2D or body is RigidBody3D:
            total_count += 1
            if body.sleeping:
                sleeping_count += 1

    Performance.add_custom_monitor("physics/sleep_percentage",
        func(): return float(sleeping_count) / max(1, total_count) * 100)
```

## Synergies

### Object Pooling (Technique #1)
- Return sleeping objects to pool
- Pooled objects start asleep
- Wake on spawn from pool

### Spatial Hashing (Technique #7)
- Don't insert sleeping objects into spatial hash
- Wake objects when query overlaps their AABB
- Reduces spatial hash size by 70-90%

### Fixed Timestep (Technique #6)
- Consistent sleep timing with fixed timestep
- Sleep threshold in units/tick (deterministic)
- Replay systems work correctly

### Dirty Flags (Technique #4)
- Sleeping objects not marked dirty
- Transform caches stay valid
- Skip rendering updates for sleeping objects

## Measurement/Profiling

### Profiling Sleep Effectiveness
```gdscript
class_name SleepProfiler
extends Node

var _total_objects: int = 0
var _sleeping_objects: int = 0
var _sleep_transitions: int = 0
var _wake_transitions: int = 0

func _physics_process(_delta):
    _total_objects = 0
    _sleeping_objects = 0

    for body in get_tree().get_nodes_in_group("physics"):
        if body is RigidBody2D or body is RigidBody3D:
            _total_objects += 1
            if body.sleeping:
                _sleeping_objects += 1

func on_body_slept():
    _sleep_transitions += 1

func on_body_woke():
    _wake_transitions += 1

func print_stats():
    var sleep_ratio = float(_sleeping_objects) / _total_objects
    print("Sleep Stats:")
    print("  Sleeping: %d / %d (%.1f%%)" % [_sleeping_objects, _total_objects, sleep_ratio * 100])
    print("  Sleep transitions: %d" % _sleep_transitions)
    print("  Wake transitions: %d" % _wake_transitions)
    print("  Thrashing: %s" % ("YES" if _wake_transitions > _total_objects * 5 else "NO"))
```

### Key Metrics
- **Sleep ratio**: Target 70-90% sleeping
  - <50%: Not enough objects sleeping, check thresholds
  - >95%: Maybe too aggressive, objects sleeping while moving
- **Transition rate**: Sleep/wake transitions per second
  - <10% of object count: Good (stable sleep)
  - >50% of object count: Thrashing (adjust thresholds)
- **Physics time reduction**: Measure with/without sleeping
  - Target: 5-10x speedup with 80% sleeping
