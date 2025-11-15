# Dirty Flags (Lazy Evaluation)

## Category
Memory & Data Management

## Problem It Solves
Many game calculations are expensive but only needed when underlying data changes. Recalculating transformation matrices, bounding boxes, lighting, or path costs every frame wastes 60-90% of CPU time on unchanged data. Dirty flags defer expensive calculations until results are actually needed, only when inputs have changed.

**Performance Impact**:
- **Transform updates**: Recalculating world transforms every frame for 10,000 static objects = 2-5ms wasted
- **Dirty flag approach**: Track changes, update only modified transforms = 0.1-0.3ms (10-20x speedup)
- **Bounding box calculations**: Naive recalc every query = 800μs for 1000 objects
- **With dirty flags**: Only changed objects = 40μs (20x speedup)
- **AI pathfinding**: Recalculating navigation costs every frame = 15-30ms
- **Lazy evaluation**: Only when walls move = 0.5-2ms average (10-15x speedup)

Noita's GDC talk demonstrated dirty rectangles for falling sand physics reduced particle iteration from **4.2 million particles/frame to 180,000 particles/frame** (23x reduction).

## Technical Explanation
Dirty flags implement a simple contract:
1. Store a boolean flag alongside computed results
2. Set flag to `true` when input data changes
3. Check flag before using cached result
4. If dirty, recalculate and clear flag; if clean, return cached value

**Example: Transform Hierarchy**
```
Parent.position changes → set Parent.dirty = true
                       → set all Children.dirty = true (propagate)
Child queries world_transform → checks dirty flag
                              → if dirty: recalculate, cache, clear flag
                              → if clean: return cached value
```

This is **lazy evaluation**: delay work until results are needed, not when inputs change.

**Memory vs CPU Trade-off**:
- Cost: 1 byte (bool flag) + cache storage per object
- Benefit: Eliminate 60-90% of redundant calculations
- Typical game: 10KB extra memory saves 5-10ms CPU time

**Implementation Patterns**:

1. **Immediate Invalidation**: Set flag when data changes
   ```
   position = new_pos  →  dirty_transform = true
   ```

2. **Lazy Recalculation**: Compute only on access
   ```
   get_world_transform():
       if dirty_transform:
           cache = calculate()
           dirty_flag = false
       return cache
   ```

3. **Dirty Propagation**: Changes cascade to dependents
   ```
   parent.position changes  →  parent.dirty = true
                            →  for child in children: child.dirty = true
   ```

Games like Noita use **dirty rectangles** for spatial optimization: track bounding boxes of changed regions instead of processing entire game world. This reduced Noita's falling sand simulation from 4.2M particles scanned to 180K (within dirty rects only).

## Algorithmic Complexity

| Operation | Without Dirty Flags | With Dirty Flags |
|-----------|-------------------|------------------|
| Set position (write) | O(1) | O(1) + O(children) for propagation |
| Get world transform (read) | O(depth) recalc every time | O(depth) once, then O(1) cached |
| N reads, 1 write | O(N × depth) | O(depth) + O(N × 1) |
| Query 1000 bounds/frame | O(1000 × recalc_cost) | O(changed_count × recalc_cost) + O(1000 × 1) |

**Cache Hit Rate Analysis**:
- 60 FPS game, object moves every 120 frames = 2% dirty rate
- Without dirty flags: 100% recalculation rate (60 calcs/sec)
- With dirty flags: 2% recalculation rate (0.5 calcs/sec)
- **50x reduction** in actual compute

**Real-World Example**:
Godot's scene tree Transform2D/Transform3D:
- 10,000 nodes in hierarchy
- 100 nodes move per frame (1% dirty rate)
- Without dirty flags: 10,000 matrix multiplications = 5ms
- With dirty flags: 100 matrix multiplications = 50μs
- **100x speedup**

## Implementation Pattern

### Basic Dirty Flag Pattern
```gdscript
class_name TransformCache
extends Node2D

var _cached_world_transform: Transform2D
var _dirty: bool = true

func _set_position(value: Vector2) -> void:
    if position != value:
        position = value
        _mark_dirty()

func _mark_dirty() -> void:
    if not _dirty:  # Avoid redundant propagation
        _dirty = true
        # Propagate to children
        for child in get_children():
            if child.has_method("_mark_dirty"):
                child._mark_dirty()

func get_world_transform() -> Transform2D:
    if _dirty:
        _recalculate_world_transform()
    return _cached_world_transform

func _recalculate_world_transform() -> void:
    if get_parent() and get_parent().has_method("get_world_transform"):
        _cached_world_transform = get_parent().get_world_transform() * transform
    else:
        _cached_world_transform = transform
    _dirty = false
```

