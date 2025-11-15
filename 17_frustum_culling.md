### 17. Frustum Culling

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Rendering objects outside camera view wastes GPU cycles. In a scene with 10,000 objects, camera sees ~20% (2000), but GPU processes all 10K = 8K wasted draw calls = 40ms overhead on mid-range GPU, breaking 16.67ms frame budget by 2.4x.

**Technical Explanation:**
View frustum is the 3D pyramidal volume visible to the camera, defined by 6 planes (near, far, left, right, top, bottom). Before rendering, test each object's bounding volume (AABB, sphere, OBB) against these planes. If object is completely outside any plane, cull it (skip rendering). Test uses dot product between plane normal and object center/extents. Modern engines use SIMD to test 4-16 objects simultaneously. Reduces draw calls, vertex processing, and overdraw.

**Algorithmic Complexity:**
- No culling: O(n) objects rendered regardless of visibility
- Frustum culling: O(n) tests, but O(k) rendered where k << n (typically 15-30%)
- Plane-sphere test: 6 dot products + 6 comparisons = ~24 ops
- AABB test: 6 plane tests × 8 corners = 48 ops (optimized to ~12 with early exit)
- Best case: 70-85% objects culled in typical 3D environment

**Implementation Pattern:**
```gdscript
# Godot Frustum Culling Implementation
class_name FrustumCuller
extends Node

var frustum_planes: Array[Plane] = []
var camera: Camera3D
var culled_count: int = 0
var visible_count: int = 0

func _ready():
    camera = get_viewport().get_camera_3d()

func _process(_delta):
    if not camera:
        return
    _update_frustum_planes()

func _update_frustum_planes():
    # Extract frustum planes from camera projection matrix
    var projection = camera.get_camera_projection()
    var transform = camera.global_transform

    # Calculate frustum planes in world space
    frustum_planes.clear()

    # Near/Far planes
    var near_dist = camera.near
    var far_dist = camera.far
    var forward = -transform.basis.z
    frustum_planes.append(Plane(forward, transform.origin + forward * near_dist))
    frustum_planes.append(Plane(-forward, transform.origin + forward * far_dist))

    # Calculate FOV-based planes
    var fov_rad = deg_to_rad(camera.fov)
    var aspect = get_viewport().get_visible_rect().size.x / get_viewport().get_visible_rect().size.y
    var half_v_side = far_dist * tan(fov_rad * 0.5)
    var half_h_side = half_v_side * aspect

    var right = transform.basis.x
    var up = transform.basis.y
    var front_mult_far = forward * far_dist

    # Right plane
    var right_point = transform.origin + front_mult_far - right * half_h_side
    var right_normal = (right_point - transform.origin).cross(up).normalized()
    frustum_planes.append(Plane(right_normal, transform.origin))

    # Left plane
    var left_point = transform.origin + front_mult_far + right * half_h_side
    var left_normal = up.cross(left_point - transform.origin).normalized()
    frustum_planes.append(Plane(left_normal, transform.origin))

    # Top plane
    var top_point = transform.origin + front_mult_far - up * half_v_side
    var top_normal = right.cross(top_point - transform.origin).normalized()
    frustum_planes.append(Plane(top_normal, transform.origin))

    # Bottom plane
    var bottom_point = transform.origin + front_mult_far + up * half_v_side
    var bottom_normal = (bottom_point - transform.origin).cross(right).normalized()
    frustum_planes.append(Plane(bottom_normal, transform.origin))

func is_sphere_in_frustum(center: Vector3, radius: float) -> bool:
    # Fast sphere-frustum test
    for plane in frustum_planes:
        var distance = plane.distance_to(center)
        if distance < -radius:
            return false  # Completely outside this plane
    return true

func is_aabb_in_frustum(aabb: AABB) -> bool:
    # Test AABB against frustum planes
    var center = aabb.position + aabb.size * 0.5
    var extents = aabb.size * 0.5

    for plane in frustum_planes:
        # Get the positive vertex (furthest point in plane normal direction)
        var p = center
        p.x += extents.x if plane.normal.x >= 0 else -extents.x
        p.y += extents.y if plane.normal.y >= 0 else -extents.y
        p.z += extents.z if plane.normal.z >= 0 else -extents.z

        # If positive vertex is outside, AABB is fully outside
        if plane.distance_to(p) < 0:
            return false

    return true

func cull_objects(objects: Array) -> Array:
    var visible_objects = []
    culled_count = 0
    visible_count = 0

    for obj in objects:
        if not obj is VisualInstance3D:
            continue

        var aabb = obj.get_aabb()
        var global_aabb = AABB(
            obj.global_transform.origin + aabb.position,
            aabb.size
        )

        if is_aabb_in_frustum(global_aabb):
            visible_objects.append(obj)
            visible_count += 1
        else:
            culled_count += 1

    return visible_objects

# Advanced: Hierarchical culling with spatial partitioning
class_name HierarchicalFrustumCuller
extends FrustumCuller

var spatial_tree: Array = []  # Octree or BVH nodes

func cull_hierarchy(node, parent_visible: bool = true):
    if not parent_visible:
        return []

    # Test node bounds first
    if not is_aabb_in_frustum(node.bounds):
        # Entire subtree culled
        culled_count += node.object_count
        return []

    var visible = []

    # If leaf node, test individual objects
    if node.is_leaf:
        for obj in node.objects:
            if is_aabb_in_frustum(obj.get_global_aabb()):
                visible.append(obj)
                visible_count += 1
            else:
                culled_count += 1
    else:
        # Recurse to children
        for child in node.children:
            visible.append_array(cull_hierarchy(child, true))

    return visible

# Optimized SIMD-style batch culling
func cull_objects_batch(objects: Array, batch_size: int = 4) -> Array:
    var visible = []

    for i in range(0, objects.size(), batch_size):
        var batch_end = min(i + batch_size, objects.size())

        for j in range(i, batch_end):
            var obj = objects[j]
            if not obj is VisualInstance3D:
                continue

            # Early exit optimization: test sphere first (cheaper)
            var aabb = obj.get_aabb()
            var center = obj.global_transform.origin + aabb.position + aabb.size * 0.5
            var radius = aabb.size.length() * 0.5

            if is_sphere_in_frustum(center, radius):
                # Sphere visible, do precise AABB test
                if is_aabb_in_frustum(AABB(obj.global_transform.origin + aabb.position, aabb.size)):
                    visible.append(obj)
                    visible_count += 1
                else:
                    culled_count += 1
            else:
                culled_count += 1

    return visible
```

