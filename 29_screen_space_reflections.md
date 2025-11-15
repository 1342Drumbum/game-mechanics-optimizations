### 29. Screen-Space Reflections (SSR)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Real-time reflection quality vs performance. Planar reflections = render scene twice (doubles frame time: 16.67ms → 33.34ms). Cubemap reflections = bake offline but no dynamic objects. SSR uses existing frame buffer for reflections, adding only 2-5ms for high-quality reflections instead of 16ms+ for additional render passes.

**Technical Explanation:**
SSR traces rays in screen space using depth buffer. For each reflective pixel, calculate reflection vector, march along ray in screen space, test depth buffer for intersections. When ray hits surface, sample color from that screen position. Works only for visible geometry (screen-space limitation). Modern implementation: Hierarchical depth buffer for faster ray marching, temporal accumulation for noise reduction, hybrid approach (SSR + cubemaps for fallback). Good quality for flat surfaces (floors, water), handles dynamic objects automatically since using current frame.

**Algorithmic Complexity:**
- No reflections: O(0) - free but no reflections
- Planar reflections: O(n) - full scene render per plane
- SSR: O(p × s) where p=reflective pixels, s=ray march steps (8-32)
- Hierarchical SSR: O(p × log(d)) where d=max ray distance
- Cost: 2-5ms for quality SSR vs 16ms+ for real-time reflections

**Implementation Pattern:**
```gdscript
# Godot Screen-Space Reflections
# Note: Godot 4.x has built-in SSR, this demonstrates the concept

class_name SSREffect
extends Node

@export var ssr_enabled: bool = true
@export var max_steps: int = 32
@export var step_size: float = 0.1
@export var thickness: float = 0.5
@export var fade_in: float = 0.15
@export var fade_out: float = 0.5

var environment: Environment

func _ready():
    environment = get_viewport().world_3d.environment
    _setup_ssr()

func _setup_ssr():
    if not environment:
        environment = Environment.new()
        get_viewport().world_3d.environment = environment

    # Configure Godot's built-in SSR
    environment.ssr_enabled = ssr_enabled
    environment.ssr_max_steps = max_steps
    environment.ssr_fade_in = fade_in
    environment.ssr_fade_out = fade_out
    environment.ssr_depth_tolerance = thickness

# Custom SSR shader (conceptual)
const SSR_SHADER = """
shader_type spatial;
render_mode unshaded;

uniform sampler2D screen_texture : hint_screen_texture;
uniform sampler2D depth_texture : hint_depth_texture;
uniform sampler2D normal_texture;

uniform int max_steps = 32;
uniform float step_size = 0.1;
uniform float thickness = 0.5;
uniform float max_distance = 100.0;

vec3 world_pos_from_depth(vec2 screen_uv, float depth, mat4 inv_proj) {
    vec4 ndc = vec4(screen_uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 view_pos = inv_proj * ndc;
    return view_pos.xyz / view_pos.w;
}

vec4 trace_ray(vec3 ray_origin, vec3 ray_dir, mat4 proj, mat4 inv_proj) {
    vec3 current_pos = ray_origin;
    vec3 step = ray_dir * step_size;

    for (int i = 0; i < max_steps; i++) {
        current_pos += step;

        // Project to screen space
        vec4 proj_pos = proj * vec4(current_pos, 1.0);
        vec3 ndc = proj_pos.xyz / proj_pos.w;

        // Out of screen bounds
        if (ndc.x < -1.0 || ndc.x > 1.0 || ndc.y < -1.0 || ndc.y > 1.0) {
            return vec4(0.0);
        }

        vec2 screen_uv = ndc.xy * 0.5 + 0.5;

        // Sample depth buffer
        float buffer_depth = texture(depth_texture, screen_uv).r;
        vec3 buffer_pos = world_pos_from_depth(screen_uv, buffer_depth, inv_proj);

        // Check intersection
        float depth_diff = current_pos.z - buffer_pos.z;

        if (depth_diff > 0.0 && depth_diff < thickness) {
            // Hit! Sample color
            vec3 color = texture(screen_texture, screen_uv).rgb;
            float fade = 1.0 - float(i) / float(max_steps);
            return vec4(color, fade);
        }
    }

    return vec4(0.0);  // Miss
}

void fragment() {
    vec2 screen_uv = SCREEN_UV;

    // Get surface normal and position
    vec3 normal = texture(normal_texture, screen_uv).rgb * 2.0 - 1.0;
    float depth = texture(depth_texture, screen_uv).r;

    mat4 inv_proj = inverse(PROJECTION_MATRIX);
    vec3 view_pos = world_pos_from_depth(screen_uv, depth, inv_proj);

    // Calculate reflection vector
    vec3 view_dir = normalize(view_pos);
    vec3 reflect_dir = reflect(view_dir, normal);

    // Trace reflection ray
    vec4 reflection = trace_ray(view_pos, reflect_dir, PROJECTION_MATRIX, inv_proj);

    // Blend with scene
    vec3 scene_color = texture(screen_texture, screen_uv).rgb;
    ALBEDO = mix(scene_color, reflection.rgb, reflection.a * 0.5);
}
"""

# Hierarchical depth buffer for faster ray marching
class HierarchicalSSR:
    var depth_mips: Array[Texture2D] = []
    var mip_count: int = 6

    func generate_depth_mips(depth_texture: Texture2D):
        depth_mips.clear()
        depth_mips.append(depth_texture)

        # Generate mipmap chain (each level 2x2 max)
        for i in range(1, mip_count):
            var prev_mip = depth_mips[i - 1]
            var next_mip = _downsample_depth(prev_mip)
            depth_mips.append(next_mip)

    func _downsample_depth(source: Texture2D) -> Texture2D:
        # Sample 2x2 block, take maximum depth (conservative)
        # This would use compute shader or offline processing
        return source  # Placeholder

    func hierarchical_trace(origin: Vector3, direction: Vector3) -> Vector4:
        # Start with coarse mip, refine to detailed
        # Much faster than linear march
        return Vector4.ZERO  # Simplified

# Temporal SSR (accumulate over frames)
class TemporalSSR:
    @export var blend_factor: float = 0.1  # 10% current, 90% history
    var history_buffer: Texture2D
    var frame_count: int = 0

    func accumulate_frame(current_ssr: Texture2D) -> Texture2D:
        if frame_count == 0 or not history_buffer:
            history_buffer = current_ssr
            frame_count += 1
            return current_ssr

        # Blend current with history
        var blended = _blend_textures(current_ssr, history_buffer, blend_factor)
        history_buffer = blended
        frame_count += 1

        return blended

    func _blend_textures(current: Texture2D, history: Texture2D, factor: float) -> Texture2D:
        # Would use shader to blend
        # result = current * factor + history * (1 - factor)
        return current  # Placeholder
```

