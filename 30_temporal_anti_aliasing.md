### 30. Temporal Anti-Aliasing (TAA)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Anti-aliasing quality vs performance trade-off. MSAA 8x = render at 8 samples per pixel = 8x fragment shader cost = 128ms (7.7x over budget). TAA achieves similar quality with ~1-2ms overhead by reusing previous frames, accumulating sub-pixel jitter over time for super-sampling effect without multi-sampling cost.

**Technical Explanation:**
TAA renders scene with sub-pixel camera jitter (offset by fractions of pixel each frame), accumulates results across frames using motion vectors. Each frame blends current sample with reprojected history (90-95% history, 5-10% current). Motion vectors track pixel movement between frames, enabling accurate history lookup. Eliminates temporal aliasing (flickering edges), reduces shader aliasing, smooths specular highlights. Drawbacks: Motion blur on camera movement, ghosting on fast-moving objects, requires stable 30+ fps. Modern standard for high-quality AA with minimal cost.

**Algorithmic Complexity:**
- MSAA 4x: O(4 × pixels × shader_cost)
- TAA: O(pixels × (shader_cost + history_sample + motion_vector))
- TAA overhead: ~10-20% vs no AA, vs 400% for MSAA 4x
- Memory: 2-4 history buffers (double/triple buffering)
- Quality: Approaches 8x-16x MSAA over 8-16 frames

