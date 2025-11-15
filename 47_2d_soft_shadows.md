### 47. 2D Soft Shadows

**Category:** Performance - 2D Visual Effects & Rendering

**Problem It Solves:** Realistic soft shadows without expensive raytracing. Per-pixel shadow raymarching for 100 lights × 1920×1080 resolution = 207M rays × 32 samples each = 6.6B operations = 106 seconds at 16ns per op. Optimized soft shadows reduce to 10-30ms using shadow maps and blur, a 99.97% reduction.

**Technical Explanation:**
2D soft shadows simulate penumbra (gradual transition from light to dark) using shadow mapping or distance fields. Light casts rays to find occluders (LightOccluder2D nodes), generates shadow polygon, renders to shadow texture, applies Gaussian blur for softness. GPU parallelization makes blur fast. Alternative: signed distance fields pre-compute nearest occluder distance, enabling real-time soft shadow calculation in shader.

Modern approach uses multiple shadow samples: cast 3-9 rays from light with slight offset, blend results. More samples = softer shadows but higher cost. Optimize by reducing sample count for distant lights, using lower-resolution shadow maps for background lights, and caching static shadows. Blur kernel size controls softness: 3×3 (subtle), 7×7 (medium), 15×15 (very soft).

**Algorithmic Complexity:**
- Raytraced Soft Shadows: O(l × p × s) where l=lights, p=pixels, s=samples = billions of ops
- Shadow Map + Blur: O(l × o × p × k²) where o=occluders, k=kernel size = millions of ops
- Distance Field: O(p) with GPU parallelization = thousands of ops
- Optimization: 1000× faster than naive raytracing

