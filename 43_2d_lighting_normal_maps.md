### 43. 2D Lighting with Normal Maps

**Category:** Performance - 2D Visual Effects & Rendering

**Problem It Solves:** Realistic 2D lighting without expensive per-pixel raytracing. Naive per-pixel lighting with distance falloff for 100 lights × 1920×1080 pixels = 207M calculations per frame = 3320ms at 16ns per calculation. Normal map lighting reduces to ~10ms with hardware acceleration, a 99.7% reduction.

**Technical Explanation:**
Normal maps encode surface orientation in RGB texture channels (x, y, z normals mapped to 0-255). GPU uses normals to calculate per-pixel lighting in fragment shader. For each pixel, compute dot product of light direction and surface normal to determine brightness. Hardware texture sampling and parallel fragment processing make this extremely fast. Instead of raytracing geometry, fake 3D depth using 2D height information.

The technique uses tangent-space normal mapping: blue channel (Z) represents "up" from surface, red (X) and green (Y) encode horizontal variations. Light shader samples normal map, transforms to world space, computes N·L (normal dot light direction), applies color and attenuation. Result is convincing 3D-style lighting on 2D sprites with minimal performance cost.

**Algorithmic Complexity:**
- Per-Pixel Raytracing: O(n × p × l) where n=objects, p=pixels, l=lights = millions of ops
- Normal Map Lighting: O(p × l) with GPU parallelization = thousands of ops
- GPU Advantage: 2000+ parallel fragment processors vs 8 CPU cores
- Typical: 100x faster than CPU-based lighting simulation

**Implementation Pattern:**
```gdscript
# Light2D setup with normal maps
class_name NormalMappedSprite
extends Sprite2D

@export var normal_map: Texture2D
@export var height_scale: float = 1.0

func _ready():
    # Assign normal map to material
    material = CanvasItemMaterial.new()
    texture_normal = normal_map

    # Configure for lighting
    material.light_mode = CanvasItemMaterial.LIGHT_MODE_NORMAL

# Optimized light manager
class_name Light2DManager
extends Node2D

const MAX_LIGHTS = 64  # Hardware limit varies
var active_lights: Array[PointLight2D] = []
var light_pool: Array[PointLight2D] = []

func _ready():
    # Pre-allocate light pool
    for i in range(MAX_LIGHTS):
        var light = PointLight2D.new()
        light.enabled = false
        light.blend_mode = Light2D.BLEND_MODE_ADD
        light_pool.append(light)
        add_child(light)

func get_light() -> PointLight2D:
    for light in light_pool:
        if not light.enabled:
            light.enabled = true
            active_lights.append(light)
            return light
    # All lights in use, disable oldest
    var oldest = active_lights.pop_front()
    active_lights.append(oldest)
    return oldest

func return_light(light: PointLight2D):
    light.enabled = false
    active_lights.erase(light)

func _process(_delta):
    # Cull lights outside viewport
    var viewport_rect = get_viewport_rect()
    for light in active_lights:
        var light_radius = light.texture_scale * 256  # Assuming 512px texture
        var bounds = Rect2(light.global_position - Vector2(light_radius, light_radius),
                           Vector2(light_radius * 2, light_radius * 2))
        light.visible = viewport_rect.intersects(bounds)

# Custom shader for advanced normal map effects
const NORMAL_LIGHTING_SHADER = """
shader_type canvas_item;

uniform sampler2D normal_map;
uniform float height_scale = 1.0;
uniform float specular_strength = 0.5;
uniform float specular_shininess = 32.0;

void light() {
    // Sample normal map
    vec3 normal = texture(normal_map, UV).rgb;
    normal = normal * 2.0 - 1.0;  // Remap from [0,1] to [-1,1]
    normal.xy *= height_scale;
    normal = normalize(normal);

    // Calculate diffuse lighting
    vec3 light_dir = normalize(vec3(LIGHT_POSITION - FRAGCOORD.xy, LIGHT_HEIGHT));
    float diffuse = max(dot(normal, light_dir), 0.0);

    // Calculate specular
    vec3 view_dir = normalize(vec3(0.0, 0.0, 1.0));
    vec3 reflect_dir = reflect(-light_dir, normal);
    float spec = pow(max(dot(view_dir, reflect_dir), 0.0), specular_shininess);

    // Combine lighting
    LIGHT = LIGHT_COLOR.rgb * (diffuse + spec * specular_strength) * LIGHT_ENERGY;
}
"""

class_name CustomNormalSprite
extends Sprite2D

func _ready():
    var shader_material = ShaderMaterial.new()
    shader_material.shader = load("res://shaders/normal_lighting.gdshader")
    shader_material.set_shader_parameter("normal_map", texture_normal)
    shader_material.set_shader_parameter("height_scale", 1.5)
    material = shader_material
```

**Key Parameters:**
- **Light Count:** 4-16 simultaneous lights (mobile), 32-64 (desktop)
- **Light Texture Size:** 256x256 (performance), 512x512 (quality), 1024x1024 (hero lights)
- **Normal Map Resolution:** Match sprite resolution (512x512 typical for characters)
- **Height Scale:** 0.5-2.0 (controls depth illusion strength)
- **Light Energy:** 0.5-2.0 typical range
- **Shadow Quality:** Light2D.shadow_filter = 0 (none), 1 (PCF5), 2 (PCF13)

