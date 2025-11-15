### 16. LOD (Level of Detail) Systems

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** GPU overload from rendering high-polygon models at all distances. A 100K poly character at 500m distance wastes 99.9% of vertices (sub-pixel triangles). Rendering 100 such models = 10M vertices/frame = 133ms at 75M verts/sec, exceeding 16.67ms budget by 8x.

**Technical Explanation:**
LOD systems dynamically swap mesh complexity based on distance/screen coverage. Each model has 3-5 variants: LOD0 (full detail), LOD1 (50% polys), LOD2 (25%), LOD3 (10%), LOD4 (billboard/imposter). Distance thresholds trigger transitions. Modern systems use smooth transitions (dithering/alpha blending) to hide pops. CPU calculates screen space area (projected bbox size), selects appropriate LOD. GPU only processes necessary vertices.

**Algorithmic Complexity:**
- No LOD: O(n × v) where n=objects, v=vertices per object
- With LOD: O(n × v_avg) where v_avg << v for distant objects
- LOD selection: O(n) per frame (distance check per object)
- Memory: 1.5-2.5x model storage for all LOD levels

**Implementation Pattern:**
```gdscript
# Godot LOD System Implementation
class_name LODManager
extends Node3D

@export var lod_meshes: Array[Mesh] = []  # LOD0 (highest) to LOD3 (lowest)
@export var lod_distances: Array[float] = [10.0, 30.0, 80.0, 200.0]
@export var use_screen_coverage: bool = false
@export var screen_coverage_thresholds: Array[float] = [0.2, 0.1, 0.05, 0.02]

var mesh_instance: MeshInstance3D
var current_lod: int = 0
var camera: Camera3D

func _ready():
    mesh_instance = get_node("MeshInstance3D")
    camera = get_viewport().get_camera_3d()

func _process(_delta):
    if not camera or lod_meshes.is_empty():
        return

    var new_lod = _calculate_lod()
    if new_lod != current_lod:
        _switch_lod(new_lod)

func _calculate_lod() -> int:
    if use_screen_coverage:
        var coverage = _get_screen_coverage()
        for i in range(screen_coverage_thresholds.size()):
            if coverage > screen_coverage_thresholds[i]:
                return i
        return lod_meshes.size() - 1
    else:
        var distance = global_position.distance_to(camera.global_position)
        for i in range(lod_distances.size()):
            if distance < lod_distances[i]:
                return i
        return lod_meshes.size() - 1

func _get_screen_coverage() -> float:
    # Calculate screen-space size
    var aabb = mesh_instance.get_aabb()
    var center = global_position
    var radius = aabb.size.length() * 0.5

    var cam_to_obj = (center - camera.global_position).length()
    if cam_to_obj < 0.01:
        return 1.0

    # Project sphere to screen space
    var fov_rad = deg_to_rad(camera.fov)
    var projected_radius = (radius / cam_to_obj) / tan(fov_rad * 0.5)
    var coverage = projected_radius * 0.5  # Approximate coverage

    return clamp(coverage, 0.0, 1.0)

func _switch_lod(new_lod: int):
    if new_lod >= 0 and new_lod < lod_meshes.size():
        mesh_instance.mesh = lod_meshes[new_lod]
        current_lod = new_lod

# Advanced: Dithered LOD transitions to hide popping
class_name DitheredLOD
extends LODManager

@export var transition_duration: float = 0.3
var transition_timer: float = 0.0
var transitioning: bool = false
var old_lod_instance: MeshInstance3D

func _switch_lod(new_lod: int):
    if new_lod == current_lod:
        return

    # Create temporary instance for old LOD
    if transitioning and old_lod_instance:
        old_lod_instance.queue_free()

    old_lod_instance = MeshInstance3D.new()
    old_lod_instance.mesh = lod_meshes[current_lod]
    add_child(old_lod_instance)

    # Switch main mesh
    mesh_instance.mesh = lod_meshes[new_lod]
    current_lod = new_lod

    # Start transition
    transitioning = true
    transition_timer = 0.0

func _process(delta):
    super._process(delta)

    if transitioning:
        transition_timer += delta
        var progress = transition_timer / transition_duration

        if old_lod_instance:
            var mat = old_lod_instance.get_surface_override_material(0)
            if mat:
                mat.albedo_color.a = 1.0 - progress

        if progress >= 1.0:
            if old_lod_instance:
                old_lod_instance.queue_free()
                old_lod_instance = null
            transitioning = false
```