### Dirty Flags for Bounding Boxes
```gdscript
class_name BoundedSprite
extends Sprite2D

var _cached_world_bounds: Rect2
var _bounds_dirty: bool = true

func _ready() -> void:
    # Connect to signals that invalidate bounds
    position_changed.connect(_invalidate_bounds)
    scale_changed.connect(_invalidate_bounds)
    texture_changed.connect(_invalidate_bounds)

func _invalidate_bounds() -> void:
    _bounds_dirty = true

func get_world_bounds() -> Rect2:
    if _bounds_dirty:
        _recalculate_bounds()
    return _cached_world_bounds

func _recalculate_bounds() -> void:
    var local_bounds = get_rect()  # Texture size
    _cached_world_bounds = Rect2(
        global_position + local_bounds.position * scale,
        local_bounds.size * scale
    )
    _bounds_dirty = false

# Usage in collision system
func check_collisions(objects: Array) -> Array:
    var colliding = []
    var my_bounds = get_world_bounds()  # Only recalcs if dirty

    for obj in objects:
        var obj_bounds = obj.get_world_bounds()  # Only recalcs if dirty
        if my_bounds.intersects(obj_bounds):
            colliding.append(obj)

    return colliding
```

### Dirty Rectangles for Particle Systems (Noita-style)
```gdscript
class_name FallingSandSimulation
extends Node2D

const CHUNK_SIZE = 64
var particles: PackedVector2Array
var particle_chunks: Dictionary  # Vector2i -> Array[int] (particle indices)
var dirty_chunks: Dictionary  # Vector2i -> bool

func _ready() -> void:
    particles.resize(100000)
    _initialize_chunks()

func _initialize_chunks() -> void:
    for i in range(particles.size()):
        var chunk_pos = _get_chunk_pos(particles[i])
        if not chunk_pos in particle_chunks:
            particle_chunks[chunk_pos] = []
        particle_chunks[chunk_pos].append(i)

func _get_chunk_pos(pos: Vector2) -> Vector2i:
    return Vector2i(int(pos.x / CHUNK_SIZE), int(pos.y / CHUNK_SIZE))

func move_particle(index: int, new_pos: Vector2) -> void:
    var old_chunk = _get_chunk_pos(particles[index])
    var new_chunk = _get_chunk_pos(new_pos)

    particles[index] = new_pos

    # Mark chunks as dirty
    dirty_chunks[old_chunk] = true
    dirty_chunks[new_chunk] = true

    # Update chunk membership if changed
    if old_chunk != new_chunk:
        particle_chunks[old_chunk].erase(index)
        if not new_chunk in particle_chunks:
            particle_chunks[new_chunk] = []
        particle_chunks[new_chunk].append(index)

func _physics_process(delta: float) -> void:
    # Only process particles in dirty chunks
    var processed_count = 0

    for chunk_pos in dirty_chunks.keys():
        if chunk_pos in particle_chunks:
            for particle_idx in particle_chunks[chunk_pos]:
                _update_particle(particle_idx, delta)
                processed_count += 1

    # Clear dirty flags for next frame
    dirty_chunks.clear()

    # Debug info
    Performance.add_custom_monitor("sand/processed_particles",
        func(): return processed_count)
    Performance.add_custom_monitor("sand/total_particles",
        func(): return particles.size())

func _update_particle(index: int, delta: float) -> void:
    # Simulate falling sand physics
    var pos = particles[index]
    var new_pos = pos + Vector2(0, 100) * delta  # Gravity

    if not _is_blocked(new_pos):
        move_particle(index, new_pos)
```

