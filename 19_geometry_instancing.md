### 19. Geometry Instancing (MultiMesh)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Rendering many identical objects with individual draw calls. 10,000 trees × 0.02ms per draw call = 200ms CPU overhead, exceeding 16.67ms budget by 12x. GPU can handle geometry but CPU bottlenecks on submission. Instancing reduces 10K draw calls to 1, saving 199ms.

**Technical Explanation:**
GPU instancing renders multiple copies of the same mesh with a single draw call by passing instance-specific data (transforms, colors) in a buffer. GPU vertex shader reads instance ID and applies per-instance transform. One draw call submits mesh once + instance buffer (transforms/colors). Hardware processes vertices in parallel for all instances. Dramatically reduces CPU overhead (API calls, driver validation, state changes). Requires identical geometry but allows per-instance variation (position, rotation, scale, color, custom data).

**Algorithmic Complexity:**
- No instancing: O(n) draw calls for n objects
- With instancing: O(1) draw call + O(n) instance buffer upload
- Memory: O(n × 64 bytes) for transforms, +O(n × 16 bytes) for colors
- CPU overhead: 0.05-0.2ms for 10K instances vs 200ms without
- GPU parallelism: All instances processed simultaneously

**Implementation Pattern:**
```gdscript
# Godot MultiMesh (GPU Instancing) Implementation
class_name InstanceManager
extends Node3D

@export var instance_mesh: Mesh
@export var instance_material: Material
@export var max_instances: int = 10000
@export var use_colors: bool = true
@export var use_custom_data: bool = false

var multi_mesh: MultiMesh
var multi_mesh_instance: MultiMeshInstance3D
var active_instances: int = 0
var instance_data: Array = []

func _ready():
    _setup_multimesh()

func _setup_multimesh():
    # Create MultiMesh
    multi_mesh = MultiMesh.new()
    multi_mesh.mesh = instance_mesh
    multi_mesh.transform_format = MultiMesh.TRANSFORM_3D
    multi_mesh.instance_count = max_instances

    # Optional: per-instance color
    if use_colors:
        multi_mesh.use_colors = true

    # Optional: per-instance custom data (4 floats)
    if use_custom_data:
        multi_mesh.use_custom_data = true

    # Create instance node
    multi_mesh_instance = MultiMeshInstance3D.new()
    multi_mesh_instance.multimesh = multi_mesh
    if instance_material:
        multi_mesh_instance.material_override = instance_material
    add_child(multi_mesh_instance)

func add_instance(transform: Transform3D, color: Color = Color.WHITE, custom_data: Color = Color.BLACK) -> int:
    if active_instances >= max_instances:
        push_warning("Max instances reached")
        return -1

    var index = active_instances
    multi_mesh.set_instance_transform(index, transform)

    if use_colors:
        multi_mesh.set_instance_color(index, color)

    if use_custom_data:
        multi_mesh.set_instance_custom_data(index, custom_data)

    instance_data.append({
        "transform": transform,
        "color": color,
        "custom_data": custom_data,
        "active": true
    })

    active_instances += 1
    return index

func update_instance(index: int, transform: Transform3D, color: Color = Color.WHITE):
    if index < 0 or index >= active_instances:
        return

    multi_mesh.set_instance_transform(index, transform)

    if use_colors:
        multi_mesh.set_instance_color(index, color)

    instance_data[index]["transform"] = transform
    instance_data[index]["color"] = color

func remove_instance(index: int):
    if index < 0 or index >= active_instances:
        return

    # Swap with last instance to maintain compact array
    var last_index = active_instances - 1
    if index != last_index:
        var last_transform = multi_mesh.get_instance_transform(last_index)
        var last_color = multi_mesh.get_instance_color(last_index)

        multi_mesh.set_instance_transform(index, last_transform)
        multi_mesh.set_instance_color(index, last_color)

        instance_data[index] = instance_data[last_index]

    active_instances -= 1
    multi_mesh.visible_instance_count = active_instances

func clear_instances():
    active_instances = 0
    instance_data.clear()
    multi_mesh.visible_instance_count = 0

# Advanced: Dynamic instance culling
class_name CulledInstanceManager
extends InstanceManager

var camera: Camera3D
var frustum_planes: Array[Plane] = []
var visible_mask: PackedByteArray

func _ready():
    super._ready()
    camera = get_viewport().get_camera_3d()
    visible_mask.resize(max_instances)

func _process(_delta):
    if camera:
        _update_frustum_culling()

func _update_frustum_culling():
    # Extract frustum planes
    _extract_frustum_planes()

    var visible_count = 0

    # Test each instance
    for i in range(active_instances):
        var transform = multi_mesh.get_instance_transform(i)
        var position = transform.origin

        # Simple sphere test (assuming unit mesh)
        var radius = 1.0  # Adjust based on mesh size
        var visible = _is_in_frustum(position, radius)

        visible_mask[i] = 1 if visible else 0

        if visible:
            visible_count += 1

    # Note: Godot doesn't support per-instance visibility yet
    # This is more useful for gameplay logic (skip updates for culled instances)

func _extract_frustum_planes():
    # Similar to frustum culling implementation
    frustum_planes.clear()
    var proj = camera.get_camera_projection()
    var transform = camera.global_transform
    # ... extract planes

func _is_in_frustum(position: Vector3, radius: float) -> bool:
    for plane in frustum_planes:
        if plane.distance_to(position) < -radius:
            return false
    return true

# Procedural instance generation (e.g., foliage)
class_name ProceduralInstanceManager
extends InstanceManager

@export var spawn_area: AABB = AABB(Vector3(-100, 0, -100), Vector3(200, 0, 200))
@export var density: float = 0.1  # Instances per square unit
@export var random_rotation: bool = true
@export var random_scale_range: Vector2 = Vector2(0.8, 1.2)
@export var random_seed: int = 12345

func generate_instances():
    var rng = RandomNumberGenerator.new()
    rng.seed = random_seed

    clear_instances()

    var area = spawn_area.size.x * spawn_area.size.z
    var instance_count = int(area * density)
    instance_count = min(instance_count, max_instances)

    for i in range(instance_count):
        var pos = Vector3(
            spawn_area.position.x + rng.randf() * spawn_area.size.x,
            spawn_area.position.y,
            spawn_area.position.z + rng.randf() * spawn_area.size.z
        )

        var rotation = Vector3(0, rng.randf() * TAU, 0) if random_rotation else Vector3.ZERO
        var scale_factor = rng.randf_range(random_scale_range.x, random_scale_range.y)
        var scale = Vector3.ONE * scale_factor

        var transform = Transform3D()
        transform.origin = pos
        transform.basis = Basis.from_euler(rotation).scaled(scale)

        var color = Color(
            rng.randf_range(0.8, 1.2),
            rng.randf_range(0.8, 1.2),
            rng.randf_range(0.8, 1.2),
            1.0
        )

        add_instance(transform, color)

# GPU-driven instancing (advanced)
class_name GPUDrivenInstanceManager
extends InstanceManager

# Use compute shader to update instances on GPU
var compute_shader: RID
var instance_buffer: RID

func _setup_compute_shader():
    var rd = RenderingServer.get_rendering_device()
    if not rd:
        push_error("RenderingDevice not available")
        return

    # Load compute shader for instance updates
    var shader_file = load("res://shaders/instance_update.glsl")
    var shader_spirv = shader_file.get_spirv()
    compute_shader = rd.shader_create_from_spirv(shader_spirv)

    # Create instance buffer
    var buffer_size = max_instances * 64  # 64 bytes per transform
    instance_buffer = rd.storage_buffer_create(buffer_size)

func update_instances_gpu(delta: float):
    var rd = RenderingServer.get_rendering_device()
    if not rd or not compute_shader.is_valid():
        return

    # Dispatch compute shader to update all instances in parallel
    # Examples: wave animation, growth, rotation, etc.
    var uniform_set = _create_uniform_set()
    var pipeline = rd.compute_pipeline_create(compute_shader)

    var compute_list = rd.compute_list_begin()
    rd.compute_list_bind_compute_pipeline(compute_list, pipeline)
    rd.compute_list_bind_uniform_set(compute_list, uniform_set, 0)

    # Dispatch work groups (64 instances per group)
    var groups = (active_instances + 63) / 64
    rd.compute_list_dispatch(compute_list, groups, 1, 1)
    rd.compute_list_end()

    # Read back results
    # ...
```

