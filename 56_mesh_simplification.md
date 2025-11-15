### 56. Mesh Simplification (LOD Generation)

**Category:** Performance - 3D Rendering / Asset Optimization

**Problem It Solves:** Rendering excessive detail for distant objects. A scene with 100 distant objects at 50K triangles each = 5M triangles, requiring 15M vertex transformations at 60fps = 900M transforms/sec. At 50 cycles/vertex on 2GHz GPU = 22.5ms, exceeding 16.67ms budget by 1.35x. With LOD reducing distant objects to 5K triangles (10x reduction), processing drops to 2.25ms.

**Technical Explanation:**
Mesh simplification algorithmically reduces polygon count while preserving visual appearance. Common algorithms: 1) Edge collapse - iteratively merge vertices by removing lowest-cost edges (Quadric Error Metric). 2) Vertex clustering - group nearby vertices and merge. 3) Plane decimation - remove nearly coplanar faces. Generate 3-5 LOD levels: LOD0 (100% detail, 0-20m), LOD1 (50%, 20-50m), LOD2 (25%, 50-100m), LOD3 (10%, 100m+). Automatically switch LODs based on camera distance, reducing vertex load by 70-90% for typical scenes.

**Algorithmic Complexity:**
- Quadric Error Metric: O(n log n) for n vertices, produces high-quality LODs
- Vertex clustering: O(n) but lower quality
- Runtime LOD selection: O(1) distance check per object
- Memory: 4-5 LOD levels = 1.5-2x original mesh size
- Performance gain: 70-90% vertex reduction in typical scene

**Implementation Pattern:**
```gdscript
# Godot implementation
class_name LODManager
extends Node3D

@export var lod_meshes: Array[Mesh] = []  # [LOD0, LOD1, LOD2, LOD3]
@export var lod_distances: Array[float] = [20.0, 50.0, 100.0, 200.0]
@export_range(0.0, 1.0) var lod_bias: float = 1.0  # Quality bias

var current_lod: int = 0
var mesh_instance: MeshInstance3D
var camera: Camera3D

func _ready():
    mesh_instance = get_node("MeshInstance3D")
    camera = get_viewport().get_camera_3d()

    if lod_meshes.is_empty():
        _generate_lods()

func _process(_delta):
    if camera == null:
        return

    var distance = global_position.distance_to(camera.global_position)
    var new_lod = _calculate_lod(distance)

    if new_lod != current_lod:
        _switch_lod(new_lod)

func _calculate_lod(distance: float) -> int:
    # Apply LOD bias for quality settings
    distance /= lod_bias

    for i in range(lod_distances.size()):
        if distance < lod_distances[i]:
            return i
    return lod_distances.size()  # Furthest LOD

func _switch_lod(new_lod: int):
    if new_lod >= lod_meshes.size():
        mesh_instance.visible = false  # Cull completely
        return

    mesh_instance.visible = true
    mesh_instance.mesh = lod_meshes[new_lod]
    current_lod = new_lod

# LOD generation using mesh decimation
func _generate_lods():
    if mesh_instance == null or mesh_instance.mesh == null:
        return

    var original_mesh = mesh_instance.mesh
    lod_meshes.clear()
    lod_meshes.append(original_mesh)  # LOD0 = original

    # Generate progressively simplified versions
    var reduction_ratios = [0.5, 0.25, 0.1, 0.05]
    for ratio in reduction_ratios:
        var simplified = _simplify_mesh(original_mesh, ratio)
        lod_meshes.append(simplified)

func _simplify_mesh(mesh: Mesh, target_ratio: float) -> Mesh:
    # Godot doesn't have built-in mesh simplification
    # Use MeshDataTool for custom implementation or external tools

    var mdt = MeshDataTool.new()
    mdt.create_from_surface(mesh, 0)

    var target_vertex_count = int(mdt.get_vertex_count() * target_ratio)

    # Simplified edge collapse algorithm
    while mdt.get_vertex_count() > target_vertex_count:
        var edge_to_collapse = _find_lowest_cost_edge(mdt)
        if edge_to_collapse == null:
            break
        _collapse_edge(mdt, edge_to_collapse)

    var new_mesh = ArrayMesh.new()
    mdt.commit_to_surface(new_mesh)
    return new_mesh

func _find_lowest_cost_edge(mdt: MeshDataTool) -> Dictionary:
    var min_cost = INF
    var best_edge = null

    # Iterate through all edges, calculate collapse cost
    for i in range(mdt.get_vertex_count()):
        var edges = mdt.get_vertex_edges(i)
        for edge in edges:
            var cost = _calculate_edge_collapse_cost(mdt, i, edge)
            if cost < min_cost:
                min_cost = cost
                best_edge = {"v1": i, "v2": edge}

    return best_edge

func _calculate_edge_collapse_cost(mdt: MeshDataTool, v1: int, v2: int) -> float:
    # Quadric Error Metric calculation
    var pos1 = mdt.get_vertex(v1)
    var pos2 = mdt.get_vertex(v2)

    # Simple cost: edge length + curvature penalty
    var length_cost = pos1.distance_to(pos2)

    var normal1 = mdt.get_vertex_normal(v1)
    var normal2 = mdt.get_vertex_normal(v2)
    var curvature_cost = 1.0 - normal1.dot(normal2)

    return length_cost + curvature_cost * 10.0

# Automatic LOD system with hysteresis
class_name AutoLODSystem

var lod_objects: Array[LODManager] = []
const HYSTERESIS_FACTOR = 1.1  # Prevent LOD thrashing

func update_lods(camera: Camera3D):
    for obj in lod_objects:
        var distance = obj.global_position.distance_to(camera.global_position)
        var desired_lod = obj._calculate_lod(distance)

        # Apply hysteresis: only switch if clearly in new LOD range
        if desired_lod < obj.current_lod:
            # Switching to higher quality: immediate
            obj._switch_lod(desired_lod)
        elif desired_lod > obj.current_lod:
            # Switching to lower quality: add hysteresis
            var threshold = obj.lod_distances[obj.current_lod] * HYSTERESIS_FACTOR
            if distance > threshold:
                obj._switch_lod(desired_lod)
```