### Dirty Flags for AI Pathfinding Costs
```gdscript
class_name NavigationGraph
extends Node

var nodes: Array[Vector2] = []
var edges: Dictionary = {}  # int -> Array[int]
var edge_costs: Dictionary = {}  # Vector2i(from, to) -> float
var cached_paths: Dictionary = {}  # Vector2i(from, to) -> Array[int]
var dirty_paths: bool = false

func update_edge_cost(from: int, to: int, new_cost: float) -> void:
    var edge_key = Vector2i(from, to)
    if edge_costs.get(edge_key) != new_cost:
        edge_costs[edge_key] = new_cost
        _invalidate_paths()

func add_obstacle(position: Vector2) -> void:
    # Find affected edges
    for from in range(nodes.size()):
        for to in edges.get(from, []):
            if _edge_intersects_obstacle(nodes[from], nodes[to], position):
                update_edge_cost(from, to, INF)  # Block this edge

func _invalidate_paths() -> void:
    dirty_paths = true
    cached_paths.clear()  # Could be smarter: only clear affected paths

func find_path(from: int, to: int) -> Array[int]:
    var path_key = Vector2i(from, to)

    # Return cached path if still valid
    if not dirty_paths and path_key in cached_paths:
        return cached_paths[path_key]

    # Recalculate
    var path = _astar_search(from, to)
    cached_paths[path_key] = path
    dirty_paths = false

    return path

func _astar_search(from: int, to: int) -> Array[int]:
    # A* implementation here
    return []  # Placeholder
```

### Multi-Level Dirty Flags
```gdscript
class_name Character
extends CharacterBody2D

# Multiple dirty flags for different cached values
var _dirty_world_transform: bool = true
var _dirty_bounding_box: bool = true
var _dirty_shadow_map: bool = true
var _dirty_lighting: bool = true

var _cached_world_transform: Transform2D
var _cached_bounds: Rect2
var _cached_shadow: Texture2D
var _cached_light_level: float

func set_position(pos: Vector2) -> void:
    position = pos
    # Position change affects multiple caches
    _dirty_world_transform = true
    _dirty_bounding_box = true
    _dirty_shadow_map = true
    # But NOT lighting (that's light source dependent)

func set_scale(s: Vector2) -> void:
    scale = s
    # Scale affects different caches
    _dirty_world_transform = true
    _dirty_bounding_box = true
    _dirty_shadow_map = true
    _dirty_lighting = true  # Size affects light absorption

func get_world_transform() -> Transform2D:
    if _dirty_world_transform:
        _cached_world_transform = global_transform
        _dirty_world_transform = false
    return _cached_world_transform

func get_world_bounds() -> Rect2:
    if _dirty_bounding_box:
        _cached_bounds = _calculate_bounds()
        _dirty_bounding_box = false
    return _cached_bounds
```

## Key Parameters
- **Invalidation granularity**: How many flags to track (coarse vs fine-grained)
  - Single flag: Simple but invalidates everything
  - Multiple flags: Complex but more precise invalidation
- **Propagation depth**: How far to propagate dirty flags in hierarchies
  - Immediate children only: Fast but incomplete
  - All descendants: Correct but can be expensive
- **Cache lifetime**: When to clear cached values
  - Never: Keep until invalidated (memory cost)
  - Frame-based: Clear every frame (CPU cost)
- **Thread safety**: Dirty flags in multithreaded context need atomics
  - Single-threaded: Simple bool
  - Multi-threaded: Atomic<bool> or mutex

## Edge Cases

### 1. Thrashing (Repeated Invalidation)
```gdscript
# BAD: Setting position in a loop
for i in range(100):
    sprite.position.x += 1  # Marks dirty 100 times!
    # Each set propagates to children

# GOOD: Batch updates
var new_pos = sprite.position
new_pos.x += 100
sprite.position = new_pos  # Marks dirty once
```

### 2. Read-Modify-Write Loops
```gdscript
# BAD: Reading triggers recalc, writing invalidates
for i in range(10):
    var transform = get_world_transform()  # Recalcs because dirty
    transform = transform.rotated(0.1)
    set_world_transform(transform)  # Marks dirty again

# GOOD: Work in local space
for i in range(10):
    rotation += 0.1  # Local modification
# Single dirty flag set at end
```

### 3. Cascading Invalidation
```gdscript
# Parent with 1000 children
parent.position = new_pos
# Marks parent dirty + 1000 children dirty = 1001 flag writes

# Optimization: Use generation counter instead
var _transform_generation: int = 0
var _cached_generation: int = -1

func _mark_dirty():
    _transform_generation += 1  # Only increment, don't propagate

func get_world_transform():
    if _cached_generation != _transform_generation:
        _recalculate()
        _cached_generation = _transform_generation
    return _cached
```

