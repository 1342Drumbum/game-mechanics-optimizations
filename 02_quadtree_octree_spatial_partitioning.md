### 2. Quadtree/Octree Spatial Partitioning

**Category:** Performance - Data Structures

**Problem It Solves:** O(n²) collision detection. With 1000 objects, naive checking requires 499,500 comparisons. At 1000 cycles each = 500M cycles = 198ms on 2.5GHz CPU, exceeding 16.67ms budget by 12x.

**Technical Explanation:**
Recursive spatial subdivision. Each node contains 4 children (quadtree 2D) or 8 (octree 3D). Objects stored in leaf nodes. Query checks only relevant subtrees. Average O(n log n) insertion, O(log n) query. Exploits spatial coherence - nearby objects likely to interact.

**Algorithmic Complexity:**
- Naive: O(n²) collision checks
- Quadtree: O(n log n) build, O(log n + k) query where k = results
- Best case: 99.8% reduction in checks

**Implementation Pattern:**
```gdscript
class_name Quadtree

const MAX_OBJECTS = 10
const MAX_DEPTH = 6
const MIN_SIZE = 10.0

var bounds: Rect2
var objects: Array = []
var children: Array = []  # 4 children for quadrants
var depth: int = 0

func insert(obj, obj_bounds: Rect2) -> bool:
    if not bounds.intersects(obj_bounds):
        return false

    if objects.size() < MAX_OBJECTS or depth >= MAX_DEPTH:
        objects.append([obj, obj_bounds])
        return true

    if children.is_empty():
        _subdivide()

    for child in children:
        if child.insert(obj, obj_bounds):
            return true
    return false

func query(area: Rect2, results: Array = []) -> Array:
    if not bounds.intersects(area):
        return results

    for item in objects:
        if item[1].intersects(area):
            results.append(item[0])

    for child in children:
        child.query(area, results)

    return results

func _subdivide():
    var half_size = bounds.size / 2
    children.append(Quadtree.new(
        Rect2(bounds.position, half_size), depth + 1))
    # ... create other 3 quadrants
```

**Key Parameters:**
- Max objects per node: 4-16 (sweet spot: 8-10)
- Max depth: 6-10 levels
- Min node size: 10-50 units (depends on object size)
- Rebuild frequency: Every frame for dynamic, once for static

**Edge Cases:**
- Objects spanning multiple nodes: Store in parent
- Moving objects: Remove from old node, insert in new
- Very large objects: Store at root level
- Unbalanced trees: Limit depth to prevent deep recursion

**When NOT to Use:**
- <50 objects total (overhead exceeds benefit)
- Uniform spatial distribution (all objects always interact)
- 1D gameplay (use sorted array instead)

**Examples from Shipped Games:**

1. **Quake III Arena:** BSP tree (specialized octree) for indoor levels, 5000-10000 brushes, 30-50ms → <1ms culling
2. **Research Study:** 1000 entities: 1M checks → 1530 checks (99.8% reduction), 198ms → 38ms frame time
3. **Terraria-like Games:** Quadtree for chunk management, 80,640 → 336 tiles rendered (99% reduction)
4. **Unity Physics:** Uses dynamic AABB tree (similar concept), handles 10K+ objects
5. **Factorio:** Quadtree for entity management across massive maps, critical for performance

**Platform Considerations:**
- **2D Games:** Quadtree (4 children per node)
- **3D Games:** Octree (8 children) or BVH (better for dynamic)
- **Mobile:** Keep max depth lower (4-6) to reduce memory
- **Indoor:** BSP trees better (pre-computed from level geometry)
- **Memory:** Each node ~64-128 bytes, limit depth accordingly

**Godot-Specific Notes:**
- Godot's built-in physics uses BVH (Bounding Volume Hierarchy)
- For custom logic, implement quadtree in GDScript or C++
- Use `Area2D/Area3D` with collision layers for simple spatial queries
- Consider `VisibleOnScreenNotifier2D` for camera-based culling
- Godot 4.x: BVH performs better than manual trees for most cases

**Synergies:**
- Essential for Broadphase Collision optimization
- Pairs with Frustum Culling (query visible area)
- Combines with Object Pooling (spatial organization of pools)
- Enables Occlusion Culling (spatial visibility tests)

**Measurement/Profiling:**
- Godot Profiler: Check `_physics_process` time
- Count collision pairs: Should be <1% of n²
- Monitor rebuild frequency (should be <1ms)
- Profile query time: Target <0.5ms for typical queries
- Debug draw: Visualize tree structure to verify balance