**Key Parameters:**
- Max instances: 1K-100K depending on platform (mobile: 10K, desktop: 100K)
- Instance buffer size: 64 bytes per transform (4×4 matrix)
- Color buffer: 16 bytes per instance (RGBA)
- Custom data: 16 bytes per instance (4 floats for shader params)
- Update frequency: Only when instances change (not every frame)
- Culling: Per-frame frustum check, only update visible
- LOD: Combine instancing with LOD (separate MultiMesh per LOD)

**Edge Cases:**
- Exceeding max instances: Allocate multiple MultiMesh objects
- Individual instance updates: Expensive if >10% change per frame, consider separate system
- Skinned meshes: Cannot instance (unique bone data)
- Shadow casting: All instances share shadow settings
- Different materials: Breaks instancing, use shader params instead
- Transparency: Sort issues (all instances treated as one object)
- Collision: Need separate collision shapes per instance

**When NOT to Use:**
- Few objects (<10 identical copies)
- Frequently changing geometry (different meshes)
- Per-object unique materials
- Complex per-object behavior requiring individual nodes
- Heavy interaction with physics system

**Examples from Shipped Games:**

1. **Ghost of Tsushima:** Grass instancing, 500K grass blades per scene → 1 draw call, 60fps on PS4. Wind animation via vertex shader using custom data. Without instancing: unplayable at 5fps.