**Implementation Pattern:**
```gdscript
# Godot Temporal Anti-Aliasing
class_name TAAEffect
extends Node

@export var taa_enabled: bool = true
@export var jitter_strength: float = 1.0
@export var history_blend: float = 0.95  # 95% history, 5% current
@export var sharpness: float = 0.5

var camera: Camera3D
var viewport: Viewport
var frame_count: int = 0
var jitter_pattern: Array[Vector2] = []

const HALTON_SEQUENCE_2_3 = [
    Vector2(0.5, 0.333333),
    Vector2(0.25, 0.666667),
    Vector2(0.75, 0.111111),
    Vector2(0.125, 0.444444),
    Vector2(0.625, 0.777778),
    Vector2(0.375, 0.222222),
    Vector2(0.875, 0.555556),
    Vector2(0.0625, 0.888889),
]

func _ready():
    camera = get_viewport().get_camera_3d()
    viewport = get_viewport()
    _setup_taa()
    _generate_jitter_pattern()

func _setup_taa():
    # Godot 4.x has built-in TAA
    var environment = viewport.world_3d.environment
    if not environment:
        environment = Environment.new()
        viewport.world_3d.environment = environment

    # Enable TAA through environment
    # Note: Godot's TAA is automatic in many cases

func _generate_jitter_pattern():
    # Use Halton sequence for good distribution
    jitter_pattern = HALTON_SEQUENCE_2_3.duplicate()

func _process(_delta):
    if not taa_enabled or not camera:
        return

    # Apply sub-pixel jitter
    var jitter = _get_current_jitter()
    _apply_camera_jitter(jitter)

    frame_count += 1

func _get_current_jitter() -> Vector2:
    var index = frame_count % jitter_pattern.size()
    var jitter = jitter_pattern[index]

    # Convert to pixel offset (-0.5 to 0.5)
    jitter = (jitter - Vector2(0.5, 0.5)) * jitter_strength

    return jitter

func _apply_camera_jitter(jitter: Vector2):
    # Apply jitter to camera projection
    # This offsets the view by sub-pixel amounts

    var screen_size = viewport.get_visible_rect().size
    var pixel_offset = Vector2(
        jitter.x / screen_size.x,
        jitter.y / screen_size.y
    )

    # Modify projection matrix
    # Godot doesn't expose this directly, would need custom projection

# Custom TAA shader (conceptual)
const TAA_SHADER = """
shader_type spatial;

uniform sampler2D current_frame : hint_screen_texture;
uniform sampler2D history_frame;
uniform sampler2D motion_vectors;
uniform sampler2D depth_texture : hint_depth_texture;

uniform float history_blend = 0.95;
uniform float sharpness = 0.5;
uniform vec2 jitter_offset;

vec3 sample_history(vec2 uv, vec2 motion) {
    // Reproject using motion vectors
    vec2 history_uv = uv - motion;

    // Clamp to screen bounds
    if (history_uv.x < 0.0 || history_uv.x > 1.0 ||
        history_uv.y < 0.0 || history_uv.y > 1.0) {
        return vec3(0.0);
    }

    return texture(history_frame, history_uv).rgb;
}

vec3 clip_history(vec3 history, vec3 current, vec2 uv) {
    // Neighborhood clamping to reduce ghosting
    vec3 near_color = vec3(0.0);
    vec3 near_min = vec3(1e10);
    vec3 near_max = vec3(-1e10);

    // Sample 3x3 neighborhood
    for (int y = -1; y <= 1; y++) {
        for (int x = -1; x <= 1; x++) {
            vec2 offset = vec2(float(x), float(y)) / textureSize(current_frame, 0);
            vec3 neighbor = texture(current_frame, uv + offset).rgb;

            near_min = min(near_min, neighbor);
            near_max = max(near_max, neighbor);
            near_color += neighbor;
        }
    }

    near_color /= 9.0;

    // Clamp history to neighborhood bounds
    vec3 clamped = clamp(history, near_min, near_max);

    return clamped;
}

void fragment() {
    vec2 uv = SCREEN_UV;

    // Sample current frame
    vec3 current = texture(current_frame, uv).rgb;

    // Sample motion vector
    vec2 motion = texture(motion_vectors, uv).rg;

    // Sample and reproject history
    vec3 history = sample_history(uv, motion);

    // Clip history to reduce ghosting
    history = clip_history(history, current, uv);

    // Blend current with history
    vec3 result = mix(current, history, history_blend);

    // Optional: Sharpen to counteract blur
    if (sharpness > 0.0) {
        vec3 blurred = (
            texture(current_frame, uv + vec2(1, 0) / textureSize(current_frame, 0)).rgb +
            texture(current_frame, uv + vec2(-1, 0) / textureSize(current_frame, 0)).rgb +
            texture(current_frame, uv + vec2(0, 1) / textureSize(current_frame, 0)).rgb +
            texture(current_frame, uv + vec2(0, -1) / textureSize(current_frame, 0)).rgb
        ) / 4.0;

        result = result + (result - blurred) * sharpness;
    }

    ALBEDO = result;
}
"""

# Motion vector generation
class MotionVectorGenerator:
    var previous_view_proj: Projection
    var previous_frame: int = -1

    func generate_motion_vectors(camera: Camera3D) -> PackedVector2Array:
        var current_vp = camera.get_camera_projection() * camera.get_camera_transform()

        if previous_frame < 0:
            previous_view_proj = current_vp
            previous_frame = 0
            return PackedVector2Array()

        # Calculate motion vectors for each pixel
        # This would require per-object tracking in real implementation

        var motion_vectors = PackedVector2Array()
        previous_view_proj = current_vp
        previous_frame += 1

        return motion_vectors

# Ghosting reduction
class GhostingReducer:
    @export var disocclusion_threshold: float = 0.1
    @export var velocity_rejection: bool = true

    func detect_disocclusion(current_depth: float, history_depth: float) -> bool:
        # Large depth mismatch = geometry changed (disocclusion)
        return abs(current_depth - history_depth) > disocclusion_threshold

    func reduce_history_blend(motion_magnitude: float, base_blend: float) -> float:
        # Fast motion = reduce history contribution
        if not velocity_rejection:
            return base_blend

        var motion_factor = clamp(motion_magnitude / 10.0, 0.0, 1.0)
        return base_blend * (1.0 - motion_factor)

# Adaptive quality TAA
class AdaptiveTAA:
    @export var min_fps: float = 30.0
    @export var target_fps: float = 60.0

    var current_quality: float = 1.0

    func update_quality(delta: float):
        var current_fps = 1.0 / delta

        if current_fps < min_fps:
            # Reduce quality
            current_quality = max(0.5, current_quality - 0.1)
        elif current_fps > target_fps:
            # Increase quality
            current_quality = min(1.0, current_quality + 0.05)

    func get_jitter_strength() -> float:
        return current_quality

    func get_history_blend() -> float:
        # Higher quality = more history frames
        return 0.9 + current_quality * 0.05
```

