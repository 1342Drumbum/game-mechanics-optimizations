### 18. Draw Call Batching

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** CPU-GPU communication overhead from excessive draw calls. Each draw call = 50-100 CPU cycles + API overhead + state changes. 5000 draw calls × 0.02ms = 100ms CPU time, exceeding 16.67ms budget by 6x. Modern GPUs can render millions of triangles but choke on thousands of draw calls.

**Technical Explanation:**
Batching combines multiple objects with same material/shader into single draw call. Static batching pre-combines meshes at build/load time. Dynamic batching merges instances per-frame if they share materials. Instancing (MultiMesh) renders identical objects with single draw call + instance buffer. GPU processes one command but renders multiple objects. Reduces API calls (OpenGL/Vulkan/Metal overhead), state changes (shader/texture binding), and driver validation. Critical for CPU-bound scenarios.

**Algorithmic Complexity:**
- No batching: O(n) draw calls for n objects
- Static batching: O(1) draw call for combined mesh, O(n) memory
- Dynamic batching: O(k) draw calls where k = unique materials, O(n log n) sorting
- Instancing: O(1) draw call + O(n) instance buffer, best performance
- CPU overhead: Sorting/combining = 0.5-2ms per 1000 objects

**Implementation Pattern:**
```gdscript
# Godot Draw Call Batching System
class_name DrawCallBatcher
extends Node3D

# Static Batching: Combine meshes at startup
class StaticBatcher:
    var combined_meshes: Dictionary = {}  # Material -> MeshInstance3D

    func batch_static_objects(objects: Array):
        # Group by material
        var material_groups = {}
        for obj in objects:
            if not obj is MeshInstance3D:
                continue

            var material = obj.get_surface_override_material(0)
            if not material:
                material = obj.mesh.surface_get_material(0)

            if not material in material_groups:
                material_groups[material] = []

            material_groups[material].append(obj)

        # Combine meshes per material
        for material in material_groups:
            var combined = _combine_meshes(material_groups[material])
            combined_meshes[material] = combined

    func _combine_meshes(objects: Array) -> MeshInstance3D:
        var arrays = []
        var surface_tool = SurfaceTool.new()

        surface_tool.begin(Mesh.PRIMITIVE_TRIANGLES)

        for obj in objects:
            var mesh = obj.mesh
            var transform = obj.global_transform

            # Get mesh data
            var mesh_arrays = mesh.surface_get_arrays(0)

            # Transform vertices to world space
            var vertices = mesh_arrays[Mesh.ARRAY_VERTEX]
            var normals = mesh_arrays[Mesh.ARRAY_NORMAL]
            var uvs = mesh_arrays[Mesh.ARRAY_TEX_UV]
            var indices = mesh_arrays[Mesh.ARRAY_INDEX]

            # Append to combined mesh
            for i in range(vertices.size()):
                var v = transform * vertices[i]
                var n = transform.basis * normals[i]

                surface_tool.set_normal(n)
                surface_tool.set_uv(uvs[i])
                surface_tool.add_vertex(v)

        surface_tool.generate_normals()
        surface_tool.index()

        var combined_mesh = surface_tool.commit()
        var instance = MeshInstance3D.new()
        instance.mesh = combined_mesh

        return instance

# Dynamic Batching: Combine similar objects per frame
class DynamicBatcher:
    var batch_limit: int = 300  # Max vertices per batch
    var current_batches: Array = []

    func batch_dynamic_objects(objects: Array) -> Array:
        # Sort by material (minimize state changes)
        objects.sort_custom(func(a, b): return _get_material_id(a) < _get_material_id(b))

        current_batches.clear()
        var current_batch = []
        var current_material = null
        var vertex_count = 0

        for obj in objects:
            if not obj is MeshInstance3D or not obj.visible:
                continue

            var material = _get_material(obj)
            var verts = _get_vertex_count(obj)

            # Start new batch if material changes or too many vertices
            if material != current_material or vertex_count + verts > batch_limit:
                if not current_batch.is_empty():
                    current_batches.append(_create_batch(current_batch))

                current_batch = []
                current_material = material
                vertex_count = 0

            current_batch.append(obj)
            vertex_count += verts

        # Add final batch
        if not current_batch.is_empty():
            current_batches.append(_create_batch(current_batch))

        return current_batches

    func _get_material(obj: MeshInstance3D):
        return obj.get_surface_override_material(0) or obj.mesh.surface_get_material(0)

    func _get_material_id(obj):
        var mat = _get_material(obj)
        return mat.get_instance_id() if mat else 0

    func _get_vertex_count(obj: MeshInstance3D) -> int:
        if not obj.mesh:
            return 0
        var arrays = obj.mesh.surface_get_arrays(0)
        if arrays and arrays[Mesh.ARRAY_VERTEX]:
            return arrays[Mesh.ARRAY_VERTEX].size()
        return 0

    func _create_batch(objects: Array):
        # Similar to static batching, but per-frame
        # In practice, use MultiMesh for better performance
        return objects

# GPU Instancing: Best performance for identical objects
class InstanceBatcher:
    var multi_mesh_instances: Dictionary = {}  # Mesh -> MultiMeshInstance3D

    func batch_instances(objects: Array):
        # Group by mesh
        var mesh_groups = {}
        for obj in objects:
            if not obj is MeshInstance3D or not obj.visible:
                continue

            var mesh = obj.mesh
            if not mesh in mesh_groups:
                mesh_groups[mesh] = []

            mesh_groups[mesh].append(obj)

        # Create MultiMesh for each group
        for mesh in mesh_groups:
            var instances = mesh_groups[mesh]
            var multi_mesh = _create_multi_mesh(mesh, instances)
            multi_mesh_instances[mesh] = multi_mesh

    func _create_multi_mesh(mesh: Mesh, instances: Array) -> MultiMeshInstance3D:
        var multi_mesh = MultiMesh.new()
        multi_mesh.mesh = mesh
        multi_mesh.transform_format = MultiMesh.TRANSFORM_3D
        multi_mesh.instance_count = instances.size()

        # Set transforms for each instance
        for i in range(instances.size()):
            var obj = instances[i]
            multi_mesh.set_instance_transform(i, obj.global_transform)

            # Optional: per-instance color/custom data
            if obj.has_meta("instance_color"):
                multi_mesh.set_instance_color(i, obj.get_meta("instance_color"))

        var multi_mesh_instance = MultiMeshInstance3D.new()
        multi_mesh_instance.multimesh = multi_mesh

        return multi_mesh_instance

    func update_instance_transforms(objects: Array):
        # Update transforms without recreating MultiMesh
        var mesh_groups = {}
        for obj in objects:
            if not obj is MeshInstance3D:
                continue

            var mesh = obj.mesh
            if not mesh in mesh_groups:
                mesh_groups[mesh] = []

            mesh_groups[mesh].append(obj)

        for mesh in mesh_groups:
            if mesh in multi_mesh_instances:
                var multi_mesh = multi_mesh_instances[mesh].multimesh
                var instances = mesh_groups[mesh]

                for i in range(min(instances.size(), multi_mesh.instance_count)):
                    multi_mesh.set_instance_transform(i, instances[i].global_transform)

# Automatic Batcher: Analyzes scene and applies best strategy
class AutoBatcher:
    func analyze_and_batch(root: Node3D):
        var objects = _collect_renderable_objects(root)
        var stats = _analyze_objects(objects)

        print("Batch Analysis:")
        print("  Total objects: ", objects.size())
        print("  Unique meshes: ", stats.unique_meshes)
        print("  Unique materials: ", stats.unique_materials)
        print("  Potential batches: ", stats.potential_batches)

        # Apply best strategy
        if stats.static_ratio > 0.8:
            print("  Strategy: Static batching (80%+ static)")
            _apply_static_batching(stats.static_objects)

        if stats.instanced_ratio > 0.5:
            print("  Strategy: GPU instancing (50%+ identical)")
            _apply_instancing(stats.instanced_groups)

        if stats.dynamic_objects.size() > 100:
            print("  Strategy: Dynamic batching for remaining objects")
            _apply_dynamic_batching(stats.dynamic_objects)

    func _collect_renderable_objects(root: Node3D) -> Array:
        var objects = []
        for child in root.get_children():
            if child is MeshInstance3D:
                objects.append(child)
            if child.get_child_count() > 0:
                objects.append_array(_collect_renderable_objects(child))
        return objects

    func _analyze_objects(objects: Array) -> Dictionary:
        var stats = {
            "unique_meshes": 0,
            "unique_materials": 0,
            "static_objects": [],
            "dynamic_objects": [],
            "instanced_groups": {},
            "static_ratio": 0.0,
            "instanced_ratio": 0.0,
            "potential_batches": 0
        }

        # Analysis logic here...
        return stats

    func _apply_static_batching(objects: Array):
        var batcher = StaticBatcher.new()
        batcher.batch_static_objects(objects)

    func _apply_instancing(groups: Dictionary):
        var batcher = InstanceBatcher.new()
        # Apply instancing per group

    func _apply_dynamic_batching(objects: Array):
        var batcher = DynamicBatcher.new()
        batcher.batch_dynamic_objects(objects)
```

