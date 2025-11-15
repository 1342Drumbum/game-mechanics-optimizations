### 24. Cascaded Shadow Maps (CSM)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Shadow quality vs performance trade-off for directional lights (sun/moon). Single 4096×4096 shadow map for 1000m scene = 0.24 pixels/meter (terrible quality). Four 2048×2048 cascades (near: 10m, mid: 50m, far: 200m, ultra-far: 1000m) = 204, 40, 10, 2 pixels/meter (acceptable quality). Same memory, 100x better quality near camera. Without CSM: pixelated shadows or unusable memory cost (16384² = 1GB per light).

**Technical Explanation:**
CSM splits camera frustum into multiple depth ranges (cascades). Each cascade renders its own shadow map focused on its range. Near cascades have high resolution, far cascades lower. At runtime, fragment shader selects appropriate cascade based on depth, samples shadow map. Prevents perspective aliasing (shadows far from camera are low-res anyway). Requires rendering scene geometry N times (once per cascade), but each cascade sees fewer objects (frustum culling). Modern standard for outdoor scenes with directional lighting.

**Algorithmic Complexity:**
- Single shadow map: O(n) objects rendered once, poor quality/coverage ratio
- CSM: O(n × c) where c = cascades (typically 2-4), but per-cascade n is smaller
- Quality: Exponential improvement near camera (100x better at 10m)
- Memory: Same as single map but distributed better (4×2K² = 1×4K²)
- CPU overhead: Cascade frustum calculation = 0.1-0.3ms per cascade