**Implementation Pattern:**
```gdscript
# Optimized PointLight2D with soft shadows
class_name SoftShadowLight
extends PointLight2D

@export var shadow_softness: float = 4.0  # Blur amount
@export var shadow_sample_count: int = 5  # Quality vs performance
@export var enable_shadows: bool = true

func _ready():
    # Enable shadow casting
    shadow_enabled = enable_shadows

    # Configure shadow filtering (PCF = Percentage Closer Filtering)
    # Higher values = softer shadows but more expensive
    match shadow_sample_count:
        1, 2, 3:
            shadow_filter = Light2D.SHADOW_FILTER_NONE  # Hard shadows, fastest
        4, 5, 6:
            shadow_filter = Light2D.SHADOW_FILTER_PCF5  # 5 samples, soft
        _:
            shadow_filter = Light2D.SHADOW_FILTER_PCF13 # 13 samples, very soft

    # Shadow buffer size affects quality and performance
    # Lower = faster but blockier shadows
    shadow_buffer_size = 2048 if shadow_sample_count > 6 else 1024

# Light occluder optimization
class_name OptimizedLightOccluder
extends LightOccluder2D

@export var simplify_threshold: float = 2.0  # Pixels
@export var cull_distance: float = 1000.0    # Don't cast shadows beyond this

var original_polygon: PackedVector2Array
var simplified_polygon: PackedVector2Array

func _ready():
    if occluder_polygon:
        original_polygon = occluder_polygon.polygon
        simplified_polygon = simplify_polygon(original_polygon, simplify_threshold)
        occluder_polygon.polygon = simplified_polygon

func _process(_delta):
    # Cull occluder if too far from any light
    var nearest_light = find_nearest_light()
    if nearest_light:
        var dist = global_position.distance_to(nearest_light.global_position)
        visible = dist < cull_distance
    else:
        visible = false

func simplify_polygon(polygon: PackedVector2Array, threshold: float) -> PackedVector2Array:
    # Douglas-Peucker algorithm for polygon simplification
    if polygon.size() < 3:
        return polygon

    var simplified = PackedVector2Array()
    simplified.append(polygon[0])

    var stack = [[0, polygon.size() - 1]]
    var keep_indices = {0: true, polygon.size() - 1: true}

    while stack.size() > 0:
        var range_data = stack.pop_back()
        var start_idx = range_data[0]
        var end_idx = range_data[1]

        var max_dist = 0.0
        var max_idx = start_idx

        for i in range(start_idx + 1, end_idx):
            var dist = point_to_line_distance(
                polygon[i],
                polygon[start_idx],
                polygon[end_idx]
            )
            if dist > max_dist:
                max_dist = dist
                max_idx = i

        if max_dist > threshold:
            keep_indices[max_idx] = true
            stack.append([start_idx, max_idx])
            stack.append([max_idx, end_idx])

    for i in polygon.size():
        if keep_indices.has(i):
            simplified.append(polygon[i])

    return simplified

func point_to_line_distance(point: Vector2, line_start: Vector2, line_end: Vector2) -> float:
    var line_vec = line_end - line_start
    var point_vec = point - line_start
    var line_len = line_vec.length()

    if line_len < 0.0001:
        return point_vec.length()

    var t = clamp(point_vec.dot(line_vec) / (line_len * line_len), 0.0, 1.0)
    var projection = line_start + t * line_vec
    return point.distance_to(projection)

# Distance field shadow shader (advanced)
const DISTANCE_FIELD_SHADOW_SHADER = """
shader_type canvas_item;

uniform sampler2D distance_field;
uniform vec2 light_position;
uniform float light_radius = 500.0;
uniform float shadow_softness = 8.0;

void fragment() {
    vec2 to_light = light_position - FRAGCOORD.xy;
    float dist_to_light = length(to_light);

    if (dist_to_light > light_radius) {
        COLOR.a = 0.0;
        return;
    }

    // Sample distance field
    vec2 uv = FRAGCOORD.xy / vec2(textureSize(distance_field, 0));
    float sdf = texture(distance_field, uv).r;

    // Calculate shadow based on distance to nearest occluder
    float shadow = smoothstep(0.0, shadow_softness, sdf * dist_to_light);

    COLOR.rgb = vec3(0.0);
    COLOR.a = 1.0 - shadow;
}
"""

# Shadow caching for static geometry
class_name ShadowCache
extends Node2D

var cached_shadows: Dictionary = {}  # Light+Occluder hash -> shadow texture
var cache_dirty: bool = true

func get_or_create_shadow(light: Light2D, occluders: Array) -> Texture2D:
    var cache_key = hash_light_occluders(light, occluders)

    if cached_shadows.has(cache_key) and not cache_dirty:
        return cached_shadows[cache_key]

    # Render shadow to texture
    var shadow_tex = render_shadow_to_texture(light, occluders)
    cached_shadows[cache_key] = shadow_tex
    cache_dirty = false

    return shadow_tex

func hash_light_occluders(light: Light2D, occluders: Array) -> int:
    var hash_val = light.global_position.x + light.global_position.y * 1000
    for occluder in occluders:
        if occluder is LightOccluder2D:
            hash_val += occluder.global_position.x + occluder.global_position.y * 100
    return int(hash_val)

func invalidate_cache():
    cache_dirty = true

# Multi-sample soft shadows (quality setting)
class_name MultiSampleShadow
extends Node2D

enum Quality { LOW, MEDIUM, HIGH, ULTRA }

@export var shadow_quality: Quality = Quality.MEDIUM

var sample_counts = {
    Quality.LOW: 1,     # Hard shadows
    Quality.MEDIUM: 5,  # PCF5
    Quality.HIGH: 13,   # PCF13
    Quality.ULTRA: 25   # Custom multi-sampling
}

func update_light_quality(light: PointLight2D):
    var samples = sample_counts[shadow_quality]

    if samples <= 1:
        light.shadow_filter = Light2D.SHADOW_FILTER_NONE
    elif samples <= 5:
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF5
    else:
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF13

    # Adjust buffer size for quality
    match shadow_quality:
        Quality.LOW:
            light.shadow_buffer_size = 512
        Quality.MEDIUM:
            light.shadow_buffer_size = 1024
        Quality.HIGH:
            light.shadow_buffer_size = 2048
        Quality.ULTRA:
            light.shadow_buffer_size = 4096

# Dynamic shadow LOD based on distance
class_name ShadowLOD
extends Node

func update_shadow_lod(light: PointLight2D, camera_pos: Vector2):
    var dist = light.global_position.distance_to(camera_pos)

    if dist > 1500:
        # Very far - disable shadows
        light.shadow_enabled = false
    elif dist > 1000:
        # Far - hard shadows only
        light.shadow_filter = Light2D.SHADOW_FILTER_NONE
        light.shadow_buffer_size = 512
    elif dist > 500:
        # Medium - soft shadows low quality
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF5
        light.shadow_buffer_size = 1024
    else:
        # Close - full quality soft shadows
        light.shadow_filter = Light2D.SHADOW_FILTER_PCF13
        light.shadow_buffer_size = 2048
```