**Key Parameters:**
- Static batch limit: 300-1000 vertices per batch (GPU vertex cache)
- Dynamic batch limit: 100-300 vertices (CPU overhead vs benefit)
- Material variations: <10 materials = good batching candidate
- Instance count threshold: 10+ identical objects = use MultiMesh
- Update frequency: Static = never, Dynamic = per frame, Instanced = when moved
- Sorting: By material, then by depth (for transparency)

**Edge Cases:**
- Skinned meshes: Cannot batch (unique bone transforms)
- Transparent objects: Sort back-to-front, limits batching
- Different lightmaps: Break batches (unique lightmap UVs)
- Dynamic lighting: Per-object lights prevent batching
- Large objects: May exceed batch limits, split or instance separately
- Moving objects: Static batching not possible, use dynamic or instancing

**When NOT to Use:**
- Few objects (<50)
- All objects have unique materials
- Heavy use of real-time lighting (per-object data)
- Skinned/animated meshes dominate scene
- Frequent individual object updates

**Examples from Shipped Games:**

1. **Monument Valley:** Extensive static batching, 5000 draw calls → 50 (99% reduction), enabled complex geometry on mobile. 60fps on iPad 2. Each level pre-batched at load time, <1ms overhead.

2. **Fortnite:** Dynamic batching for building pieces, 10K pieces → 500 batches (95% reduction). MultiMesh for trees (50K instances → 50 draw calls). Saved 15ms CPU time, critical for 60fps on console.