**Edge Cases:**
- **Light Overlap:** Too many lights on one pixel causes overdraw, limit to 8-16 per pixel
- **Normal Map Artifacts:** Ensure proper tangent-space normals, check blue channel ~128
- **Tile Seams:** Normal maps on tiles can show seams, use padding or seamless generation
- **Dynamic Lights:** Moving lights every frame forces re-render, limit to 8-16 dynamic
- **Alpha Blending:** Transparent sprites with normals need careful ordering
- **Light Bleeding:** Lights affect sprites they shouldn't - use light masks/layers

**When NOT to Use:**
- Pixel art with hard shadows - normal maps look wrong at low resolution
- Pure ambient lighting - no directional lights, normal maps provide no benefit
- Top-down games where lighting is flat - overhead lights don't benefit from normals
- Performance critical mobile - lighting doubles fragment shader cost
- Retro aesthetic requiring flat shading

**Examples from Shipped Games:**

1. **Dead Cells:** Extensive normal map lighting on characters and environments. Torches use PointLight2D with normal-mapped stone walls creating realistic depth. ~20-30 lights typical, optimized by culling off-screen lights. Critical for atmospheric dungeon feel.

2. **Hollow Knight:** Boss arenas use normal-mapped lighting for dramatic effect. The Radiance fight has 40+ light rays, each with normal interaction. Optimized by limiting light range and using light masks. Character sprites have subtle normal maps for rim lighting.

3. **Moonlighter:** Shop interior uses normal-mapped lighting extensively. Dynamic day/night cycle with 1 sun light + 10-15 interior lights. Items on shelves have normal maps creating realistic 3D appearance. Optimized via chunk-based light culling.

4. **Ori and the Blind Forest:** Heavy normal map usage for forest environments. Magical abilities emit light interacting with normal-mapped trees/rocks. 30-50 lights on screen during intense moments, culled aggressively outside viewport. Specular highlights on water using normal maps.

5. **Shovel Knight (newer ports):** Added optional normal map lighting for enhanced editions. Maintains pixel art aesthetic while adding subtle depth. Uses conservative light count (8-12) and simplified normals to preserve retro feel.

**Platform Considerations:**
- **Mobile (iOS/Android):** Limit to 8-16 lights, use 256x256 light textures, disable shadows or use simple shadow_filter=1. Fragment shader cost high on mobile GPUs. Target <3ms for lighting.
- **Desktop (Windows/Mac/Linux):** Can handle 32-64 lights, 512x512 textures, PCF13 shadows. Modern GPUs handle normal mapping easily. Monitor fillrate on 4K displays.
- **Console (Switch/PS/Xbox):** Switch: 16-24 lights max, 256-512px textures. PS5/Xbox: 64+ lights fine. Use hardware-specific optimizations.
- **WebGL:** Limited to 16-24 lights, simple shadows only. Fragment shader compilation slower. Use shader caching.
- **VRAM Usage:** 512x512 RGBA normal map = 1MB uncompressed. Use texture compression (DXT5/BC5 for normals).

**Godot-Specific Notes:**
- Godot 4.x uses Vulkan/OpenGL 3.3+ for lighting, much faster than Godot 3.x
- Set `CanvasItemMaterial.light_mode = LIGHT_MODE_NORMAL` to enable normal map usage
- Use `Sprite2D.texture_normal` for automatic normal map assignment
- Light2D has built-in shadow support with `shadow_enabled = true`
- Monitor lighting cost: Profiler → Visual → CanvasItem Draw Calls
- Use `Light2D.range_item_cull_mask` to limit which items light affects (major optimization)
- Godot supports light occluders via `LightOccluder2D` for shadows

**Synergies:**
- **Sprite Batching:** Normal maps don't break batching if material identical
- **Canvas Layer Management:** Separate lit and unlit layers for performance
- **Dirty Rectangles:** Only recalculate lighting for changed areas
- **2D Soft Shadows:** Normal maps enhance shadow realism
- **Particle Systems:** Normal-mapped particles for magical effects

**Measurement/Profiling:**
- **Frame Time:** Monitor `Performance.TIME_RENDER` - lighting should be <30% of frame
- **Light Count:** Track active lights - target <20 on mobile, <50 on desktop
- **Overdraw:** Use Godot's overdraw debug mode - lighting increases overdraw 2-5x
- **Shader Compilation:** First-time shader load can spike 50-200ms, pre-warm shaders
- **Profiling Pattern:**
```gdscript
var lights_before = $LightManager.active_lights.size()
var time_before = Performance.get_monitor(Performance.TIME_RENDER)
# ... gameplay frame ...
var time_after = Performance.get_monitor(Performance.TIME_RENDER)
var lighting_time = time_after - time_before
print("Lights: ", lights_before, " | Render: ", lighting_time, "ms")
```

**Common Optimization Patterns:**
```gdscript
# Pattern 1: Light LOD based on distance to camera
func update_light_quality(light: PointLight2D, camera_pos: Vector2):
    var dist = light.global_position.distance_to(camera_pos)
    if dist > 1000:
        light.shadow_enabled = false
        light.texture_scale = 0.5
    elif dist > 500:
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF5
    else:
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF13

# Pattern 2: Light pooling with priority
func allocate_light_with_priority(priority: int) -> PointLight2D:
    if active_lights.size() < MAX_LIGHTS:
        return get_light()
    # Find lowest priority active light
    var lowest_priority_light = null
    var lowest_priority = priority
    for light in active_lights:
        if light.get_meta("priority") < lowest_priority:
            lowest_priority = light.get_meta("priority")
            lowest_priority_light = light
    if lowest_priority_light:
        return_light(lowest_priority_light)
        return get_light()
    return null

# Pattern 3: Viewport-based culling
func cull_lights_outside_viewport():
    var vp = get_viewport_rect()
    for light in active_lights:
        light.visible = vp.has_point(light.global_position)
```