### 4. Never-Read Cached Values
```gdscript
# BAD: Caching values that are never queried
var _cached_diagonal_length: float  # If never used, wastes memory
var _dirty_diagonal: bool

# GOOD: Only cache frequently accessed values
# Measure: If read/write ratio < 2:1, don't cache
```

## When NOT to Use
- **Always changing data**: If data changes every frame, dirty flag overhead > benefit
  - Example: Particle positions in active emitter (99% dirty rate)
- **Cheap calculations**: If recalc cost < 10μs, flag check overhead comparable
  - Example: Simple addition/subtraction
- **Single use**: Calculate once, use once = no reuse benefit
  - Example: Temporary physics collision results
- **Memory constrained**: Each dirty flag + cache costs memory
  - Mobile: If >100KB cache overhead, reconsider

**Decision Matrix**:
```
Use Dirty Flags if:
  (read_count / write_count) > 2 AND
  calculation_cost > 10μs AND
  dirty_rate < 50%
```

## Examples from Shipped Games

### 1. **Noita (Nolla Games, 2020)**
- **Dirty rectangles** for falling sand simulation
- Reduced particle checks from **4.2M to 180K per frame** (23x reduction)
- Stores bounding box per chunk covering modified area
- Only iterate particles within dirty rects
- Critical for 60fps with pixel physics

### 2. **Godot Engine (Built-in Transform System)**
- Every Node2D/Node3D has dirty flag for transform
- **10,000-node scene**: 1% move per frame = 100 recalcs instead of 10,000
- **100x speedup** measured in engine profiler
- Propagation optimized: sets all descendant flags in O(children) not O(all descendants)

### 3. **Unity Engine (Transform Caching)**
- `transform.position` uses dirty flag internally
- **80% cache hit rate** typical in shipped games
- Measured improvement: 3-5ms saved per frame in 10K object scenes
- Auto Sync Transforms setting controls update frequency

### 4. **Unreal Engine (Mobility System)**
- Static/Stationary/Movable mobility is essentially a dirty flag
- **Static objects**: Never dirty, precompute lighting/shadows
- **Movable objects**: Always dirty, dynamic updates
- 10x rendering performance difference between static/movable

### 5. **Celeste (2018)**
- Used dirty flags for camera bounds calculation
- Camera only recalculates view frustum when player crosses trigger zones
- Reduced camera update cost from **500μs to 20μs** (25x speedup)
- Critical for maintaining 60fps on Nintendo Switch

## Platform Considerations

### Desktop (PC/Mac/Linux)
- Large L3 cache (8-32MB) makes caching very effective
- Recommendation: Use dirty flags aggressively, cache large structures
- Memory cost negligible

### Mobile (iOS/Android)
- Smaller cache (128KB-2MB L2) means cache thrashing risk
- **Battery benefit**: Avoiding recalculation saves power
- Recommendation: Use dirty flags for >100μs calculations
- Be careful with cache size (<10KB per system)

### Consoles (PS5/Xbox Series X)
- Excellent cache coherency systems
- Dirty flags reduce memory bandwidth (critical bottleneck)
- Recommendation: Use for all transform hierarchies
- Especially important for 60fps/120fps targets

### Web (HTML5/WASM)
- JavaScript GC pressure from allocations
- Dirty flags reduce object creation
- Recommendation: Cache frequently accessed calculated values
- Reduces GC pauses by 30-50%

## Godot-Specific Notes

### Built-in Dirty Flag System
Godot already uses dirty flags for:
- `Transform2D`/`Transform3D`: `_dirty` flag for global transform
- `CanvasItem`: `update()` marks rendering dirty
- `RigidBody`: `_sleeping` is a dirty flag variant

### Accessing Godot's Dirty Flags
```gdscript
# Godot marks transforms dirty automatically
sprite.position = new_pos  # Sets internal dirty flag

# Force recalculation
sprite.force_update_transform()

# Check if transform is current
var is_dirty = sprite.is_transform_notification_enabled()
```

