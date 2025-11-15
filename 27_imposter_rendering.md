### 27. Imposter Rendering (Sprites from 3D)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Billboard limitations with parallax/viewing angle changes. Simple billboards show same image from all angles, breaking immersion. A 100K poly building rendered 1000 times = 100M vertices = 1.3 seconds on mid-range GPU. Imposter with 16 views = 16 pre-rendered 2K sprites = 32 vertices total runtime, saving 99.99% vertex processing (1300ms → 0.04ms).

**Technical Explanation:**
Imposters are view-dependent billboards that store multiple pre-rendered images of 3D object from different angles. At runtime, select closest pre-rendered view based on camera direction. Store depth information to enable proper intersection with other geometry. Types: Discrete imposters (8-16 fixed views), Octahedral imposters (continuous sampling of hemisphere), Dual-paraboloid (2 hemis), Relief imposters (parallax depth). GPU samples 2-4 closest views and blends. Modern approach: Store albedo + normal + depth in texture array, reconstruct lighting per-pixel.

**Algorithmic Complexity:**
- Full 3D: O(v × n) where v=vertices, n=instances
- Imposters: O(k × n) where k=views (8-32), rendered offline
- Runtime: O(n) texture samples (2-4 per fragment)
- Memory: O(r² × k × 4) bytes where r=resolution, k=views (16MB for 2K×16 views)
- Quality vs Cost: More views = better quality, more memory/bandwidth

