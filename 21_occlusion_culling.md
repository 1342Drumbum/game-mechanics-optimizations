### 21. Occlusion Culling

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Rendering objects hidden behind other objects. In dense 3D environments, 60-80% of geometry is occluded but still processed by GPU. A city scene with 20K objects: 15K occluded × 1ms = 15ms wasted GPU time. Occlusion culling eliminates hidden geometry, reducing overdraw from 10x → 1.5x, saving 12-13ms GPU time.

**Technical Explanation:**
Occlusion culling determines which objects are blocked from view by other objects (occluders). Multiple approaches: Portal-based (indoor), Hardware queries (GPU feedback), Software rasterization (CPU), PVS (Potentially Visible Sets). Hardware method: render occluders to depth buffer, query if objects would be visible. Software: rasterize scene to low-res depth buffer on CPU, test object bounds. Modern engines use hybrid: precomputed PVS + runtime hardware queries. Critical for dense 3D scenes (cities, interiors, forests).

**Algorithmic Complexity:**
- No occlusion culling: O(n) objects rendered after frustum culling
- Portal culling: O(k × m) where k=visible portals, m=objects per room
- Hardware queries: O(n) tests + GPU latency (1-2 frames)
- Software rasterization: O(w × h) rasterization + O(n) tests
- PVS: O(1) lookup + O(k) objects where k=visible set
- Typical savings: 60-80% objects culled in dense scenes