**Key Parameters:**
- LOD count: 3-5 levels (sweet spot: 4 levels)
- LOD0 distance: 0-20m (full detail)
- LOD1 distance: 20-50m (50% triangles)
- LOD2 distance: 50-100m (25% triangles)
- LOD3 distance: 100-200m (10% triangles)
- Culling distance: 200m+ (completely hidden)
- Hysteresis factor: 1.05-1.15 to prevent LOD popping

**Edge Cases:**
- LOD popping: Use smooth transitions or cross-fade between LODs
- Silhouette preservation: Maintain outer edges even at low LOD
- UV seam preservation: Keep texture mapping consistent across LODs
- Skeletal animation: Ensure bone weights are preserved or recalculated
- Vertex colors/attributes: Preserve during simplification
- Very low poly counts: Below 100 triangles, manual modeling better than automatic

**When NOT to Use:**
- Small objects (<2m) that are rarely distant
- Skyboxes and backgrounds (already optimized)
- UI elements (screen-space, not affected by distance)
- Highly geometric shapes (spheres, cubes) where simplification adds overhead
- Procedurally generated geometry (simplify algorithm, not output)

**Examples from Shipped Games:**

1. **Horizon Zero Dawn:** 5 LOD levels for machines. Thunderjaw: LOD0 (550K tris), LOD1 (180K), LOD2 (60K), LOD3 (15K), LOD4 (3K). Enabled 30+ machines on screen at 30fps. LOD system saved 80% of vertex processing in typical scenes.

2. **The Witcher 3:** Aggressive LOD on vegetation and buildings. Trees: LOD0 (8K tris) to LOD3 (200 tris). Generated 4 LODs per asset automatically. Enabled vast forests without vertex bottleneck, reducing geometry by 90% beyond 50m.

3. **Spider-Man (PS4):** Building LODs with 7 levels. Skyscrapers visible from miles away using 500-triangle LODs. Automatic LOD generation from photogrammetry scans. Reduced citywide geometry from 5B to 50M active triangles via aggressive LODing.

4. **Assassin's Creed Unity:** Buildings with 5 LOD levels. Notre Dame: LOD0 (300K tris) for interior, LOD4 (5K) for distant skyline. Hysteresis system prevented LOD popping during parkour. Enabled densest city in AC history.

5. **Doom 2016:** Demon LODs: 4 levels. Hell Knight: LOD0 (25K tris) to LOD3 (2K). Dynamic LOD bias based on GPU load - reduced quality when frame rate dropped. Maintained 60fps lock with 50+ demons.

**Platform Considerations:**
- **PC High-End:** Use 4-5 LODs, aggressive distances (50m, 100m, 200m, 500m)
- **Console (PS4/Xbox):** 3-4 LODs, conservative distances (20m, 50m, 100m)
- **Mobile:** 2-3 LODs mandatory, aggressive switching (10m, 30m)
- **VR:** Critical for 90fps, use 5 LODs with very aggressive distances
- **Memory:** Each LOD adds 10-50% to mesh memory, budget accordingly
- **Switch:** Essential - use aggressive LOD bias, reduce distances by 50%

**Godot-Specific Notes:**
- Godot 4.x has built-in LOD support: `MeshInstance3D.set_lod_bias()` and automatic HLOD
- Use `GeometryInstance3D.set_custom_aabb()` to control LOD distance calculation
- Import settings: Enable "Generate LODs" for automatic creation from FBX/GLTF
- Manual LOD: Use visibility ranges via `VisibilityRange` node in Godot 4.2+
- For custom LOD generation, use MeshDataTool or external tools (Blender, Simplygon)
- `RenderingServer.mesh_surface_get_format_offset()` for format preservation
- Profile: Monitor triangle count via "Visible Information" debug overlay

**Synergies:**
- **Occlusion Culling:** Don't render low LODs if occluded anyway
- **Normal Mapping (Technique 55):** Transfer lost detail to normal maps
- **Frustum Culling:** Skip LOD calculation for off-screen objects
- **Impostors:** Replace furthest LOD with billboard textures
- **Streaming:** Load/unload LOD levels based on memory pressure

**Measurement/Profiling:**
- **Triangle count:** Monitor `RenderingServer.get_rendering_info(RENDERING_INFO_TOTAL_PRIMITIVES_IN_FRAME)`
- **Vertex processing:** GPU profiler (RenderDoc) shows vertex shader time
- **LOD distribution:** Log how many objects at each LOD level
- **Visual quality:** Screenshot comparisons at various distances
- **Performance:** Target 50-70% triangle reduction in average scenes
- **Switching frequency:** Should be <5 LOD switches per second to avoid popping
- **Profile:** Godot's "Visible Information" overlay shows active triangle count

---