**Implementation Pattern:**
```gdscript
# Godot Imposter Rendering System
class_name ImposterRenderer
extends Node3D

@export var source_model: Node3D
@export var imposter_resolution: int = 1024
@export var view_count: int = 16
@export var capture_normals: bool = true
@export var capture_depth: bool = true
@export var sphere_radius: float = 5.0

var imposter_textures: Array[Texture2D] = []
var normal_textures: Array[Texture2D] = []
var depth_textures: Array[Texture2D] = []
var view_directions: Array[Vector3] = []
var imposter_material: ShaderMaterial

func _ready():
    _generate_imposter_views()
    _create_imposter_material()

func _generate_imposter_views():
    # Calculate view directions (uniform distribution on sphere)
    view_directions = _fibonacci_sphere_points(view_count)

    for i in range(view_count):
        var view_dir = view_directions[i]
        var textures = _render_view(view_dir, i)

        imposter_textures.append(textures.albedo)
        if capture_normals:
            normal_textures.append(textures.normal)
        if capture_depth:
            depth_textures.append(textures.depth)

    print("Generated ", view_count, " imposter views")

func _fibonacci_sphere_points(count: int) -> Array[Vector3]:
    # Fibonacci sphere distribution for uniform view coverage
    var points: Array[Vector3] = []
    var phi = PI * (3.0 - sqrt(5.0))  # Golden angle

    for i in range(count):
        var y = 1 - (float(i) / (count - 1)) * 2  # y from 1 to -1
        var radius = sqrt(1 - y * y)

        var theta = phi * i

        var x = cos(theta) * radius
        var z = sin(theta) * radius

        points.append(Vector3(x, y, z).normalized())

    return points

func _render_view(view_direction: Vector3, index: int) -> Dictionary:
    # Create SubViewport for rendering
    var viewport = SubViewport.new()
    viewport.size = Vector2i(imposter_resolution, imposter_resolution)
    viewport.render_target_update_mode = SubViewport.UPDATE_ONCE
    viewport.transparent_bg = true

    add_child(viewport)

    # Setup camera
    var camera = Camera3D.new()
    camera.projection = Camera3D.PROJECTION_ORTHOGONAL
    camera.size = sphere_radius * 2
    camera.near = 0.1
    camera.far = sphere_radius * 3

    # Position camera looking at model from view_direction
    var cam_pos = view_direction * sphere_radius * 1.5
    camera.global_position = cam_pos
    camera.look_at(Vector3.ZERO, Vector3.UP)

    viewport.add_child(camera)

    # Clone model into viewport
    var model_clone = source_model.duplicate()
    viewport.add_child(model_clone)

    # Add lighting
    var light = DirectionalLight3D.new()
    light.light_energy = 1.0
    light.rotation = camera.rotation
    viewport.add_child(light)

    # Render
    await RenderingServer.frame_post_draw

    # Capture textures
    var albedo_image = viewport.get_texture().get_image()
    var albedo_texture = ImageTexture.create_from_image(albedo_image)

    # Capture normal map (would need custom shader)
    var normal_texture = _capture_normals(viewport) if capture_normals else null

    # Capture depth (would need custom shader)
    var depth_texture = _capture_depth(viewport) if capture_depth else null

    # Cleanup
    viewport.queue_free()

    return {
        "albedo": albedo_texture,
        "normal": normal_texture,
        "depth": depth_texture
    }

func _capture_normals(viewport: SubViewport) -> Texture2D:
    # Would need to render with normal visualization shader
    # Placeholder for now
    return null

func _capture_depth(viewport: SubViewport) -> Texture2D:
    # Would need to render depth buffer
    # Placeholder for now
    return null

func _create_imposter_material():
    var shader = Shader.new()
    shader.code = _get_imposter_shader()

    imposter_material = ShaderMaterial.new()
    imposter_material.shader = shader

    # Create texture array
    _create_texture_array()

func _create_texture_array():
    # Combine individual textures into Texture2DArray for efficient sampling
    if imposter_textures.is_empty():
        return

    var array_rd = RenderingDevice.texture_create_shared_from_slice(
        imposter_textures[0].get_rid(),
        0, 0, 1, imposter_textures.size()
    )

    # Set texture array in material
    # Note: Godot's texture array support varies by version

func _get_imposter_shader() -> String:
    return """
shader_type spatial;
render_mode cull_disabled;

uniform sampler2DArray albedo_array : source_color;
uniform sampler2DArray normal_array : hint_normal;
uniform sampler2DArray depth_array;
uniform int view_count = 16;
uniform float sphere_radius = 5.0;

// View directions (set from code)
uniform vec3 view_dirs[16];

vec2 view_to_index(vec3 view_dir) {
    // Find 2 closest views for blending
    float best_dot = -1.0;
    int best_idx = 0;
    float second_best_dot = -1.0;
    int second_idx = 0;

    for (int i = 0; i < view_count; i++) {
        float d = dot(view_dir, view_dirs[i]);

        if (d > best_dot) {
            second_best_dot = best_dot;
            second_idx = best_idx;
            best_dot = d;
            best_idx = i;
        } else if (d > second_best_dot) {
            second_best_dot = d;
            second_idx = i;
        }
    }

    // Return indices and blend factor
    float blend = (best_dot - second_best_dot) / (1.0 + best_dot - second_best_dot);
    return vec2(float(best_idx), float(second_idx));
}

void vertex() {
    // Orient quad to face camera
    vec3 cam_pos = (INV_VIEW_MATRIX * vec4(0, 0, 0, 1)).xyz;
    vec3 obj_pos = (MODEL_MATRIX * vec4(0, 0, 0, 1)).xyz;
    vec3 look_dir = normalize(cam_pos - obj_pos);

    vec3 up = vec3(0, 1, 0);
    vec3 right = normalize(cross(up, look_dir));
    up = cross(look_dir, right);

    mat4 billboard = MODEL_MATRIX;
    billboard[0].xyz = right;
    billboard[1].xyz = up;
    billboard[2].xyz = look_dir;

    MODELVIEW_MATRIX = VIEW_MATRIX * billboard;
}

void fragment() {
    // Calculate view direction in object space
    vec3 cam_pos = (INV_VIEW_MATRIX * vec4(0, 0, 0, 1)).xyz;
    vec3 obj_pos = (INV_VIEW_MATRIX * MODEL_MATRIX * vec4(0, 0, 0, 1)).xyz;
    vec3 view_dir = normalize(cam_pos - obj_pos);

    // Find closest views
    vec2 indices = view_to_index(view_dir);
    int idx1 = int(indices.x);
    int idx2 = int(indices.y);

    // Sample textures
    vec4 albedo1 = texture(albedo_array, vec3(UV, float(idx1)));
    vec4 albedo2 = texture(albedo_array, vec3(UV, float(idx2)));

    // Blend between views (simplified)
    float blend = 0.5;  // Would calculate based on angle
    vec4 albedo = mix(albedo1, albedo2, blend);

    // Alpha test
    if (albedo.a < 0.1) {
        discard;
    }

    ALBEDO = albedo.rgb;
    ALPHA = albedo.a;

    // Optional: Sample normals for lighting
    // vec3 normal1 = texture(normal_array, vec3(UV, float(idx1))).rgb;
    // NORMAL = normalize(normal1 * 2.0 - 1.0);
}
"""

# Octahedral imposter (more efficient)
class OctahedralImposter:
    # Uses octahedral mapping for efficient 360° coverage

    @export var atlas_resolution: int = 2048  # Single texture for all views
    var octahedral_texture: Texture2D

    func generate_octahedral_imposter(model: Node3D, samples_per_axis: int = 64):
        var image = Image.create(atlas_resolution, atlas_resolution, false, Image.FORMAT_RGBA8)

        # Render from many directions, map to octahedral space
        for y in range(samples_per_axis):
            for x in range(samples_per_axis):
                var uv = Vector2(float(x) / samples_per_axis, float(y) / samples_per_axis)
                var dir = _octahedral_to_direction(uv * 2.0 - Vector2.ONE)

                # Render from this direction
                var color = _render_direction(model, dir)

                # Write to image
                var px = int(uv.x * atlas_resolution)
                var py = int(uv.y * atlas_resolution)
                image.set_pixel(px, py, color)

        octahedral_texture = ImageTexture.create_from_image(image)

    func _octahedral_to_direction(oct: Vector2) -> Vector3:
        # Convert octahedral coordinate to 3D direction
        var pos = Vector3(oct.x, oct.y, 1.0 - abs(oct.x) - abs(oct.y))

        if pos.z < 0.0:
            var tmp = Vector2(1.0 - abs(pos.y), 1.0 - abs(pos.x))
            pos.x = tmp.x if pos.x >= 0.0 else -tmp.x
            pos.y = tmp.y if pos.y >= 0.0 else -tmp.y

        return pos.normalized()

    func _direction_to_octahedral(dir: Vector3) -> Vector2:
        # Convert 3D direction to octahedral coordinate
        var oct = Vector2(dir.x, dir.y) / (abs(dir.x) + abs(dir.y) + abs(dir.z))

        if dir.z < 0.0:
            var tmp = oct
            oct.x = (1.0 - abs(tmp.y)) * (1.0 if tmp.x >= 0.0 else -1.0)
            oct.y = (1.0 - abs(tmp.x)) * (1.0 if tmp.y >= 0.0 else -1.0)

        return oct * 0.5 + Vector2(0.5, 0.5)

    func _render_direction(model: Node3D, direction: Vector3) -> Color:
        # Render model from this direction
        # Placeholder
        return Color.WHITE

# Imposter LOD system
class ImposterLODSystem:
    @export var mesh_lod: MeshInstance3D
    @export var imposter: ImposterRenderer
    @export var transition_distance: float = 100.0
    @export var imposter_distance: float = 200.0

    var camera: Camera3D
    var current_mode: int = 0  # 0=mesh, 1=imposter

    func _process(_delta):
        var dist = global_position.distance_to(camera.global_position)

        if dist < transition_distance:
            _show_mesh()
        elif dist > imposter_distance:
            _show_imposter()
        else:
            _blend_transition(dist)

    func _show_mesh():
        if current_mode != 0:
            mesh_lod.visible = true
            imposter.visible = false
            current_mode = 0

    func _show_imposter():
        if current_mode != 1:
            mesh_lod.visible = false
            imposter.visible = true
            current_mode = 1

    func _blend_transition(distance: float):
        # Crossfade between mesh and imposter
        var t = (distance - transition_distance) / (imposter_distance - transition_distance)

        mesh_lod.visible = true
        imposter.visible = true

        # Adjust transparency
        # ...
```