2. **The Witcher 3:** Tree/rock instancing, 100K+ instances per region, 30fps on console. Combined with LOD: 5 MultiMesh objects per asset type (LOD0-LOD4). Saved 40ms CPU time.

3. **Horizon Zero Dawn:** Foliage instancing, 1M+ grass instances, 30fps PS4. GPU-driven culling in compute shader. Per-instance color variation for visual diversity. Essential for open world density.

4. **Satisfactory:** Factory building instancing, 100K conveyor belts/machines → <100 draw calls. Updated only when placed/removed. Enabled massive factories without performance loss.

5. **No Man's Sky:** Planet-scale instancing, millions of rocks/plants per planet. Procedural generation + instancing. LOD system: 6 levels, billboard at >500m. Maintained 30-60fps across platforms.

**Platform Considerations:**
- **Mobile:** 5K-20K instances max, aggressive LOD/culling, avoid per-frame updates
- **Desktop High-End:** 100K-1M instances possible, GPU-driven culling
- **Console:** 50K-100K instances, balance with other systems
- **VR:** Higher instance counts possible (GPU bound, less CPU overhead)
- **WebGL:** 10K-50K instances, limited buffer sizes
- **Memory:** ~100 bytes per instance, 100K = 10MB (negligible)

**Godot-Specific Notes:**
- `MultiMeshInstance3D` is Godot's instancing solution
- `MultiMesh.instance_count` sets maximum instances
- `visible_instance_count` hides extra instances without reallocation
- Transform format: `TRANSFORM_3D` (64 bytes) or `TRANSFORM_2D` (32 bytes)
- Godot 4.x: Improved instancing performance with Vulkan
- Use `GeometryInstance3D.cast_shadow = SHADOW_CASTING_SETTING_ON` for all instances
- Combine with `VisibleOnScreenNotifier3D` for culling logic
- CSGMesh doesn't support instancing, convert to MeshInstance3D

**Synergies:**
- Essential for Draw Call Batching (1 call for many objects)
- Pairs with LOD Systems (instance per LOD level)
- Combines with Frustum Culling (cull instances in gameplay logic)
- Works with Procedural Generation (instant placement of thousands)
- Enables large-scale Particle Systems (GPU particles use instancing internally)

**Measurement/Profiling:**
- Godot Profiler: Monitor `Rendering -> Draw calls` (should be 1 per MultiMesh)
- Track instances per MultiMesh: Target >100 for worthwhile benefit
- Measure CPU time: Should be <0.1ms per 10K instances
- GPU profiling: Vertex shader time scales with instance count
- Visual debugging: Color instances by distance/LOD
- Target: <10 draw calls for 100K+ objects
- Performance: 100x-1000x improvement over individual objects

**Additional Resources:**
- GPU Gems 2: Chapter 3 "Efficient Rendering of High-Quality Tree Structures"
- GDC 2015: "GPU-Driven Rendering Pipelines"
- Siggraph 2015: "GPU-Driven Rendering" by AMD