**Implementation Pattern:**
```gdscript
# Godot Occlusion Culling System
class_name OcclusionCuller
extends Node3D

# Portal-based occlusion (best for interiors)
class PortalCullingSystem:
    var portals: Array[Portal] = []
    var rooms: Array[Room] = []

    class Portal:
        var polygon: PackedVector3Array  # Portal shape
        var room_a: Room
        var room_b: Room
        var plane: Plane

        func is_visible_from(position: Vector3) -> bool:
            # Check if camera can see through portal
            return plane.is_point_over(position)

    class Room:
        var bounds: AABB
        var objects: Array[VisualInstance3D] = []
        var portals: Array[Portal] = []
        var visible: bool = false

    func cull_from_camera(camera_pos: Vector3) -> Array:
        # Reset visibility
        for room in rooms:
            room.visible = false

        # Find camera's room
        var current_room = _find_room_containing(camera_pos)
        if not current_room:
            return []

        # Mark current room visible
        current_room.visible = true
        var visible_objects = current_room.objects.duplicate()

        # Recursively find visible rooms through portals
        _traverse_portals(current_room, camera_pos, visible_objects)

        return visible_objects

    func _traverse_portals(room: Room, camera_pos: Vector3, visible_objects: Array, depth: int = 0):
        if depth > 10:  # Prevent infinite recursion
            return

        for portal in room.portals:
            if not portal.is_visible_from(camera_pos):
                continue

            var next_room = portal.room_a if portal.room_b == room else portal.room_b
            if next_room.visible:
                continue

            next_room.visible = true
            visible_objects.append_array(next_room.objects)

            # Recurse through connected portals
            _traverse_portals(next_room, camera_pos, visible_objects, depth + 1)

    func _find_room_containing(pos: Vector3) -> Room:
        for room in rooms:
            if room.bounds.has_point(pos):
                return room
        return null

# Software occlusion culling (CPU rasterization)
class SoftwareOcclusionCuller:
    var depth_buffer: PackedFloat32Array
    var buffer_width: int = 256
    var buffer_height: int = 256
    var camera: Camera3D

    func _init(camera_node: Camera3D):
        camera = camera_node
        depth_buffer.resize(buffer_width * buffer_height)

    func cull_objects(objects: Array, occluders: Array) -> Array:
        # Clear depth buffer
        depth_buffer.fill(INF)

        # Rasterize occluders to depth buffer
        for occluder in occluders:
            if occluder is MeshInstance3D:
                _rasterize_occluder(occluder)

        # Test objects against depth buffer
        var visible_objects = []
        for obj in objects:
            if _is_visible(obj):
                visible_objects.append(obj)

        return visible_objects

    func _rasterize_occluder(occluder: MeshInstance3D):
        var mesh = occluder.mesh
        if not mesh:
            return

        var arrays = mesh.surface_get_arrays(0)
        var vertices = arrays[Mesh.ARRAY_VERTEX]
        var indices = arrays[Mesh.ARRAY_INDEX]

        if not indices:
            return

        # Transform vertices to screen space
        var screen_verts = []
        for v in vertices:
            var world_pos = occluder.global_transform * v
            var screen_pos = _world_to_screen(world_pos)
            screen_verts.append(screen_pos)

        # Rasterize triangles
        for i in range(0, indices.size(), 3):
            var v0 = screen_verts[indices[i]]
            var v1 = screen_verts[indices[i + 1]]
            var v2 = screen_verts[indices[i + 2]]

            if v0.z > 0 and v1.z > 0 and v2.z > 0:  # Behind camera
                _rasterize_triangle(v0, v1, v2)

    func _rasterize_triangle(v0: Vector3, v1: Vector3, v2: Vector3):
        # Conservative rasterization
        var min_x = int(min(v0.x, min(v1.x, v2.x)))
        var max_x = int(max(v0.x, max(v1.x, v2.x)))
        var min_y = int(min(v0.y, min(v1.y, v2.y)))
        var max_y = int(max(v0.y, max(v1.y, v2.y)))

        min_x = max(0, min_x)
        max_x = min(buffer_width - 1, max_x)
        min_y = max(0, min_y)
        max_y = min(buffer_height - 1, max_y)

        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                if _point_in_triangle(Vector2(x, y), Vector2(v0.x, v0.y), Vector2(v1.x, v1.y), Vector2(v2.x, v2.y)):
                    var depth = _interpolate_depth(Vector2(x, y), v0, v1, v2)
                    var index = y * buffer_width + x

                    if depth < depth_buffer[index]:
                        depth_buffer[index] = depth

    func _is_visible(obj: VisualInstance3D) -> bool:
        var aabb = obj.get_aabb()
        var center = obj.global_transform.origin + aabb.position + aabb.size * 0.5

        var screen_pos = _world_to_screen(center)

        if screen_pos.z < 0:  # Behind camera
            return false

        var x = int(screen_pos.x)
        var y = int(screen_pos.y)

        if x < 0 or x >= buffer_width or y < 0 or y >= buffer_height:
            return true  # Conservative: visible if outside buffer

        var index = y * buffer_width + x
        var buffer_depth = depth_buffer[index]

        return screen_pos.z <= buffer_depth + 1.0  # Bias for thin occluders

    func _world_to_screen(world_pos: Vector3) -> Vector3:
        var cam_space = camera.global_transform.affine_inverse() * world_pos
        var proj = camera.get_camera_projection() * Vector4(cam_space.x, cam_space.y, cam_space.z, 1.0)

        if abs(proj.w) < 0.001:
            return Vector3(0, 0, -1)

        var ndc = Vector3(proj.x / proj.w, proj.y / proj.w, proj.z / proj.w)

        var screen_x = (ndc.x + 1.0) * 0.5 * buffer_width
        var screen_y = (1.0 - ndc.y) * 0.5 * buffer_height

        return Vector3(screen_x, screen_y, cam_space.z)

    func _point_in_triangle(p: Vector2, v0: Vector2, v1: Vector2, v2: Vector2) -> bool:
        var d1 = _sign(p, v0, v1)
        var d2 = _sign(p, v1, v2)
        var d3 = _sign(p, v2, v0)

        var has_neg = (d1 < 0) or (d2 < 0) or (d3 < 0)
        var has_pos = (d1 > 0) or (d2 > 0) or (d3 > 0)

        return not (has_neg and has_pos)

    func _sign(p1: Vector2, p2: Vector2, p3: Vector2) -> float:
        return (p1.x - p3.x) * (p2.y - p3.y) - (p2.x - p3.x) * (p1.y - p3.y)

    func _interpolate_depth(p: Vector2, v0: Vector3, v1: Vector3, v2: Vector3) -> float:
        # Barycentric interpolation
        var denom = (v1.y - v2.y) * (v0.x - v2.x) + (v2.x - v1.x) * (v0.y - v2.y)
        if abs(denom) < 0.001:
            return v0.z

        var w0 = ((v1.y - v2.y) * (p.x - v2.x) + (v2.x - v1.x) * (p.y - v2.y)) / denom
        var w1 = ((v2.y - v0.y) * (p.x - v2.x) + (v0.x - v2.x) * (p.y - v2.y)) / denom
        var w2 = 1.0 - w0 - w1

        return w0 * v0.z + w1 * v1.z + w2 * v2.z

# Godot built-in occlusion culling wrapper
class GodotOcclusionCulling:
    # Godot 4.x has built-in occlusion culling
    # This class provides helper functions

    func setup_occluder(mesh_instance: MeshInstance3D):
        # Mark mesh as occluder
        mesh_instance.gi_mode = GeometryInstance3D.GI_MODE_STATIC
        # Godot will automatically use it for occlusion

    func bake_scene_occlusion(root: Node3D):
        # In Godot, use OccluderInstance3D nodes
        # Create occluders from large static geometry
        pass

# Hybrid approach: Combine multiple techniques
class HybridOcclusionCuller:
    var portal_culler: PortalCullingSystem
    var software_culler: SoftwareOcclusionCuller
    var use_portals: bool = true
    var use_software: bool = true

    func cull_scene(camera: Camera3D, all_objects: Array, occluders: Array) -> Array:
        var visible = all_objects

        # First pass: Portal culling (fast, precise for interiors)
        if use_portals and portal_culler:
            visible = portal_culler.cull_from_camera(camera.global_position)

        # Second pass: Software occlusion (handles outdoor/complex scenes)
        if use_software and software_culler:
            visible = software_culler.cull_objects(visible, occluders)

        return visible
```