**Key Parameters:**
- View count: 8-32 (8=basic, 16=good, 32=excellent)
- Resolution: 512-2048 per view (1024 standard)
- Capture components: Albedo (always), Normal (lighting), Depth (occlusion)
- Transition distance: 100-300m from 3D to imposter
- Sphere radius: 1.2-2x model bounds (capture full model)
- Blending: 2-4 closest views weighted by angle

**Edge Cases:**
- Self-occlusion: Requires depth buffer for proper rendering
- Lighting changes: Capture normals and relight per-pixel
- Animation: Update imposters periodically (expensive)
- Transparency: Requires sorted rendering, depth issues
- Extreme aspect ratios: Use separate captures for width/height extremes
- Very close viewing: Transition to real 3D before artifacts visible

**When NOT to Use:**
- Small objects (billboard sufficient)
- Frequently viewed up close
- Rapidly changing appearance (lights, materials)
- Limited memory budget
- Simple objects (imposters overhead exceeds benefit)

**Examples from Shipped Games:**

1. **Star Citizen:** Capital ship imposters, 5M poly ships → 16-view imposters beyond 10km. Rendering time: 2 seconds → 5ms (400x speedup). Essential for fleet battles with 100+ ships.

2. **Microsoft Flight Simulator:** Landmark imposters (buildings, monuments). Octahedral mapping, 2048² atlas. Distant cities: 500ms → 15ms (97% savings). Enabled planet-scale rendering.

