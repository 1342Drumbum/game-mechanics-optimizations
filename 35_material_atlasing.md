### 35. Material Atlasing

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Material switching overhead and broken batching. 1000 objects with 100 unique materials = 100 material binds × 0.1ms = 10ms CPU overhead. GPU state changes break batching: 1000 draw calls instead of 10. Material atlasing combines materials (textures + parameters) into unified system: 100 materials → 1 atlas = 10ms → 0.5ms, enabling aggressive batching (1000 → 10-50 draw calls).

**Technical Explanation:**
Combines textures from multiple materials into texture atlases, stores material parameters in lookup textures/buffers. Objects reference atlas coordinates instead of individual materials. GPU samples atlas using object's material ID. Enables batching objects with "different" materials since they share same atlas. Trade-off: More complex shaders (indirect lookups), larger textures, but massive draw call reduction. Best for scenes with many similar materials (environment props, foliage).

**Implementation:**
```gdscript
# Godot Material Atlasing System
class_name MaterialAtlas
extends Resource

var albedo_atlas: Texture2D
var normal_atlas: Texture2D
var material_params: PackedFloat32Array  # Per-material params
var atlas_size: int = 4096
var material_count: int = 0

# UV regions per material
var material_uvs: Dictionary = {}  # MaterialID -> Rect2

func add_material(mat_id: int, textures: Dictionary):
    # Pack textures into atlases
    # Store UV offset in material_uvs
    pass

# Shader supporting material atlasing
const ATLAS_SHADER = """
shader_type spatial;

uniform sampler2D albedo_atlas : source_color;
uniform sampler2D normal_atlas : hint_normal;
uniform sampler2D material_params;  // Roughness, metallic per material

uniform int materials_per_row = 16;

varying flat int material_id;

void vertex() {
    // Pass material ID from instance custom data
    material_id = int(INSTANCE_CUSTOM.x);
}

void fragment() {
    // Calculate atlas UV
    int mat_x = material_id % materials_per_row;
    int mat_y = material_id / materials_per_row;

    vec2 atlas_uv_offset = vec2(float(mat_x), float(mat_y)) / float(materials_per_row);
    vec2 atlas_uv_scale = vec2(1.0) / float(materials_per_row);

    vec2 atlas_uv = atlas_uv_offset + UV * atlas_uv_scale;

    // Sample from atlas
    vec4 albedo = texture(albedo_atlas, atlas_uv);
    vec3 normal = texture(normal_atlas, atlas_uv).rgb;

    // Get material parameters
    vec2 param_uv = vec2(float(material_id) / float(materials_per_row * materials_per_row), 0.5);
    vec4 params = texture(material_params, param_uv);

    ALBEDO = albedo.rgb;
    NORMAL_MAP = normal;
    ROUGHNESS = params.r;
    METALLIC = params.g;
}
"""
```

**Key Parameters:**
- Atlas size: 2048-8192 per texture type
- Materials per atlas: 16-256
- Parameter storage: Texture (16-256 materials) or buffer (thousands)
- Padding: 2-4 pixels between materials
- Compression: BC7/ASTC for quality

**Examples:**
1. **Fortnite**: Material atlasing for building pieces, 1000 materials → 10 atlases, 90% draw call reduction
2. **Star Citizen**: Ship material atlasing, complex PBR in single atlas
3. **Assassin's Creed**: Environment atlasing, enabled dense cities
4. **Spider-Man**: Building materials atlased, NYC rendering optimized
5. **Horizon**: Terrain/foliage atlasing, open world performance

**When NOT to Use:**
- Few unique materials (<20)
- Need individual material editing at runtime
- Every object has unique material
- Transparent materials (sorting issues)

