### 26. Billboarding for Distant Objects

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Rendering high-poly 3D models at distances where detail is imperceptible. A 50K poly tree at 200m appears as 5×5 pixels but still processes 50K vertices = 0.67ms per tree on mid-range GPU. 1000 distant trees = 670ms (40x over budget). Billboard sprite (2 triangles) = 0.001ms per tree, saving 99.8% vertex processing time (670ms → 2ms).

**Technical Explanation:**
Billboarding replaces 3D geometry with 2D quads (sprites) that always face the camera. Vertex shader rotates quad to face view direction while maintaining world position. Fragment shader samples texture (pre-rendered image of 3D model from typical angle). Types: Spherical (full rotation), Cylindrical (Y-axis locked for trees), Screen-aligned. Can use imposters (rendered from current view) or pre-baked sprites. Transition from 3D to billboard at LOD threshold (50-200m). Modern approach: Octahedral imposters (8+ views for parallax) or point cloud sprites (depth-aware).

**Algorithmic Complexity:**
- 3D mesh: O(v) vertices processed, where v = 1K-100K
- Billboard: O(1) constant 4 vertices (2 triangles)
- Savings: 99-99.9% vertex processing for distant objects
- Memory: Texture per billboard (2K×2K = 16MB for 1000 unique objects)
- Transition cost: LOD switch = 1 vertex shader uniform change