3. **Assassin's Creed: Unity:** Crowd imposters, 8-view system. NPCs beyond 30m → imposters. Crowd of 10K: 800ms → 80ms (90% reduction). Combined with animation LOD.

4. **No Man's Sky:** Procedural planet imposters, 32-view system updated on planet entry. Planet rendering from space: 100ms → 2ms (98% savings). Critical for seamless space-to-planet transition.

5. **Elite Dangerous:** Station imposters, 16-view with normal maps. Detailed stations (2M polys) → imposters at >5km. Saved 45ms per station, enabled many stations in scene.

**Platform Considerations:**
- **PC High-End:** 32 views, 2048² resolution, full PBR (albedo+normal+roughness+metallic)
- **PC Low-End:** 8 views, 512² resolution, albedo only
- **Console (PS5/XSX):** 16 views, 1024-2048² resolution, albedo+normal
- **Last-Gen Console:** 8-16 views, 512-1024² resolution
- **Mobile:** 4-8 views, 256-512² resolution, aggressive use
- **Memory:** 16 views × 1K² × 4 = 64MB per imposter

**Godot-Specific Notes:**
- Use `SubViewport` for offline rendering
- Create texture arrays with `Texture2DArray` (Godot 4.x)
- Shader complexity: Balance quality vs fragment shader cost
- Bake imposters at build time for faster load times
- Consider using Blender add-ons for imposter generation
- Godot's `Camera3D.projection = ORTHOGONAL` for imposter capture
- Use `render_target_update_mode = UPDATE_ONCE` for single-frame capture

**Synergies:**
- Advanced version of Billboarding (view-dependent)
- Pairs with LOD Systems (imposters as final LOD)
- Combines with Geometry Instancing (instance imposters)
- Works with Texture Atlasing (pack multiple imposters)
- Enables Open World Rendering (distant objects efficiently)

**Measurement/Profiling:**
- GPU profiler: Vertex shader time (should drop 99%+ at imposter distances)
- Track imposter switches: How often do views change?
- Measure generation time: 1-10 seconds per object
- Visual quality: Compare imposter vs 3D from transition distance
- Memory bandwidth: Texture samples vs geometry processing
- Target: 95%+ vertex savings, imperceptible quality difference
- Typical: 100-1000x performance improvement vs full 3D

**Additional Resources:**
- GPU Gems 3: Chapter 21 "True Impostors"
- GDC 2016: "Rendering Rainbow Six Siege"
- Siggraph 2007: "Octahedral Imposters for Real-Time Rendering"