**Key Parameters:**
- Max ray steps: 16-64 (32 typical, quality vs performance)
- Step size: 0.05-0.2 (smaller = better quality, slower)
- Thickness threshold: 0.1-1.0 (intersection tolerance)
- Max distance: 50-200m (longer rays = more steps)
- Fade distances: Fade in 0.1-0.3, fade out 0.5-0.8
- Roughness cutoff: Only apply to smooth surfaces (roughness <0.3)

**Edge Cases:**
- Screen-space limitation: Missing off-screen geometry (use cubemap fallback)
- Grazing angles: Poor quality at shallow angles
- Thin objects: May miss due to thickness threshold
- Noise: Use temporal accumulation or higher step count
- Performance spikes: Limit steps per pixel or use adaptive quality

**When NOT to Use:**
- Heavily rough/matte surfaces (no visible reflections)
- VR (performance cost too high for 90fps)
- Mobile low-end (too expensive)
- Scenes with minimal reflective surfaces
- When cubemaps/planar sufficient

**Examples from Shipped Games:**

1. **Battlefield 1/V**: SSR for water, ice, metallic surfaces. 3ms cost at 1080p, 32 steps. Combined with cubemap fallback. Maintained 60fps on console with high-quality reflections.

2. **Control**: Extensive SSR for metallic Oldest House interior. 4ms cost, hierarchical depth buffer. 30fps on PS4, 60fps on PS5. Critical for art direction (brutalist reflective architecture).

3. **Metro Exodus**: SSR + RT reflections hybrid. SSR: 2.5ms (standard), RT: 8ms (RTX ON). Dynamic quality based on GPU. Non-RTX GPUs use SSR exclusively.

4. **Spider-Man (PS4/PS5)**: SSR for puddles, windows. 2ms on PS4, adaptive quality. PS5: Higher resolution SSR (4K) in 60fps mode. Enabled wet NYC streets aesthetic.

5. **Call of Duty: Modern Warfare**: High-quality SSR, 64 steps for hero surfaces. 5ms budget, scales to 2ms on lower settings. Glossy floor reflections in realistic lighting.

**Platform Considerations:**
- **PC High-End**: 64 steps, hierarchical tracing, 4K resolution, 3-5ms
- **PC Low-End**: 16 steps, simple tracing, 1080p, 1-2ms or disabled
- **Console (PS5/XSX)**: 32-48 steps, 4K/60fps, 2-4ms
- **Last-Gen Console**: 16-32 steps, dynamic resolution, 2-3ms
- **Mobile**: Generally disabled (too expensive) or 8-16 steps
- **VR**: Disabled (latency/performance concerns)

**Godot-Specific Notes:**
- Godot 4.x: Built-in SSR in `Environment` resource
- `ssr_enabled` = toggle on/off
- `ssr_max_steps` = quality (16-64)
- `ssr_fade_in/out` = control reflection strength by distance
- `ssr_depth_tolerance` = thickness for ray hits
- Works automatically with Forward+ renderer
- Performance: 1-3ms typical for 1080p, scales with resolution

**Synergies:**
- Requires **Z-Prepass** (depth buffer for ray marching)
- Pairs with **Deferred Rendering** (G-Buffer provides normals/roughness)
- Combines with **Temporal Anti-Aliasing** (reduces SSR noise)
- Works with **Cubemap Reflections** (fallback for off-screen)
- Enhances **PBR Materials** (realistic reflections for metallics)

**Measurement/Profiling:**
- GPU profiler: SSR pass time (should be 1-5ms)
- Track ray steps: Average steps taken per pixel
- Measure hit rate: How often rays find intersections?
- Visual quality: Compare with/without SSR
- Resolution impact: 1080p vs 4K (4x pixels = 4x cost)
- Target: <3ms at 1080p, <60% of reflective pixels traced
- Adaptive quality: Reduce steps when GPU-bound

**Additional Resources:**
- GPU Pro 5: "Screen-Space Reflections"
- GDC 2015: "Stochastic Screen-Space Reflections"
- Digital Foundry: SSR analysis in AAA games