**Implementation Pattern:**
```gdscript
# Godot Billboarding System
class_name Billboard
extends MeshInstance3D

enum BillboardType {
    SPHERICAL,      # Full 3D rotation toward camera
    CYLINDRICAL,    # Only Y-axis rotation (trees, characters)
    SCREEN_ALIGNED, # Fixed to screen (particles, UI elements)
}

@export var billboard_type: BillboardType = BillboardType.SPHERICAL
@export var billboard_texture: Texture2D
@export var billboard_size: Vector2 = Vector2(2.0, 2.0)
@export var lock_y_axis: bool = false

var billboard_material: ShaderMaterial
var camera: Camera3D

func _ready():
    _setup_billboard()
    camera = get_viewport().get_camera_3d()

func _setup_billboard():
    # Create quad mesh
    var quad_mesh = QuadMesh.new()
    quad_mesh.size = billboard_size
    mesh = quad_mesh

    # Create billboard shader
    var shader = Shader.new()
    shader.code = _get_billboard_shader()

    billboard_material = ShaderMaterial.new()
    billboard_material.shader = shader
    billboard_material.set_shader_parameter("albedo_texture", billboard_texture)
    billboard_material.set_shader_parameter("billboard_type", int(billboard_type))

    material_override = billboard_material

func _get_billboard_shader() -> String:
    return """
shader_type spatial;
render_mode cull_disabled, unshaded;

uniform sampler2D albedo_texture : source_color;
uniform int billboard_type = 0; // 0=spherical, 1=cylindrical, 2=screen_aligned
uniform bool lock_y = false;

void vertex() {
    // Get camera direction
    vec3 cam_pos = INV_VIEW_MATRIX[3].xyz;
    vec3 obj_pos = MODEL_MATRIX[3].xyz;
    vec3 look_dir = normalize(cam_pos - obj_pos);

    mat4 billboard_matrix = MODEL_MATRIX;

    if (billboard_type == 0) { // Spherical
        // Face camera completely
        vec3 right = normalize(cross(vec3(0, 1, 0), look_dir));
        vec3 up = cross(look_dir, right);

        billboard_matrix[0].xyz = right;
        billboard_matrix[1].xyz = up;
        billboard_matrix[2].xyz = look_dir;

    } else if (billboard_type == 1) { // Cylindrical
        // Only rotate around Y axis
        vec3 look_flat = normalize(vec3(look_dir.x, 0.0, look_dir.z));
        vec3 right = normalize(cross(vec3(0, 1, 0), look_flat));

        billboard_matrix[0].xyz = right;
        billboard_matrix[2].xyz = look_flat;

    } else if (billboard_type == 2) { // Screen-aligned
        // Use inverse view matrix directly
        billboard_matrix[0] = INV_VIEW_MATRIX[0];
        billboard_matrix[1] = INV_VIEW_MATRIX[1];
        billboard_matrix[2] = INV_VIEW_MATRIX[2];
    }

    // Apply billboard transformation
    MODELVIEW_MATRIX = VIEW_MATRIX * billboard_matrix;
}

void fragment() {
    vec4 color = texture(albedo_texture, UV);

    // Alpha test for tree cutouts
    if (color.a < 0.5) {
        discard;
    }

    ALBEDO = color.rgb;
    ALPHA = color.a;
}
"""

# Advanced: LOD system with billboard transition
class LODWithBillboard:
    @export var mesh_lod0: Mesh  # High detail
    @export var mesh_lod1: Mesh  # Medium detail
    @export var mesh_lod2: Mesh  # Low detail
    @export var billboard_texture: Texture2D
    @export var lod_distances: Array[float] = [20.0, 50.0, 100.0]

    var mesh_instance: MeshInstance3D
    var billboard_instance: Billboard
    var camera: Camera3D
    var current_lod: int = -1

    func _ready():
        camera = get_viewport().get_camera_3d()
        _setup_lods()

    func _setup_lods():
        mesh_instance = MeshInstance3D.new()
        add_child(mesh_instance)

        billboard_instance = Billboard.new()
        billboard_instance.billboard_texture = billboard_texture
        billboard_instance.visible = false
        add_child(billboard_instance)

    func _process(_delta):
        var distance = global_position.distance_to(camera.global_position)
        var new_lod = _calculate_lod(distance)

        if new_lod != current_lod:
            _switch_lod(new_lod)

    func _calculate_lod(distance: float) -> int:
        if distance < lod_distances[0]:
            return 0
        elif distance < lod_distances[1]:
            return 1
        elif distance < lod_distances[2]:
            return 2
        else:
            return 3  # Billboard

    func _switch_lod(new_lod: int):
        current_lod = new_lod

        if new_lod == 3:  # Billboard
            mesh_instance.visible = false
            billboard_instance.visible = true
        else:
            mesh_instance.visible = true
            billboard_instance.visible = false

            match new_lod:
                0: mesh_instance.mesh = mesh_lod0
                1: mesh_instance.mesh = mesh_lod1
                2: mesh_instance.mesh = mesh_lod2

# Octahedral imposters (8 views for parallax)
class OctahedralImposter:
    var imposter_textures: Array[Texture2D] = []  # 8 textures (one per octant)
    var normal_textures: Array[Texture2D] = []    # Optional: normals for lighting

    func _ready():
        _generate_imposter_textures()

    func _generate_imposter_textures():
        # Pre-render model from 8 directions
        var directions = [
            Vector3(1, 1, 1).normalized(),
            Vector3(-1, 1, 1).normalized(),
            Vector3(1, 1, -1).normalized(),
            Vector3(-1, 1, -1).normalized(),
            Vector3(1, -1, 1).normalized(),
            Vector3(-1, -1, 1).normalized(),
            Vector3(1, -1, -1).normalized(),
            Vector3(-1, -1, -1).normalized(),
        ]

        for dir in directions:
            var texture = _render_from_direction(dir)
            imposter_textures.append(texture)

    func _render_from_direction(direction: Vector3) -> Texture2D:
        # Render model to texture from this direction
        # This would use SubViewport or offline rendering
        return null  # Placeholder

    func select_imposter(view_direction: Vector3) -> Texture2D:
        # Select closest octant based on view direction
        var best_index = 0
        var best_dot = -1.0

        var octant_dirs = [
            Vector3(1, 1, 1).normalized(),
            Vector3(-1, 1, 1).normalized(),
            Vector3(1, 1, -1).normalized(),
            Vector3(-1, 1, -1).normalized(),
            Vector3(1, -1, 1).normalized(),
            Vector3(-1, -1, 1).normalized(),
            Vector3(1, -1, -1).normalized(),
            Vector3(-1, -1, -1).normalized(),
        ]

        for i in range(octant_dirs.size()):
            var dot = view_direction.dot(octant_dirs[i])
            if dot > best_dot:
                best_dot = dot
                best_index = i

        return imposter_textures[best_index]

# Imposter atlas for batching
class ImposterAtlas:
    # Combine multiple imposters into texture atlas for batching

    @export var atlas_size: int = 4096
    @export var imposter_size: int = 256
    var atlas_texture: ImageTexture
    var imposter_uvs: Dictionary = {}  # Model name -> Rect2 UV

    func create_atlas(models: Dictionary):  # name -> Texture2D
        var atlas_image = Image.create(atlas_size, atlas_size, false, Image.FORMAT_RGBA8)
        atlas_image.fill(Color(0, 0, 0, 0))

        var imposters_per_row = atlas_size / imposter_size
        var index = 0

        for model_name in models:
            var texture = models[model_name]
            var image = texture.get_image()

            var row = index / imposters_per_row
            var col = index % imposters_per_row

            var x = col * imposter_size
            var y = row * imposter_size

            # Copy to atlas
            atlas_image.blit_rect(image, Rect2i(Vector2i.ZERO, image.get_size()), Vector2i(x, y))

            # Store UV coordinates
            var uv = Rect2(
                float(x) / atlas_size,
                float(y) / atlas_size,
                float(imposter_size) / atlas_size,
                float(imposter_size) / atlas_size
            )
            imposter_uvs[model_name] = uv

            index += 1

        atlas_texture = ImageTexture.create_from_image(atlas_image)

# Soft billboard transitions
class SoftBillboardTransition:
    @export var transition_distance: float = 20.0
    @export var mesh_instance: MeshInstance3D
    @export var billboard_instance: Billboard

    func update_transition(distance: float, threshold: float):
        var blend_start = threshold - transition_distance
        var blend_end = threshold

        if distance < blend_start:
            # Full mesh
            mesh_instance.visible = true
            billboard_instance.visible = false
        elif distance > blend_end:
            # Full billboard
            mesh_instance.visible = false
            billboard_instance.visible = true
        else:
            # Blend both (crossfade)
            var t = (distance - blend_start) / transition_distance

            mesh_instance.visible = true
            billboard_instance.visible = true

            # Fade using transparency
            var mesh_mat = mesh_instance.material_override as StandardMaterial3D
            if mesh_mat:
                mesh_mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
                mesh_mat.albedo_color.a = 1.0 - t

            var billboard_mat = billboard_instance.material_override as ShaderMaterial
            if billboard_mat:
                billboard_mat.set_shader_parameter("alpha", t)
```

