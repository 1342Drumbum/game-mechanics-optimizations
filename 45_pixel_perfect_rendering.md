### 45. Pixel-Perfect Rendering

**Category:** Performance - 2D Rendering Quality & Optimization

**Problem It Solves:** Sub-pixel rendering causes blurry sprites and wasted GPU cycles. Without pixel-perfect constraints, GPU performs bilinear filtering on every sprite every frame. For 1000 sprites at 32×32 pixels = 1M filtered pixels per frame = 16ms at 16ns per pixel. Pixel-perfect snapping eliminates filtering, reduces to 0.5ms, and produces crisp visuals.

**Technical Explanation:**
Pixel-perfect rendering ensures sprites align to exact pixel boundaries on screen. Round sprite positions to integers, disable texture filtering (use nearest-neighbor), match camera zoom to integer multiples of native resolution. This eliminates sub-pixel offsets that trigger bilinear interpolation. GPU can use faster point sampling instead of weighted average of 4 pixels.

Implementation requires careful coordination: viewport size must be integer multiple of design resolution, camera zoom must be whole numbers (1x, 2x, 3x), sprite positions rounded to integers after physics/movement calculations. Use shader tricks for sub-pixel movement in animation while maintaining pixel alignment. Critical for retro-style games where blur destroys aesthetic.

**Algorithmic Complexity:**
- Filtered Rendering: O(n × 4) samples per pixel (bilinear interpolation)
- Pixel-Perfect: O(n × 1) samples per pixel (point sampling)
- Position Rounding: O(1) per sprite vs O(0) for raw floats
- Overall: 4× faster texture sampling, cleaner cache usage

**Implementation Pattern:**
```gdscript
# Pixel-Perfect Camera Controller
class_name PixelPerfectCamera
extends Camera2D

@export var design_resolution: Vector2i = Vector2i(320, 180)  # Base resolution
@export var snap_to_pixels: bool = true
@export var integer_zoom_only: bool = true

var target_zoom: float = 1.0
var actual_zoom: int = 1

func _ready():
    # Disable texture filtering globally
    RenderingServer.set_default_clear_color(Color.BLACK)

    # Calculate initial zoom
    calculate_zoom()

func _process(_delta):
    if snap_to_pixels:
        # Round camera position to pixels
        var snapped_pos = (global_position / actual_zoom).round() * actual_zoom
        global_position = snapped_pos

    if integer_zoom_only:
        calculate_zoom()

func calculate_zoom():
    var viewport_size = get_viewport_rect().size
    var zoom_x = floor(viewport_size.x / design_resolution.x)
    var zoom_y = floor(viewport_size.y / design_resolution.y)

    # Use smallest zoom to ensure entire design resolution fits
    actual_zoom = max(1, int(min(zoom_x, zoom_y)))
    zoom = Vector2.ONE * actual_zoom

# Pixel-Perfect Sprite Handler
class_name PixelPerfectSprite
extends Sprite2D

@export var snap_to_pixels: bool = true

func _ready():
    # Disable texture filtering
    texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST

    # Ensure no texture repeat
    texture_repeat = CanvasItem.TEXTURE_REPEAT_DISABLED

func _process(_delta):
    if snap_to_pixels:
        # Round position to nearest pixel
        position = position.round()

# Advanced: Sub-pixel movement with pixel-perfect rendering
class_name SubPixelCharacter
extends CharacterBody2D

var sub_pixel_position: Vector2 = Vector2.ZERO
var render_position: Vector2 = Vector2.ZERO

@onready var sprite: Sprite2D = $Sprite2D

func _ready():
    sprite.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST
    sub_pixel_position = global_position

func _physics_process(delta):
    # Physics uses sub-pixel precision
    var movement = velocity * delta
    sub_pixel_position += movement

    # Update physics body position
    global_position = sub_pixel_position

    # Round sprite position for rendering
    render_position = sub_pixel_position.round()
    sprite.global_position = render_position

# Viewport-based pixel-perfect setup
class_name PixelPerfectViewport
extends SubViewport

@export var base_resolution: Vector2i = Vector2i(320, 180)
@export var scaling_mode: ScalingMode = ScalingMode.INTEGER_SCALE

enum ScalingMode {
    INTEGER_SCALE,  # Only 1x, 2x, 3x, etc.
    ASPECT_RATIO,   # Maintain aspect ratio, allow fractional
    STRETCH         # Fill screen (not recommended for pixel-perfect)
}

func _ready():
    # Set viewport size to design resolution
    size = base_resolution

    # Disable anti-aliasing
    msaa_2d = Viewport.MSAA_DISABLED
    screen_space_aa = Viewport.SCREEN_SPACE_AA_DISABLED

    # Setup texture filtering
    canvas_item_default_texture_filter = Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST

func _process(_delta):
    var window_size = DisplayServer.window_get_size()
    update_scaling(window_size)

func update_scaling(window_size: Vector2i):
    match scaling_mode:
        ScalingMode.INTEGER_SCALE:
            var scale_x = window_size.x / base_resolution.x
            var scale_y = window_size.y / base_resolution.y
            var scale = max(1, int(min(scale_x, scale_y)))

            # Center viewport with black bars if needed
            var scaled_size = base_resolution * scale
            var offset = (window_size - scaled_size) / 2
            # Apply to parent TextureRect showing viewport

        ScalingMode.ASPECT_RATIO:
            var aspect = float(base_resolution.x) / float(base_resolution.y)
            var window_aspect = float(window_size.x) / float(window_size.y)

            if window_aspect > aspect:
                # Window wider than design - pillarbox
                var height = window_size.y
                var width = int(height * aspect)
                size = Vector2i(width, height)
            else:
                # Window taller - letterbox
                var width = window_size.x
                var height = int(width / aspect)
                size = Vector2i(width, height)

# Shader for pixel-perfect scaling with CRT effect (optional)
const PIXEL_PERFECT_SHADER = """
shader_type canvas_item;

uniform sampler2D screen_texture : hint_screen_texture, filter_nearest;
uniform vec2 design_resolution = vec2(320.0, 180.0);
uniform float pixel_size = 1.0;

void fragment() {
    // Snap UV to pixel grid
    vec2 pixel_uv = floor(UV * design_resolution) / design_resolution;

    // Sample with nearest filtering
    COLOR = texture(screen_texture, pixel_uv);

    // Optional: Add scanlines for CRT effect
    if (mod(FRAGCOORD.y, pixel_size * 2.0) < pixel_size) {
        COLOR.rgb *= 0.9;
    }
}
"""
```