**Key Parameters:**
- Test frequency: Every frame (camera moves constantly)
- Bounding volume type: Sphere (fast) or AABB (accurate) or OBB (best but slow)
- Early exit: Test cheapest planes first (near/far)
- Padding: Add 5-10% margin to frustum to prevent pop-in
- Batch size: 4-8 objects per SIMD batch
- Hierarchical depth: 4-6 levels for octree integration

**Edge Cases:**
- Large objects: May be partially visible, use conservative AABB test
- Camera inside object: Reverse plane test logic
- Orthographic cameras: Different frustum shape, adjust plane calculation
- Split-screen: Multiple frustums, test against all
- Portal rendering: Custom frustum per portal
- Shadow frustums: Separate culling for shadow-casting lights
- Skinned meshes: Use pre-skinned AABB or conservative bounds

**When NOT to Use:**
- 2D games with screen-space culling (simpler rect test)
- Indoor scenes with occlusion culling (more effective)
- Very few objects (<100)
- Minimal overdraw scenarios
- Fixed camera perspective with known visibility

**Examples from Shipped Games:**

1. **Uncharted 4:** Combined frustum + occlusion culling, reduced rendered objects from 50K → 8K (84% cull rate), saved 25ms per frame on PS4. Hierarchical culling with octree, 0.3ms CPU overhead.

2. **Horizon Zero Dawn:** Aggressive frustum culling for vegetation, 500K grass instances → 80K visible (84% cull), enabled dense forests at 30fps. GPU-driven culling in compute shaders for massive scenes.

3. **Assassin's Creed Unity:** Paris crowd system, 10K NPCs → 2K rendered (80% cull), combined with LOD. Frustum test on CPU (0.5ms), prevented GPU bottleneck. Essential for crowd density.

4. **DOOM (2016):** Hierarchical frustum culling, 20K entities → 3K visible average (85% cull). Combined with portal system for interior spaces. Maintained 60fps on console.

5. **Fortnite:** Dynamic frustum culling for building pieces, 50K potential objects → 5-10K rendered (80-90% cull). CPU cost: 0.4ms for 50K tests using SIMD. Critical for building destruction performance.

**Platform Considerations:**
- **PC:** Use SIMD instructions (SSE/AVX) for 4-8x faster plane tests
- **Console:** Fixed hardware, optimize for specific SIMD capabilities
- **Mobile:** Essential - limited fill rate, aggressive culling saves battery
- **VR:** Cull per eye, different frustums, or use conservative unified frustum
- **CPU vs GPU:** CPU culling standard, GPU culling for massive scenes (>100K objects)
- **Memory:** Negligible overhead, 6 planes × 16 bytes = 96 bytes

**Godot-Specific Notes:**
- Godot automatically performs frustum culling for all `VisualInstance3D` nodes
- Access via `VisibleOnScreenNotifier3D` for gameplay logic
- Custom culling: Disable auto-culling, implement in `_process()`
- `Camera3D.is_position_behind()` for quick visibility test
- For particle systems: `GPUParticles3D` has built-in frustum culling
- Override `GeometryInstance3D.extra_cull_margin` for aggressive objects
- Godot 4.x: Improved culling with clustering, handles 100K+ objects efficiently

**Synergies:**
- Essential foundation for Occlusion Culling (pre-cull before occlusion tests)
- Pairs with LOD Systems (only update LODs for visible objects)
- Combines with Spatial Partitioning (cull entire octree nodes)
- Enables GPU Instancing (instance only visible objects)
- Works with Portal Rendering (modified frustum per portal)

**Measurement/Profiling:**
- Godot Profiler: Monitor `Rendering -> Objects culled`
- Track cull ratio: Should be 70-85% in outdoor scenes
- Measure CPU cost: <0.5ms for 10K objects
- Visual debug: Render frustum planes, color-code culled objects
- GPU savings: Measure vertex shader time (should decrease with culling)
- Profile false positives: Objects culled but should be visible (<1%)
- Target: 80%+ cull rate, <0.1ms per 1000 objects on CPU

**Additional Resources:**
- Real-Time Rendering 4th Edition: Chapter 19.1 "Culling"
- GPU Gems 1: Chapter 29 "Efficient Occlusion Culling"
- GDC 2015: "Rendering the Hellscape of DOOM"