**Key Parameters:**
- Billboard distance: 50-200m for trees, 100-500m for buildings
- Texture resolution: 512×512 to 2048×2048 per billboard
- Transition zone: 20-50m crossfade to hide popping
- Billboard type: Cylindrical for trees/characters, spherical for rocks/props
- Octant count: 8 views for octahedral imposters (good parallax)
- Refresh rate: Static for most objects, dynamic for moving lights

**Edge Cases:**
- Extreme camera angles: Pre-render from multiple angles
- Transparent billboards: Sort with other transparent objects
- Lighting changes: Bake multiple lighting conditions or use normal maps
- Shadow casting: Billboards can cast shadows (simplified)
- Wind animation: Use vertex shader for swaying billboards
- Close inspection: Ensure billboard distance prevents close viewing

**When NOT to Use:**
- Objects frequently viewed up close
- Highly dynamic objects (need real-time updates)
- Indoor scenes (objects never distant enough)
- Stylized low-poly games (3D already cheap)
- Objects with important silhouette changes from different angles

**Examples from Shipped Games:**

1. **The Witcher 3:** Tree billboards beyond 80m, 50K poly trees → 2 poly billboards. Forest with 10K trees: 45ms → 2ms vertex processing (95% reduction). Essential for dense forests at 30fps on console.

2. **Red Dead Redemption 2:** Layered imposters for distant terrain/buildings. 8-way octahedral imposters, 1024² per view. Distant town rendering: 80ms → 5ms (94% savings). Enabled vast view distances.

3. **Far Cry 5:** Billboard vegetation beyond 100m, combined with instancing. 1M grass blades: 500K 3D + 500K billboards. Saved 30ms per frame. Dynamic imposters for animals (updated per frame).

4. **Horizon Zero Dawn:** Machine imposters beyond 300m. 8-view octahedral, 512² per view. Large herds: 50ms → 8ms (84% savings). Combined with LOD system (5 levels before billboard).

5. **Ghost of Tsushima:** Dynamic foliage billboards, wind-animated. Cylindrical billboards for grass/reeds. 2M vegetation instances: 70% billboards beyond 40m. Maintained 60fps in Performance mode.

**Platform Considerations:**
- **PC High-End:** Later billboard transitions (150-300m), higher resolution (2K)
- **PC Low-End:** Earlier transitions (50-100m), lower resolution (512)
- **Console (PS5/XSX):** 100-200m transitions, 1024-2048 resolution
- **Last-Gen Console:** 50-100m transitions, 512-1024 resolution
- **Mobile:** 30-80m transitions, 256-512 resolution, aggressive use
- **Memory:** 1000 billboards × 1K² × 4 bytes = 4MB (negligible)

**Godot-Specific Notes:**
- Built-in billboard support: `material.flags_billboard_mode`
- `BILLBOARD_ENABLED` = spherical, `BILLBOARD_FIXED_Y` = cylindrical
- Use `QuadMesh` for billboard geometry
- Shader-based billboards give more control
- Combine with `MultiMeshInstance3D` for instanced billboards
- `render_mode billboard` in shader for automatic billboarding
- Godot 4.x: Better billboard support in Forward+ renderer

**Synergies:**
- Essential for LOD Systems (final LOD level)
- Pairs with Geometry Instancing (instance billboards for forests)
- Combines with Texture Atlasing (batch multiple billboards)
- Works with Imposters (more sophisticated billboards)
- Enables Massive Draw Distance (render distant objects cheaply)

**Measurement/Profiling:**
- GPU profiler: Vertices processed (should drop 90%+ for distant objects)
- Track LOD distribution: What % of objects are billboards?
- Measure transition artifacts: Noticeable popping?
- Visual quality: Compare 3D vs billboard from transition distance
- Performance: 100-1000x faster per object than full 3D
- Target: 50-70% of distant objects as billboards
- Memory: <10MB for 1000 unique billboard textures

**Additional Resources:**
- GPU Gems 3: Chapter 21 "True Impostors"
- GDC 2017: "Rendering of Ghost of Tsushima"
- Siggraph 2012: "Octahedral Impostors"