**Implementation Pattern:**
```gdscript
# Godot Cascaded Shadow Maps System
class_name CascadedShadowMapper
extends Node3D

@export var directional_light: DirectionalLight3D
@export var cascade_count: int = 4
@export var cascade_splits: Array[float] = [0.05, 0.15, 0.5, 1.0]  # Normalized camera frustum
@export var shadow_map_size: int = 2048  # Per cascade
@export var blend_splits: bool = true
@export var blend_distance: float = 10.0  # Meters

var camera: Camera3D
var cascade_frustums: Array = []
var cascade_matrices: Array = []

func _ready():
    camera = get_viewport().get_camera_3d()
    _setup_cascades()

func _setup_cascades():
    if not directional_light:
        return

    # Configure Godot's DirectionalLight3D
    directional_light.shadow_enabled = true
    directional_light.directional_shadow_mode = DirectionalLight3D.SHADOW_PARALLEL_4_SPLITS

    # Set cascade distances
    directional_light.directional_shadow_split_1 = cascade_splits[0]
    directional_light.directional_shadow_split_2 = cascade_splits[1]
    directional_light.directional_shadow_split_3 = cascade_splits[2]

    # Shadow quality settings
    directional_light.shadow_bias = 0.1
    directional_light.shadow_normal_bias = 1.0
    directional_light.directional_shadow_max_distance = 200.0

    # Blend between cascades
    directional_light.directional_shadow_blend_splits = blend_splits

func _process(_delta):
    if camera and directional_light:
        _update_cascade_frustums()

func _update_cascade_frustums():
    cascade_frustums.clear()

    var near = camera.near
    var far = camera.far
    var fov = camera.fov

    # Calculate cascade splits in world space
    var cascade_distances = []
    for split in cascade_splits:
        cascade_distances.append(near + (far - near) * split)

    # Create frustum for each cascade
    for i in range(cascade_count):
        var cascade_near = cascade_distances[i] if i > 0 else near
        var cascade_far = cascade_distances[i + 1] if i < cascade_count else far

        var frustum = _create_cascade_frustum(cascade_near, cascade_far, fov)
        cascade_frustums.append(frustum)

func _create_cascade_frustum(near_dist: float, far_dist: float, fov: float) -> AABB:
    # Calculate frustum corners in world space
    var cam_transform = camera.global_transform
    var forward = -cam_transform.basis.z
    var right = cam_transform.basis.x
    var up = cam_transform.basis.y

    var aspect = get_viewport().get_visible_rect().size.x / get_viewport().get_visible_rect().size.y
    var fov_rad = deg_to_rad(fov)

    # Calculate frustum corners
    var near_height = 2.0 * tan(fov_rad * 0.5) * near_dist
    var near_width = near_height * aspect
    var far_height = 2.0 * tan(fov_rad * 0.5) * far_dist
    var far_width = far_height * aspect

    # Create AABB encompassing frustum corners
    var center_near = camera.global_position + forward * near_dist
    var center_far = camera.global_position + forward * far_dist

    var corners = [
        center_near + up * near_height * 0.5 + right * near_width * 0.5,
        center_near + up * near_height * 0.5 - right * near_width * 0.5,
        center_near - up * near_height * 0.5 + right * near_width * 0.5,
        center_near - up * near_height * 0.5 - right * near_width * 0.5,
        center_far + up * far_height * 0.5 + right * far_width * 0.5,
        center_far + up * far_height * 0.5 - right * far_width * 0.5,
        center_far - up * far_height * 0.5 + right * far_width * 0.5,
        center_far - up * far_height * 0.5 - right * far_width * 0.5,
    ]

    # Create AABB from corners
    var bounds = AABB(corners[0], Vector3.ZERO)
    for corner in corners:
        bounds = bounds.expand(corner)

    return bounds

# Advanced cascade optimization
class OptimizedCSM:
    # Logarithmic split distribution (better quality distribution)

    func calculate_log_splits(cascade_count: int, near: float, far: float, lambda: float = 0.5) -> Array:
        # lambda = 0: uniform, lambda = 1: logarithmic
        var splits = []

        for i in range(cascade_count + 1):
            var p = float(i) / cascade_count

            # Uniform split
            var uniform = near + (far - near) * p

            # Logarithmic split
            var log_split = near * pow(far / near, p)

            # Blend between uniform and logarithmic
            var split = lambda * log_split + (1.0 - lambda) * uniform

            splits.append((split - near) / (far - near))  # Normalize

        return splits

# Cascade culling
class CascadeCuller:
    # Cull objects per cascade (render only what's visible)

    func cull_for_cascade(cascade_frustum: AABB, all_objects: Array) -> Array:
        var visible = []

        for obj in all_objects:
            if not obj is VisualInstance3D:
                continue

            var obj_aabb = obj.get_aabb()
            var world_aabb = AABB(
                obj.global_transform.origin + obj_aabb.position,
                obj_aabb.size
            )

            if cascade_frustum.intersects(world_aabb):
                visible.append(obj)

        return visible

# Shadow fade for distant cascades
class CascadeFader:
    @export var fade_start: float = 0.8  # Start fade at 80% of max distance
    @export var fade_end: float = 1.0

    func calculate_shadow_fade(fragment_depth: float, max_distance: float) -> float:
        var normalized_depth = fragment_depth / max_distance

        if normalized_depth < fade_start:
            return 1.0  # Full shadow

        if normalized_depth > fade_end:
            return 0.0  # No shadow

        # Linear fade
        var fade_range = fade_end - fade_start
        var fade = (normalized_depth - fade_start) / fade_range

        return 1.0 - fade

# Custom CSM shader helper
const CSM_SHADER = """
shader_type spatial;

uniform sampler2DArray cascade_shadow_maps : hint_default_white;
uniform mat4 cascade_matrices[4];
uniform vec4 cascade_splits;  // Split distances
uniform float shadow_bias = 0.001;

varying float vertex_depth;

void vertex() {
    vec4 view_pos = MODELVIEW_MATRIX * vec4(VERTEX, 1.0);
    vertex_depth = -view_pos.z;
}

float sample_shadow(vec3 world_pos, int cascade_index) {
    vec4 shadow_coord = cascade_matrices[cascade_index] * vec4(world_pos, 1.0);
    shadow_coord.xyz /= shadow_coord.w;
    shadow_coord.xyz = shadow_coord.xyz * 0.5 + 0.5;

    if (shadow_coord.x < 0.0 || shadow_coord.x > 1.0 ||
        shadow_coord.y < 0.0 || shadow_coord.y > 1.0 ||
        shadow_coord.z < 0.0 || shadow_coord.z > 1.0) {
        return 1.0;  // Outside shadow map
    }

    float shadow_depth = texture(cascade_shadow_maps, vec3(shadow_coord.xy, float(cascade_index))).r;
    float current_depth = shadow_coord.z - shadow_bias;

    return current_depth <= shadow_depth ? 1.0 : 0.0;
}

int select_cascade(float depth) {
    if (depth < cascade_splits.x) return 0;
    if (depth < cascade_splits.y) return 1;
    if (depth < cascade_splits.z) return 2;
    return 3;
}

void fragment() {
    vec3 world_pos = (INV_VIEW_MATRIX * vec4(VERTEX, 1.0)).xyz;

    // Select appropriate cascade
    int cascade = select_cascade(vertex_depth);

    // Sample shadow
    float shadow = sample_shadow(world_pos, cascade);

    // Optional: Blend between cascades
    // ...

    // Apply shadow to lighting
    ALBEDO = vec3(1.0) * shadow;
}
"""

# Cascade visualization for debugging
class CascadeDebugger:
    var debug_colors: Array[Color] = [
        Color.RED,
        Color.GREEN,
        Color.BLUE,
        Color.YELLOW
    ]

    func visualize_cascades(fragment_depth: float, cascade_splits: Array) -> Color:
        for i in range(cascade_splits.size()):
            if fragment_depth < cascade_splits[i]:
                return debug_colors[i]

        return debug_colors[cascade_splits.size() - 1]
```