**Key Parameters:**
- **Design Resolution:** 320×180 (NES-like), 640×360 (SNES-like), 1280×720 (modern pixel art)
- **Zoom Levels:** 1× (1:1), 2× (2:1), 3× (3:1) - avoid fractional (1.5×, 2.7×)
- **Texture Filter:** NEAREST always, never LINEAR/BILINEAR
- **Position Precision:** Round to 1.0 pixels, not 0.1 or smaller
- **Viewport MSAA:** DISABLED - anti-aliasing ruins pixel-perfect
- **Canvas Item Filter:** DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST

**Edge Cases:**
- **Window Resize:** Must recalculate integer zoom, may introduce black bars
- **Rotation:** Non-90° rotations break pixel-perfect, avoid or accept blur
- **Scaling:** Non-integer scales blur pixels, use integer zoom only
- **Particle Effects:** Difficult to keep pixel-perfect with alpha blending
- **UI Scaling:** UI may need separate rendering path for high DPI
- **Animation:** Smooth sub-pixel movement requires offset tricks (separate visual/physics position)

**When NOT to Use:**
- High-resolution realistic art - defeats purpose of high-res assets
- Smooth rotation required - pixel-perfect and rotation incompatible
- Mobile with varying DPI - better to embrace smooth scaling
- 3D-rendered sprites - already pixel-aligned at render
- Accessibility needs - some players prefer slight blur for readability

**Examples from Shipped Games:**

1. **Celeste:** Strict 320×180 base resolution, integer scaling only (1×-6× depending on display). All sprites snap to pixels, camera movement rounded. Character animation uses sub-pixel tricks for smoothness while maintaining crisp appearance. Critical for platforming precision and visual clarity.

2. **Dead Cells:** 1920×1080 native but with pixel-art aesthetic. Uses 3× pixel size guideline (each "pixel" is 3×3 screen pixels). Maintains pixel-perfect at 1× and 2× zoom levels. Particles slightly blur for effect but core sprites always crisp.

3. **Shovel Knight:** 400×240 base resolution (NES-inspired). Enforces integer scaling with black bars if needed. Zero tolerance for blur - all sprites, tiles, UI pixel-perfect. Used custom engine to guarantee perfect NES-era appearance across all platforms.

4. **Stardew Valley:** 1280×720 base with pixel-perfect rendering. Sprites at 16×16 and 32×32 exact. Camera snaps to 1-pixel grid. UI scaling separate system for 4K displays. Maintains crisp pixels at all supported resolutions via integer scaling.

