### 55. Normal Map Baking

**Category:** Performance - 3D Rendering / Asset Optimization

**Problem It Solves:** High polygon count causing excessive GPU vertex processing. A detailed character with 500,000 triangles requires 1.5M vertices processed per frame at 60fps = 90M vertex transformations/sec. At 50 GPU cycles per vertex = 4.5B cycles = 2.25ms on 2GHz GPU shader cores. Reducing to 5,000 triangles (100x reduction) with normal maps maintains visual quality while dropping vertex processing to 0.02ms.

**Technical Explanation:**
Normal map baking transfers geometric detail from high-polygon mesh to texture on low-polygon mesh. High-poly mesh (500K triangles) is used to generate a normal map (2048×2048 texture) that stores surface normal directions as RGB values. Low-poly mesh (5K triangles) uses this normal map in fragment shader to calculate per-pixel lighting, simulating high-poly detail. Each texel stores XYZ normal components encoded as RGB (0-255). At runtime, fragment shader reads normal map and perturbs surface normal before lighting calculations, creating illusion of detailed geometry at fraction of vertex cost.

**Algorithmic Complexity:**
- High-poly rendering: O(vertices) vertex processing + O(fragments) fragment processing
- Normal-mapped low-poly: O(vertices_low) vertex + O(fragments) fragment + O(1) texture lookup
- Vertex reduction: 100x fewer vertices (500K → 5K)
- Fragment cost: +1 texture lookup (~4 cycles) vs thousands fewer vertex transforms
- Net gain: 50-100x performance improvement in vertex-bound scenarios

**Implementation Pattern:**
```gdscript
# Godot implementation
extends MeshInstance3D

@export var normal_map: Texture2D
@export var use_tangent_space: bool = true

func _ready():
    setup_normal_mapping()

func setup_normal_mapping():
    # Godot handles normal mapping automatically via StandardMaterial3D
    var material = StandardMaterial3D.new()
    material.normal_enabled = true
    material.normal_texture = normal_map
    material.normal_scale = 1.0  # Adjust intensity (0.0-2.0)

    set_surface_override_material(0, material)

# Custom shader for advanced normal mapping
const NORMAL_MAPPING_SHADER = """
shader_type spatial;

uniform sampler2D normal_map : hint_normal;
uniform float normal_strength : hint_range(0.0, 2.0) = 1.0;
uniform sampler2D albedo_texture : hint_albedo;

void fragment() {
    // Sample normal map (stored in tangent space)
    vec3 normal_tex = texture(normal_map, UV).rgb;

    // Decode from [0,1] to [-1,1]
    normal_tex = normal_tex * 2.0 - 1.0;

    // Apply strength
    normal_tex.xy *= normal_strength;
    normal_tex = normalize(normal_tex);

    // Transform from tangent space to world space
    // TANGENT and BINORMAL are automatically provided by Godot
    vec3 world_normal = TANGENT * normal_tex.x +
                        BINORMAL * normal_tex.y +
                        NORMAL * normal_tex.z;

    NORMAL = normalize(world_normal);

    ALBEDO = texture(albedo_texture, UV).rgb;
}
"""

# Tool for baking normal maps in Godot editor
@tool
class_name NormalMapBaker

var high_poly_mesh: Mesh
var low_poly_mesh: Mesh
var output_resolution: int = 2048

func bake_normal_map() -> Image:
    # Godot doesn't have built-in baking, use external tools or viewport rendering
    # This is a conceptual example

    var viewport = SubViewport.new()
    viewport.size = Vector2i(output_resolution, output_resolution)
    viewport.render_target_update_mode = SubViewport.UPDATE_ALWAYS

    # Render high-poly mesh from multiple angles
    # Compare with low-poly mesh surface to generate normal deltas
    # Store deltas as RGB in texture

    var image = viewport.get_texture().get_image()
    return image

# Runtime normal map application
class_name NormalMappedModel

@export var normal_intensity: float = 1.0
@export var parallax_occlusion: bool = false

func apply_normal_map(mesh_instance: MeshInstance3D, normal_tex: Texture2D):
    var mat = mesh_instance.get_active_material(0) as StandardMaterial3D
    if mat:
        mat.normal_enabled = true
        mat.normal_texture = normal_tex
        mat.normal_scale = normal_intensity

        # Optional: Add parallax occlusion mapping for depth effect
        if parallax_occlusion:
            mat.heightmap_enabled = true
            mat.heightmap_scale = 0.05
            mat.heightmap_deep_parallax = true
```