**Key Parameters:**
- **Shadow Filter:** NONE (hard), PCF5 (soft 5-sample), PCF13 (very soft 13-sample)
- **Shadow Buffer Size:** 512 (performance), 1024 (balanced), 2048 (quality), 4096 (hero lights)
- **Blur Kernel:** 3×3 (subtle), 5×5 (medium), 7×7 (soft), 15×15 (very soft)
- **Sample Count:** 1 (hard), 5 (soft), 13 (very soft), 25+ (ultra soft, expensive)
- **Occluder Complexity:** 4-12 vertices typical, >20 vertices needs simplification
- **Shadow Distance:** 500-2000 pixels max range before culling

**Edge Cases:**
- **Overlapping Occluders:** Multiple overlapping creates darker shadows, may need blending limit
- **Light Inside Occluder:** Shadow artifacts, clamp light position or disable shadows
- **Extremely Soft Shadows:** Large blur kernels (20+) cause performance spikes
- **Moving Occluders:** Invalidates cached shadows, needs re-render every frame
- **Shadow Acne:** Self-shadowing artifacts, adjust shadow bias parameter
- **Performance Cliff:** >8 soft shadow lights causes exponential performance drop

**When NOT to Use:**
- Top-down games with overhead lighting - soft shadows rarely visible
- Pixel art with hard shadow aesthetic - soft shadows look wrong
- Mobile low-end devices - fragment shader cost too high
- Fast-paced action - players won't notice shadow softness
- Already struggling performance - hard shadows 3-5× faster

**Examples from Shipped Games:**

1. **Dead Cells:** Torch lights use PCF5 soft shadows on walls/platforms. 15-20 lights typical, shadow buffer 1024×1024. Distant lights use hard shadows (LOD system). Critical atmospheric feature in dark biomes. Optimized by culling shadows outside viewport.

2. **Hollow Knight:** Boss arenas feature dramatic soft shadow lighting. The White Palace uses 30+ candles with soft shadows. Uses distance-based LOD: close lights PCF13 (2048 buffer), far lights PCF5 (512 buffer). Static geometry shadows cached, only dynamic shadows update.

3. **Ori and the Blind Forest:** Extensive soft shadow usage for depth. Spirit light creates soft radial shadows. 40-60 lights on screen, aggressive LOD system. Uses 5-sample soft shadows for most lights, 13-sample for Ori's glow. Occluder simplification keeps vertex count <8 per polygon.

4. **Don't Starve:** Campfire/torch shadows use custom soft shadow shader. Simplified shadow calculations for performance on mobile. Uses 3-sample shadow for soft effect with minimal cost. Shadow distance limited to 300 pixels from light source.

5. **Mark of the Ninja:** Stealth gameplay emphasizes lighting/shadows. Soft shadows communicate visibility clearly. Uses PCF5 for all shadows (consistency). Shadow buffer optimized at 1024×1024 for all platforms. Occluders pre-simplified at export time.

**Platform Considerations:**
- **Mobile (iOS/Android):** Use PCF5 maximum, 512-1024 shadow buffers. Limit to 8-12 shadow-casting lights. Disable shadows for lights beyond 800 pixels. Target <5ms for all shadow rendering.
- **Desktop (Windows/Mac/Linux):** PCF13 comfortable for 30+ lights. 2048 shadow buffers standard. Can afford higher quality on modern GPUs. Target <3ms shadow rendering.
- **Console (Switch/PS/Xbox):** Switch: PCF5, 1024 buffers, <20 lights. PS5/Xbox: PCF13+, 2048-4096 buffers, 60+ lights fine.
- **WebGL:** PCF5 maximum, limit to 10-15 lights total. Shadow compilation slower, pre-warm shaders. 512-1024 buffers only.
- **VRAM:** Shadow buffers add up: 10 lights × 2048² × 4 bytes = 160MB. Use smaller buffers or fewer lights.