**Key Parameters:**
- LOD levels: 3-5 (diminishing returns beyond 5)
- Distance thresholds: [15m, 40m, 100m, 300m] for characters
- Vertex reduction: 50-60% per level (exponential falloff)
- Transition zones: 10-20% of distance threshold
- Screen coverage: >20% = LOD0, 10-20% = LOD1, 5-10% = LOD2, <5% = LOD3
- Update frequency: Every 5-10 frames (not every frame)

**Edge Cases:**
- LOD popping: Use dithered/crossfade transitions (0.1-0.3s)
- Hysteresis: Add 10-15% buffer to prevent thrashing (distance +/- 10%)
- Camera cuts: Force immediate LOD update, skip transitions
- Many objects: Update LODs in batches (100-200 per frame)
- Occluded objects: Skip LOD updates when not visible
- High-speed movement: Increase LOD distances by velocity factor

**When NOT to Use:**
- Small game worlds (<100m view distance)
- Few unique models (<10)
- Low poly aesthetic (all models <1K polys)
- Stylized games where geometry is already optimized
- VR: More aggressive LODs needed, but transitions more noticeable

**Examples from Shipped Games:**

1. **Forza Horizon 5:** 7-8 LOD levels per car, 500K polys (LOD0) → 500 polys (LOD7), 60fps on Xbox Series S. Distance: 2m, 10m, 30m, 80m, 200m, 500m, 1500m. Saved 70% GPU time in traffic scenarios.

2. **The Witcher 3:** Character LODs: 25K → 12K → 6K → 1.5K → 200 poly billboard. Distances: 5m, 15m, 50m, 150m. Combined with texture LODs, reduced memory by 40% on console.

3. **Microsoft Flight Simulator:** Terrain LODs up to 20 levels, 50cm/pixel → 10m/pixel, enables planet-scale rendering. Dynamic LOD based on altitude and speed, 30-60fps on mid-range PCs.

4. **Far Cry 6:** Vegetation LODs: 4 levels, 5K leaves → 500 → 50 → billboard. Transitions at 20m, 60m, 150m. Enabled dense jungle environments at 60fps on PS5.

5. **Red Dead Redemption 2:** Horses: 70K polys (player) → 500 polys (distant), 8 LOD levels. Dynamic LOD bias based on importance (player horse always highest). Maintained 30fps with 20+ horses on screen.

**Platform Considerations:**
- **PC High-End:** More aggressive LOD0/1 distances (maintain quality at 4K)
- **PC Low-End:** Earlier LOD transitions, skip LOD0 entirely if needed
- **Console (PS5/XSX):** Balance quality/performance, 4-5 LOD levels
- **Last-Gen Console:** 3-4 LOD levels, aggressive transitions at 15m/30m/80m
- **Mobile:** 2-3 LOD levels max, billboard at 30-50m, LOD0 only for player
- **VR:** More conservative LODs (higher baseline quality), but earlier transitions

**Godot-Specific Notes:**
- Godot 4.x: Automatic LOD support in `GeometryInstance3D.lod_bias`
- Use `VisibleOnScreenNotifier3D` to disable LOD updates for offscreen objects
- LOD meshes share materials to reduce draw calls
- Consider using `MultiMeshInstance3D` for instanced LODs (trees, rocks)
- Import LOD meshes: Use Blender's Decimate modifier, export as separate files
- Godot's automatic LOD: Set in import settings, engine handles transitions
- Custom LOD: More control, better for specific game requirements

**Synergies:**
- Pairs with Frustum Culling (skip LOD updates for culled objects)
- Combines with Occlusion Culling (only update visible LODs)
- Enhances MultiMesh instancing (instance by LOD level)
- Works with Texture LOD/Mipmapping (coordinate texture and geometry LODs)
- Billboarding (final LOD level often a billboard sprite)

**Measurement/Profiling:**
- Godot Profiler: Monitor vertices rendered (`Rendering -> Vertices`)
- Track LOD distribution: How many objects at each level?
- Measure transition frequency: Should be <5 per object per second
- Profile LOD selection cost: <0.1ms for 1000 objects
- GPU profiling: Vertex shader time should decrease with distance
- Visual debugging: Color-code objects by LOD level (red=LOD0, green=LOD1, etc.)
- Target: 30-50% vertex reduction in typical gameplay scenes

**Additional Resources:**
- GDC 2019: "Controlling Geometric Complexity in Forza Horizon 4"
- Epic Games: Unreal Engine LOD documentation
- GPU Gems 3: Chapter 12 "High-Quality Ambient Occlusion"