**Key Parameters:**
- Normal map resolution: 1024×1024 to 4096×4096 (2048² is sweet spot)
- Normal strength/intensity: 0.5 - 2.0 (1.0 = full strength)
- Polygon reduction ratio: 50:1 to 200:1 (target: 100:1)
- Texture format: BC5/RGTC (2 channel) or BC7/ASTC (3 channel) compressed
- Mipmap levels: Full chain for proper LOD
- Memory per 2K normal map: 16MB uncompressed, 1-4MB compressed

**Edge Cases:**
- Silhouette detail: Normal maps can't add geometry at edges
- Extreme angles: Normal maps lose effectiveness at grazing angles
- UV seams: Ensure continuous normals across seams (use padding)
- Low-poly base too simple: Need sufficient geometry for smooth interpolation
- Compression artifacts: BC5 compression can introduce banding, use BC7 for quality

**When NOT to Use:**
- Silhouette-critical details (use actual geometry)
- Flat surfaces with no detail (wastes texture memory)
- Dynamically deforming meshes (normals won't match deformation)
- Very distant objects (detail not visible, use simpler materials)
- Mobile low-end devices (texture bandwidth cost may exceed vertex cost)

**Examples from Shipped Games:**

1. **Doom 2016:** Extensive normal mapping on all environment and demon models. Reduced demon triangle count from 300K to 15K per model (20x reduction) while maintaining AAA visual fidelity. Saved 15-20ms of vertex processing per frame.

2. **The Last of Us Part II:** High-resolution normal maps (4096²) on character faces captured photogrammetry detail. Reduced face geometry from 200K to 20K triangles (10x) while maintaining film-quality close-ups.

3. **Horizon Zero Dawn:** Machine creatures use 2048² normal maps with 100:1 polygon reduction. Background rocks reduced from 50K to 500 triangles with normal maps, enabling vast open world without vertex bottleneck.

4. **Spider-Man (PS4):** Buildings use aggressive normal mapping (200:1 reduction) from photogrammetry scans. Reduced New York City geometry from 2B to 10M triangles while maintaining visual detail, critical for streaming performance.

5. **The Witcher 3:** Character armor and monsters use 2048² normal maps. Geralt's detailed armor reduced from 80K to 15K triangles. Enabled complex character models to run at 30fps on consoles.

**Platform Considerations:**
- **PC/Console:** Full normal map support, 4096² textures feasible
- **Mobile:** Use 1024² or lower, BC5/ASTC compression mandatory
- **Switch:** 2048² max for main characters, 1024² for NPCs
- **VR:** Normal maps critical for performance, but watch texture bandwidth (2-4GB/s limit)
- **Texture memory:** 2048² BC5 normal map = 2.67MB, budget accordingly
- **Bandwidth:** Normal maps add 1-2 texture lookups per fragment (8-16 bytes/fragment)

**Godot-Specific Notes:**
- Godot automatically applies normal maps via `StandardMaterial3D.normal_texture`
- Use `normal_enabled = true` to activate normal mapping
- `normal_scale` parameter controls intensity (0.0 = disabled, 1.0 = full, 2.0 = exaggerated)
- Godot expects OpenGL-style normal maps (Y+ = up). DirectX normal maps need Y-flip
- Compress normal maps using BC5/RGTC format via import settings
- For baking, use external tools: Substance Painter, Blender, xNormal
- Godot 4.x import pipeline: Set "Normal Map" in texture import mode
- Use `hint_normal` in custom shaders to ensure proper sRGB handling

**Synergies:**
- **Mesh Simplification (Technique 56):** Generate low-poly mesh for normal map baking
- **Texture Atlasing:** Combine multiple normal maps into atlases
- **Parallax Occlusion Mapping:** Add height map for depth effect with normal maps
- **PBR Materials:** Normal maps essential for physically-based rendering quality
- **LOD Systems:** Reduce normal map resolution at lower LOD levels

**Measurement/Profiling:**
- **Vertex processing time:** Measure before/after in GPU profiler (RenderDoc, NSight)
- **Triangle count reduction:** Track poly count (500K → 5K)
- **Memory usage:** Monitor texture memory (RenderingServer.get_rendering_info())
- **Visual quality:** Side-by-side comparison at various distances and angles
- **Fragment cost:** Each normal map lookup adds ~4-8 cycles per fragment
- **Target:** 50-100x triangle reduction, <5% visual quality loss
- **Profile:** Use Godot's "Visible Information" overlay to see triangle counts

---