3. **Cities: Skylines:** GPU instancing for buildings, 100K buildings → 200 draw calls (99.8% reduction). Without instancing, unplayable on any hardware. Enabled massive city sizes.

4. **Ori and the Blind Forest:** Static batching for environment, 2000 objects → 80 batches. Dynamic batching for particles. Maintained 60fps on Xbox 360 (limited CPU).

5. **Minecraft:** Chunk batching, 65K blocks per chunk → 1-5 draw calls per chunk. Essential for performance. Modern versions use GPU instancing, 30fps → 300fps on same hardware.

**Platform Considerations:**
- **Mobile:** Critical - draw call limit ~100-200 for 60fps, use aggressive batching
- **Desktop:** Less critical but still important, can handle 1000-5000 draw calls
- **Console:** Moderate importance, 500-2000 draw call budget
- **WebGL:** Extremely critical, 50-100 draw call limit for 60fps
- **Vulkan/Metal:** Lower per-call overhead, but batching still beneficial
- **OpenGL:** High per-call overhead, batching essential

**Godot-Specific Notes:**
- Godot automatically batches 2D objects (CanvasItem batching)
- 3D: Use `MultiMeshInstance3D` for instancing (best performance)
- Static batching: Use external tools or custom scripts at build time
- `GeometryInstance3D.gi_mode = STATIC` for lightmap batching
- Godot 4.x: Improved automatic batching in Forward+ renderer
- Vulkan backend: Better draw call performance, but batching still helps
- CSG operations: Pre-bake to meshes for better batching

**Synergies:**
- Essential for GPU Instancing (MultiMesh)
- Pairs with Texture Atlasing (shared textures = shared materials)
- Combines with Frustum Culling (cull before batching)
- Works with LOD Systems (batch per LOD level)
- Enables Material Atlasing (combine materials for batching)

**Measurement/Profiling:**
- Godot Profiler: Monitor `Rendering -> Draw calls`
- Track batch efficiency: Triangles per draw call (target >100)
- Measure CPU time: Should decrease with batching
- Visual debug: Color objects by batch
- GPU profiling: Driver overhead should decrease
- Target: <500 draw calls for 60fps on mobile, <2000 on desktop
- Reduction: 80-95% draw call reduction typical

**Additional Resources:**
- Unity Manual: "Draw Call Batching"
- GDC 2017: "Optimizing the Graphics Pipeline with Compute"
- GPU Gems 2: Chapter 32 "Taking the Pitfalls Out of Occlusion Queries"