5. **Enter the Gungeon:** 640×360 base resolution, 2× scaling to 1280×720. All sprites nearest-neighbor filtered. Bullets and effects snap to pixel grid for clarity in bullet-hell chaos. Camera shake rounded to full pixels to prevent wobble blur.

**Platform Considerations:**
- **Mobile (iOS/Android):** Challenging due to varied resolutions (1080p, 1440p, etc.). Use design resolution with integer scale + letterboxing. Target 2× or 3× scale for most devices.
- **Desktop (Windows/Mac/Linux):** Easiest platform - users accept black bars for pixel-perfect. Support windowed mode with integer scales. Allow users to choose zoom level.
- **Console (Switch/PS/Xbox):** Fixed resolution simplifies setup. Switch docked (1920×1080) vs handheld (1280×720) needs different scale factors.
- **WebGL:** Must handle arbitrary browser window sizes. Default to aspect-ratio scaling with nearest-neighbor filter. Performance cost of rescaling every frame.
- **High DPI Displays:** 4K monitors (3840×2160) perfect for 6× scale of 640×360. Retina displays need special handling to avoid OS-level scaling interference.

**Godot-Specific Notes:**
- Godot 4.x has built-in pixel snap: Project Settings → Rendering → 2D → Snap 2D Transforms to Pixel
- Use `SubViewport` with `size` set to design resolution, render to `TextureRect` with `NEAREST` filter
- Set `canvas_item_default_texture_filter` in Project Settings → Rendering → Textures
- Monitor `get_tree().root.content_scale_mode` for different scaling strategies
- `Camera2D.position_smoothing_enabled` should be false for pixel-perfect
- Use `floor()` and `round()` liberally in movement code
- Godot's `Camera2D.smoothing_speed` incompatible with pixel-perfect

**Synergies:**
- **Sprite Batching:** Pixel-perfect sprites batch better (no filtering variations)
- **Tilemap Rendering:** Tiles naturally pixel-aligned, perfect synergy
- **Canvas Layer Management:** Each layer can be pixel-perfect independently
- **Dirty Rectangles:** Pixel-aligned updates more cache-friendly
- **Integer Zoom Only:** Simplifies all other rendering optimizations

**Measurement/Profiling:**
- **Visual Inspection:** Zoom in on sprites - should see hard pixel edges, no blur
- **Texture Filtering Check:** Monitor Project Settings → Rendering, verify NEAREST everywhere
- **Position Precision:** Log sprite positions - should be whole numbers (123.0, 45.0 not 123.7, 45.3)
- **Overdraw Reduction:** Pixel-perfect often reduces overdraw by 10-20% (no filtering overlap)
- **Performance Test:**
```gdscript
# Measure texture sampling cost
var time_before = Performance.get_monitor(Performance.TIME_RENDER)
# Toggle between NEAREST and LINEAR filtering
sprite.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST  # vs LINEAR
var time_after = Performance.get_monitor(Performance.TIME_RENDER)
print("Render time delta: ", (time_after - time_before) * 1000, "ms")
```

**Common Setup Patterns:**
```gdscript
# Pattern 1: Project-wide pixel-perfect setup
# In project.godot or Project Settings
# Rendering → Textures → Canvas Textures → Default Texture Filter = Nearest
# Rendering → 2D → Snap 2D Transforms to Pixel = Enabled

# Pattern 2: Per-sprite guarantee
func ensure_pixel_perfect(sprite: Sprite2D):
    sprite.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST
    sprite.centered = true  # Helps with rounding
    sprite.position = sprite.position.round()

# Pattern 3: Camera setup
func setup_pixel_perfect_camera(camera: Camera2D, design_res: Vector2i):
    camera.position_smoothing_enabled = false
    camera.rotation_smoothing_enabled = false

    var viewport_size = camera.get_viewport_rect().size
    var zoom_factor = max(1, int(min(
        viewport_size.x / design_res.x,
        viewport_size.y / design_res.y
    )))
    camera.zoom = Vector2.ONE * zoom_factor

# Pattern 4: Physics/rendering separation
var physics_position: Vector2  # Sub-pixel accurate
var render_position: Vector2   # Pixel-snapped

func _physics_process(delta):
    physics_position += velocity * delta
    global_position = physics_position  # Physics uses exact position

func _process(_delta):
    render_position = physics_position.round()
    $Sprite2D.global_position = render_position  # Rendering uses snapped
```