**Key Parameters:**
- Cascade count: 2-4 (2 = performance, 4 = quality)
- Split distribution: Logarithmic (λ=0.5-0.9) for better near-camera quality
- Shadow map size: 1024-2048 per cascade (4096 for hero cascades)
- Blend distance: 5-20m between cascades (smooth transitions)
- Max shadow distance: 100-500m (further = lower quality)
- Shadow bias: 0.05-0.2 (prevent acne, avoid peter-panning)
- Normal bias: 0.5-2.0 (offset along normal)

**Edge Cases:**
- Cascade popping: Blend between cascades (10-20m transition)
- Shadow acne: Increase bias or normal bias
- Peter panning: Decrease bias, increase normal bias
- Flickering: Stabilize cascade frustums (round to texel grid)
- Extreme FOV: Adjust split distribution
- Fast camera movement: Increase cascade overlap

**When NOT to Use:**
- Indoor scenes (point/spot lights more appropriate)
- Single directional light with small coverage (single shadow map sufficient)
- Stylized games without shadows
- Very low-end hardware (shadow maps too expensive)
- Top-down 2D games

**Examples from Shipped Games:**

1. **The Witcher 3:** 4 cascades, 2048² per cascade. Splits: 15m, 50m, 150m, 500m. Logarithmic distribution (λ=0.7). Saved 18ms vs single 8K map (same quality). Essential for open-world performance.

2. **Red Dead Redemption 2:** Dynamic cascade count (2-4 based on scene). 2048-4096² per cascade. Advanced stabilization (minimal flickering). 30fps on console, shadows <8ms.

3. **Far Cry 6:** 3 cascades + contact shadows. Splits: 10m, 40m, 200m. Aggressive culling per cascade (50% fewer objects in far cascades). Maintained 60fps on PS5.

4. **Ghost of Tsushima:** 4 cascades with aggressive LOD per cascade. Near: full detail, far: simplified geometry. Shadow rendering 12ms → 4ms. Crucial for 60fps mode.

5. **Horizon Forbidden West:** Hybrid CSM + SDSM (Sample Distribution Shadow Maps). 4 cascades, dynamic resolution per cascade. Maintained quality at 60fps on PS5.

**Platform Considerations:**
- **PC High-End:** 4 cascades, 2048-4096² per cascade, long shadow distance (500m+)
- **PC Low-End:** 2 cascades, 1024² per cascade, short distance (100m)
- **Console (PS5/XSX):** 3-4 cascades, 2048² per cascade, moderate distance (200-300m)
- **Last-Gen Console:** 2-3 cascades, 1024-2048² per cascade, 100-200m distance
- **Mobile:** 1-2 cascades, 512-1024² per cascade, 50-100m distance
- **Memory:** Cascades share resolution budget (4×2K² = 1×4K² = 64MB)

**Godot-Specific Notes:**
- `DirectionalLight3D.directional_shadow_mode` controls cascade count
- `SHADOW_PARALLEL_4_SPLITS` = 4 cascades (best quality)
- `SHADOW_PARALLEL_2_SPLITS` = 2 cascades (performance)
- `directional_shadow_split_1/2/3` sets split ratios
- `directional_shadow_blend_splits` enables smooth transitions
- `directional_shadow_max_distance` limits shadow range
- Godot 4.x: Automatic cascade stabilization (reduced flickering)
- Use `shadow_bias` and `shadow_normal_bias` to prevent artifacts

**Synergies:**
- Essential for outdoor Open World Games (sun shadows)
- Pairs with LOD Systems (lower LOD for far cascades)
- Combines with Frustum Culling (cull per cascade)
- Works with Contact Shadows (enhance near-cascade detail)
- Enables Time-of-Day Systems (dynamic sun angle)

**Measurement/Profiling:**
- GPU profiler: Shadow pass time (should be 3-8ms for 4 cascades)
- Track cascade utilization: Each cascade should cover appropriate area
- Measure artifacts: Shadow acne, peter panning, flickering
- Visual debug: Color-code cascades to verify splits
- Performance: 4 cascades typically 2-3x cost of 1, but 100x better quality
- Target: <8ms total shadow time, <2ms per cascade
- Memory bandwidth: 4×2K shadow maps = ~256MB/frame read bandwidth

**Additional Resources:**
- GPU Gems 3: Chapter 10 "Parallel-Split Shadow Maps"
- GDC 2008: "Shadow Rendering in Frostbite"
- Microsoft: Cascaded Shadow Maps Best Practices