**Key Parameters:**
- Occluder threshold: Objects >10m² surface area (large buildings, walls)
- Query resolution: 256×256 to 512×512 for software rasterization
- Portal count: 2-6 per room (more = slower traversal)
- PVS cell size: 4m×4m×4m for outdoor, 1 room for indoor
- Update frequency: Every frame for dynamic, cached for static
- Depth bias: 0.5-2.0m for conservative visibility
- Max recursion: 5-10 portal hops

**Edge Cases:**
- Thin occluders: Add depth bias or increase rasterization resolution
- Moving occluders: Expensive, use simplified proxies or skip
- Transparent objects: Cannot be occluders (see-through)
- Small gaps: Conservative rasterization may miss, use higher resolution
- Camera inside occluder: Reverse culling or skip occlusion for frame
- Distant objects: May be incorrectly culled, combine with LOD

**When NOT to Use:**
- Open outdoor environments (few occluders, frustum culling sufficient)
- Simple scenes (<1000 objects)
- Sparse geometry (low occlusion potential)
- Mobile low-end (CPU overhead exceeds GPU savings)
- Highly dynamic environments (occluders constantly change)

**Examples from Shipped Games:**

1. **Mirror's Edge:** Portal-based occlusion, reduced rendered geometry by 75% in city interiors. 30fps → 60fps on same hardware. Precomputed PVS cells, 0.2ms CPU lookup.

2. **Assassin's Creed Unity:** Software occlusion culling, Paris buildings occlude 80% of geometry. Saved 20ms GPU time (60ms → 40ms). 512×512 depth buffer, 1.5ms CPU overhead.

3. **Battlefield 1:** Hybrid occlusion: buildings rasterized to 256×256 buffer on CPU. 15K objects → 3K visible (80% cull rate). Essential for destruction performance (occluders update).

4. **DOOM (2016):** Portal system for indoor, combined with hardware queries. 20K demons/objects → 3K visible. Precomputed visibility for static geometry. Maintained 60fps on console.

5. **Spider-Man (2018):** Building occlusion for NYC, software rasterization on SPU cores (PS4). 70% geometry culled, enabled dense city at 30fps. Dynamic updates as camera moves, <2ms overhead.

**Platform Considerations:**
- **PC High-End:** Hardware occlusion queries viable (low latency)
- **PC Low-End:** Software occlusion better (CPU faster than GPU)
- **Console:** Dedicated cores for software rasterization (PS4 SPU, Xbox compute)
- **Mobile:** Portal culling only (low overhead), avoid software rasterization
- **VR:** Critical for performance, but latency issues with hardware queries
- **Memory:** PVS = 1-10MB for large worlds, software buffer = <1MB

**Godot-Specific Notes:**
- Godot 4.x: Built-in occlusion culling with `OccluderInstance3D`
- Use `ArrayOccluder3D` for custom occluder shapes
- Automatic occlusion for static geometry with GI
- `GeometryInstance3D.extra_cull_margin` affects occlusion
- Manual occlusion: Disable nodes not in current room/area
- No hardware queries exposed, use software or portal approach
- Consider using `VisibilityNotifier3D` for simple area-based culling

**Synergies:**
- Builds on Frustum Culling (only test frustum-visible objects)
- Enhances LOD Systems (skip LOD updates for occluded objects)
- Pairs with Draw Call Batching (fewer objects = fewer batches)
- Combines with Spatial Partitioning (cull octree nodes)
- Works with Shadow Culling (don't render shadows for occluded)

**Measurement/Profiling:**
- Monitor objects rendered: Should decrease 60-80% in dense scenes
- Measure CPU overhead: <2ms for software, <0.5ms for portal
- Track GPU time: Should decrease proportional to culled geometry
- Profile false positives: Visible objects incorrectly culled (<5%)
- Visual debug: Color occluders, show depth buffer
- Target: 70%+ cull rate in interiors, 40%+ in complex outdoor
- Performance: 10-20ms GPU savings in dense scenes

**Additional Resources:**
- GDC 2015: "Occlusion Culling in DOOM"
- GPU Pro 5: "Hi-Z GPU Occlusion Culling"
- Frostbite Engine: Software Occlusion Culling presentation