**Godot-Specific Notes:**
- Godot 4.x shadow quality significantly improved over 3.x (better PCF implementation)
- `Light2D.shadow_filter` controls softness: NONE, PCF5, PCF13
- `Light2D.shadow_buffer_size` sets shadow map resolution (power of 2)
- Use `LightOccluder2D` with simplified polygons for performance
- Monitor shadow cost: Profiler → Visual → Shadow Rendering
- Godot automatically culls shadows outside viewport
- `shadow_item_cull_mask` limits which items cast shadows (major optimization)

**Synergies:**
- **2D Lighting with Normal Maps:** Soft shadows enhance normal-mapped lighting
- **Canvas Layer Management:** Separate shadow layers for better control
- **Sprite Batching:** Shadow occluders don't break batching
- **Dirty Rectangles:** Cache static shadows, update only dynamic
- **Object Pooling:** Pool LightOccluder2D nodes for dynamic objects

**Measurement/Profiling:**
- **Shadow Render Time:** Monitor `Performance.TIME_RENDER` with shadows on/off
- **Light Count:** Track shadow-casting lights - target <20 on mobile, <40 desktop
- **Buffer Memory:** Each 2048² buffer = 16MB, monitor `RENDER_TEXTURE_MEM_USED`
- **Frame Time Impact:** Shadows typically add 2-8ms depending on quality/count
- **Profiling Pattern:**
```gdscript
var lights_with_shadows = 0
var total_shadow_buffer_memory = 0

for light in get_tree().get_nodes_in_group("lights"):
    if light is PointLight2D and light.shadow_enabled:
        lights_with_shadows += 1
        var buffer_size = light.shadow_buffer_size
        total_shadow_buffer_memory += buffer_size * buffer_size * 4  # RGBA

print("Shadow lights: ", lights_with_shadows)
print("Shadow buffer memory: ", total_shadow_buffer_memory / 1024.0 / 1024.0, " MB")

var time_before = Performance.get_monitor(Performance.TIME_RENDER)
await get_tree().process_frame
var time_after = Performance.get_monitor(Performance.TIME_RENDER)
print("Shadow render time: ", (time_after - time_before) * 1000, " ms")
```

**Advanced Optimization Patterns:**
```gdscript
# Pattern 1: Shadow pooling (reuse buffers)
var shadow_buffer_pool: Array[Image] = []

func get_shadow_buffer(size: int) -> Image:
    for buffer in shadow_buffer_pool:
        if buffer.get_width() == size and not buffer.is_in_use():
            return buffer
    # Create new if none available
    var new_buffer = Image.create(size, size, false, Image.FORMAT_RGBA8)
    shadow_buffer_pool.append(new_buffer)
    return new_buffer

# Pattern 2: Temporal shadow smoothing (reduce flicker)
var previous_shadow_texture: Texture2D
var shadow_blend_factor: float = 0.2

func blend_with_previous_shadow(current: Texture2D) -> Texture2D:
    if not previous_shadow_texture:
        previous_shadow_texture = current
        return current

    # Blend current frame shadow with previous for smoothness
    var blended = blend_textures(previous_shadow_texture, current, shadow_blend_factor)
    previous_shadow_texture = blended
    return blended

# Pattern 3: Adaptive shadow quality
var target_frametime: float = 16.67  # 60 FPS
var current_shadow_quality: int = 5  # PCF5

func adapt_shadow_quality():
    var frame_time = Performance.get_monitor(Performance.TIME_PROCESS) * 1000

    if frame_time > target_frametime * 1.1:  # 10% over budget
        current_shadow_quality = max(1, current_shadow_quality - 1)
    elif frame_time < target_frametime * 0.9:  # 10% under budget
        current_shadow_quality = min(13, current_shadow_quality + 1)

    update_all_light_quality(current_shadow_quality)
```