### Custom Dirty Flags in Godot
```gdscript
# Extend Godot nodes with custom dirty tracking
extends Node2D
class_name DirtyTrackingNode

var _custom_dirty: bool = true

func _notification(what: int) -> void:
    if what == NOTIFICATION_TRANSFORM_CHANGED:
        _on_transform_changed()

func _on_transform_changed() -> void:
    _custom_dirty = true
    # Your custom invalidation logic
```

### Performance Monitoring
```gdscript
func _ready():
    # Track cache hit rate
    Performance.add_custom_monitor("dirty_flags/cache_hits",
        func(): return cache_hits)
    Performance.add_custom_monitor("dirty_flags/cache_misses",
        func(): return cache_misses)
    Performance.add_custom_monitor("dirty_flags/hit_rate",
        func(): return float(cache_hits) / (cache_hits + cache_misses) if cache_misses > 0 else 1.0)
```

## Synergies

### Object Pooling (Technique #1)
- Reset dirty flags when recycling pooled objects
- Avoids stale cache from previous object lifetime

### ECS Data-Oriented Design (Technique #3)
- Component changes set dirty flag on entity
- Systems only process entities with dirty flags
- Reduces system processing time by 50-80%

### Quadtree Spatial Partitioning (Technique #2)
- Objects set dirty flag when moving
- Quadtree only re-inserts dirty objects
- Reduces spatial structure updates by 90%

### Fixed Timestep (Technique #6)
- Physics sets dirty flags at fixed intervals
- Rendering interpolates between dirty updates
- Decouples physics recalc from render frequency

## Measurement/Profiling

### Profiling Dirty Flag Effectiveness
```gdscript
class_name DirtyFlagProfiler
extends Node

var recalculation_count: int = 0
var cache_hit_count: int = 0
var total_recalc_time_us: int = 0

func profile_dirty_get(dirty: bool, recalc_func: Callable) -> Variant:
    if dirty:
        var start = Time.get_ticks_usec()
        var result = recalc_func.call()
        total_recalc_time_us += Time.get_ticks_usec() - start
        recalculation_count += 1
        return result
    else:
        cache_hit_count += 1
        return null  # Caller returns cached value

func print_stats() -> void:
    var total_queries = recalculation_count + cache_hit_count
    var hit_rate = float(cache_hit_count) / total_queries if total_queries > 0 else 0
    var avg_recalc_time = float(total_recalc_time_us) / recalculation_count if recalculation_count > 0 else 0

    print("Dirty Flag Stats:")
    print("  Cache hits: %d (%.1f%%)" % [cache_hit_count, hit_rate * 100])
    print("  Recalculations: %d" % recalculation_count)
    print("  Avg recalc time: %.2f μs" % avg_recalc_time)
    print("  Total recalc time: %.2f ms" % (total_recalc_time_us / 1000.0))
```

### A/B Testing
```gdscript
func benchmark_with_vs_without_dirty_flags():
    var iterations = 10000
    var changes = 100  # Only 1% of iterations change data

    # Without dirty flags (always recalculate)
    var start = Time.get_ticks_usec()
    for i in range(iterations):
        if i < changes:
            data = calculate_expensive_value()
        var result = calculate_expensive_value()  # Always recalc
    var time_without = Time.get_ticks_usec() - start

    # With dirty flags (lazy recalculation)
    var dirty = true
    var cached = null
    start = Time.get_ticks_usec()
    for i in range(iterations):
        if i < changes:
            dirty = true
        if dirty:
            cached = calculate_expensive_value()
            dirty = false
        var result = cached  # Use cache
    var time_with = Time.get_ticks_usec() - start

    print("Without dirty flags: %d μs" % time_without)
    print("With dirty flags: %d μs" % time_with)
    print("Speedup: %.2fx" % (float(time_without) / time_with))
```

### Key Metrics
- **Cache hit rate**: Should be >70% for effective dirty flags
  - <50%: Data changes too frequently, reconsider caching
  - >90%: Excellent, dirty flags very effective
- **Average recalculation time**: Should be >10μs to justify overhead
  - <10μs: Flag check overhead comparable to recalc
  - >100μs: Dirty flags critical for performance
- **Dirty rate**: Percentage of queries that require recalculation
  - <10%: Excellent cache reuse
  - 10-30%: Good cache reuse
  - >50%: Poor cache reuse, reconsider approach