**Key Parameters:**
- History blend: 0.90-0.97 (higher = smoother but more ghosting)
- Jitter pattern: 8-16 samples (Halton sequence optimal)
- Neighborhood clamp: 3×3 or 5×5 pixels for ghost reduction
- Sharpening: 0.3-0.7 (counteract TAA blur)
- Velocity threshold: >5 pixels/frame = reduce history weight
- Disocclusion detection: Depth difference >10% = reject history

**Edge Cases:**
- Fast camera motion: Reduce history weight, increase current frame contribution
- Thin objects: Prone to flickering, may need higher sample count
- Transparency: TAA can cause artifacts, need separate pass
- Very low fps (<30): TAA quality degrades, fallback to FXAA
- Scene cuts: Detect and reset history buffer

**When NOT to Use:**
- Unstable framerate (<30fps)
- Very fast-paced games (motion blur objectionable)
- Stylized art style where sharpness critical
- VR (can cause discomfort, though modern VR uses adapted TAA)
- When MSAA available and performant (older games)

**Examples from Shipped Games:**

1. **Uncharted 4**: Custom TAA, 30fps rock-solid on PS4. 1-1.5ms overhead, eliminates edge aliasing. Combined with temporal upscaling (1440p → 4K). Ghosting reduction via velocity rejection.

2. **Control**: High-quality TAA on all platforms, 1.2ms cost. Essential for clean image at 30fps. Reduces specular aliasing on Oldest House's reflective surfaces. PS5 60fps mode uses same TAA.

3. **Red Dead Redemption 2**: Advanced TAA with 16-sample Halton pattern. 1.8ms cost, near 8x MSAA quality. Handles complex foliage/hair without flickering. Scales from Xbox One to PC ultra.

4. **Spider-Man (PS4/PS5)**: TAA + sharpening filter. 1.5ms on PS4, 0.8ms on PS5 (better GPU). Enables 60fps mode on PS5. Reduces aliasing on buildings and webs significantly.

5. **DOOM Eternal**: TAA optional, 1.3ms when enabled. Hybrid with SMAA for sharpness. 60fps target on all platforms maintained. Lower ghosting via aggressive history rejection.

**Platform Considerations:**
- **PC High-End**: TAA + optional MSAA 2x/4x, 1-2ms overhead
- **PC Low-End**: TAA only viable AA method, <1ms
- **Console (PS5/XSX)**: TAA standard, 1-1.5ms at 4K/60fps
- **Last-Gen Console**: TAA essential, 1-2ms at 1080p/30fps
- **Mobile**: TAA too expensive for most, use FXAA instead
- **VR**: Adapted TAA (lower history blend), 1-2ms per eye

**Godot-Specific Notes:**
- Godot 4.x: TAA built into renderer (automatic)
- Works with Forward+ renderer by default
- No manual jitter control exposed (engine handles it)
- `Environment.screen_space_aa` can enable additional FXAA
- TAA always active when rendering at native resolution
- Performance: ~1ms overhead typical
- Scales with resolution (4K = 4x cost of 1080p)

**Synergies:**
- Essential for **Temporal Upscaling** (DLSS-like techniques)
- Pairs with **Screen-Space Reflections** (reduces SSR noise)
- Combines with **Motion Blur** (shares motion vector data)
- Works with **Depth of Field** (temporal accumulation for quality)
- Enhances **Thin Geometry Rendering** (reduces shimmer on wires, hair)

**Measurement/Profiling:**
- GPU profiler: TAA pass time (1-2ms typical)
- Visual: Compare with/without (should eliminate jaggies)
- Ghosting: Fast camera pans, check for trails
- Sharpness: May need additional sharpening pass
- Framerate impact: 5-15% typical
- Target: <2ms at target resolution, no visible ghosting
- Quality: Equivalent to 4x-8x MSAA over time

**Additional Resources:**
- GDC 2016: "Temporal Reprojection Anti-Aliasing in INSIDE"
- Siggraph 2014: "Temporal Antialiasing" (NVIDIA)
- Digital Foundry: TAA comparisons in modern games
