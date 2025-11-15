### 44. Parallax Layer Optimization

**Category:** Performance - 2D Rendering & Visual Effects

**Problem It Solves:** Multiple parallax layers cause redundant rendering and transform calculations. 5 parallax layers × 1920×1080 resolution × 60fps = 622M pixel fills per second. Naive implementation recalculates all layers every frame even when camera static, wasting 30-50% of render time. Optimized parallax reduces updates by 90%+ for static cameras.

**Technical Explanation:**
Parallax scrolling creates depth illusion by moving background layers at different speeds relative to camera. Distant mountains move slower than nearby trees. Naive implementation updates all layer positions every frame. Optimization: only update when camera moves, cache layer positions, cull off-screen layers, use texture wrapping for infinite scrolling, and batch layers with same scroll speed. GPU texture wrapping eliminates expensive position recalculation.

Modern approach uses shader-based parallax - calculate offset in vertex/fragment shader using camera position uniform. Eliminates CPU transforms entirely. For complex scenes, group layers by update frequency: static (never move), slow (0.1-0.5× camera), medium (0.5-0.8×), foreground (0.9-1.0×). Only recalculate changed groups.

**Algorithmic Complexity:**
- Naive: O(n × p) every frame where n = layers, p = pixels per layer
- Optimized: O(n) when camera moves (position update only), O(0) when static
- Shader-based: O(p) with GPU parallelization, no CPU cost
- Culling: O(k) where k = visible layers (typically 50% of total)

**Implementation Pattern:**
```gdscript
# Optimized Parallax Background Manager
class_name ParallaxManager
extends ParallaxBackground

var last_camera_pos: Vector2 = Vector2.ZERO
var camera_moved: bool = false
const MOVEMENT_THRESHOLD = 0.1  # Pixels - ignore tiny movements

# Layer groups by update frequency
var static_layers: Array[ParallaxLayer] = []
var slow_layers: Array[ParallaxLayer] = []
var fast_layers: Array[ParallaxLayer] = []

func _ready():
    # Categorize layers
    for child in get_children():
        if child is ParallaxLayer:
            var motion_scale = child.motion_scale.x
            if motion_scale < 0.3:
                static_layers.append(child)
            elif motion_scale < 0.7:
                slow_layers.append(child)
            else:
                fast_layers.append(child)

            # Enable texture wrapping for infinite scroll
            setup_layer_wrapping(child)

func _process(_delta):
    var camera = get_viewport().get_camera_2d()
    if not camera:
        return

    var camera_pos = camera.get_screen_center_position()

    # Check if camera moved significantly
    camera_moved = camera_pos.distance_squared_to(last_camera_pos) > MOVEMENT_THRESHOLD * MOVEMENT_THRESHOLD

    if camera_moved:
        # Update layers based on camera movement delta
        var camera_delta = camera_pos - last_camera_pos

        # Fast layers update every frame
        for layer in fast_layers:
            layer.motion_offset += camera_delta * (1.0 - layer.motion_scale.x)

        # Slow layers can skip frames
        if Engine.get_frames_drawn() % 2 == 0:  # Update every 2 frames
            for layer in slow_layers:
                layer.motion_offset += camera_delta * (1.0 - layer.motion_scale.x) * 2.0

        # Static layers rarely update
        if Engine.get_frames_drawn() % 4 == 0:  # Update every 4 frames
            for layer in static_layers:
                layer.motion_offset += camera_delta * (1.0 - layer.motion_scale.x) * 4.0

        last_camera_pos = camera_pos

func setup_layer_wrapping(layer: ParallaxLayer):
    # Enable mirroring for infinite scroll
    layer.motion_mirroring = layer.get_child(0).texture.get_size() * layer.scale

    # Ensure textures have REPEAT flag
    for child in layer.get_children():
        if child is Sprite2D and child.texture:
            child.texture.flags |= Texture2D.FLAG_REPEAT

# Shader-based parallax (most performant)
class_name ShaderParallax
extends Node2D

const PARALLAX_SHADER = """
shader_type canvas_item;

uniform sampler2D layer_texture : repeat_enable;
uniform vec2 scroll_speed = vec2(0.5, 0.0);
uniform vec2 camera_offset;

void fragment() {
    vec2 parallax_uv = UV + (camera_offset * scroll_speed) / TEXTURE_PIXEL_SIZE;
    COLOR = texture(layer_texture, parallax_uv);
}
"""

var camera_offset: Vector2 = Vector2.ZERO
var layers: Array[ColorRect] = []

func create_layer(texture: Texture2D, speed: Vector2, z_index: int):
    var layer = ColorRect.new()
    layer.material = ShaderMaterial.new()
    layer.material.shader = preload("res://shaders/parallax.gdshader")
    layer.material.set_shader_parameter("layer_texture", texture)
    layer.material.set_shader_parameter("scroll_speed", speed)
    layer.size = get_viewport_rect().size
    layer.z_index = z_index
    add_child(layer)
    layers.append(layer)

func _process(_delta):
    var camera = get_viewport().get_camera_2d()
    if camera:
        camera_offset = camera.get_screen_center_position()
        # Update shader uniform for all layers
        for layer in layers:
            layer.material.set_shader_parameter("camera_offset", camera_offset)

# Culled parallax - only render visible layers
class_name CulledParallax
extends ParallaxBackground

func _process(_delta):
    var viewport_rect = get_viewport_rect()
    var camera = get_viewport().get_camera_2d()
    if not camera:
        return

    var camera_pos = camera.get_screen_center_position()

    for child in get_children():
        if not child is ParallaxLayer:
            continue

        # Calculate layer world bounds
        var layer_offset = child.motion_offset + child.motion_scale * camera_pos
        var layer_bounds = Rect2(layer_offset, viewport_rect.size)

        # Cull if completely outside viewport (with margin)
        var margin = 200  # Extra pixels for smooth appear/disappear
        var expanded_viewport = viewport_rect.grow(margin)

        child.visible = expanded_viewport.intersects(layer_bounds)

# Dirty flag optimization
class_name DirtyParallax
extends ParallaxBackground

var dirty: bool = false
var cached_positions: Dictionary = {}  # Layer -> last position

func _process(_delta):
    if not dirty:
        return

    var camera = get_viewport().get_camera_2d()
    if not camera:
        return

    for child in get_children():
        if child is ParallaxLayer:
            var new_pos = calculate_layer_position(child, camera)
            if cached_positions.get(child, Vector2.INF) != new_pos:
                child.position = new_pos
                cached_positions[child] = new_pos

    dirty = false

func mark_dirty():
    dirty = true

func calculate_layer_position(layer: ParallaxLayer, camera: Camera2D) -> Vector2:
    return layer.motion_offset + layer.motion_scale * camera.get_screen_center_position()
```

**Key Parameters:**
- **Layer Count:** 3-5 layers (mobile), 5-10 (desktop), 15+ causes performance issues
- **Motion Scale:** 0.1-0.2 (far background), 0.4-0.6 (mid), 0.8-0.95 (foreground)
- **Texture Size:** 1024-2048px wide (repeating), match viewport height
- **Update Frequency:** Fast layers every frame, slow layers every 2-4 frames
- **Mirroring Distance:** Texture width × scale for seamless repeat
- **Culling Margin:** 200-500px beyond viewport edges

**Edge Cases:**
- **Camera Teleportation:** Instant camera jumps require all layers immediate update
- **Zoom Changes:** Layer scales must recalculate, mirroring distances change
- **Layer Ordering:** Z-index conflicts with regular sprites, use negative z-index for backgrounds
- **Precision Loss:** Very slow layers (0.01×) accumulate floating-point error over time
- **Seams in Tiling:** Mirroring artifacts if textures not seamless, use pixel-perfect alignment
- **Dynamic Layer Addition:** Adding layers mid-game requires recalculation of all positions

**When NOT to Use:**
- Single-screen games without scrolling - static background cheaper
- 3D games - use actual depth instead
- Heavy sprite-based parallax - individual sprites better than layered textures
- Extremely fast scrolling - motion blur hides parallax effect anyway
- Limited viewport size - phone screens may not show depth effectively

**Examples from Shipped Games:**

1. **Celeste:** 4-6 parallax layers per level (clouds, mountains, distant buildings). Shader-based implementation, zero CPU cost. Uses mirroring for infinite vertical scrolling in Core levels. Optimized by culling layers outside current room. Critical for maintaining 60fps during high-speed dashes.

2. **Dead Cells:** 3-5 parallax layers per biome with distinct atmosphere (fog, ruins, background enemies). Updates only when camera moves. Layers batch together sharing same shader. Foreground vines/chains at 0.95× create convincing depth in 2D.

3. **Hollow Knight:** Extensive parallax in City of Tears (6-8 layers of distant buildings/rain). Slow update rate (30fps) for background layers while gameplay at 60fps. Uses dirty flag - layers freeze when player stands still. Saves ~15% render time during exploration.

4. **Terraria:** Parallax backgrounds generated procedurally per biome. 4-6 layers typical, ocean biome has animated wave layers. Uses texture wrapping for infinite horizontal scroll. Layers cached when underground (not visible), major optimization.

5. **Stardew Valley:** Simple 2-3 layer parallax (clouds, trees). Extremely optimized - updates only on screen transitions. Shader-based on newer platforms. Prioritizes compatibility over visual complexity, proving effective parallax doesn't need many layers.

**Platform Considerations:**
- **Mobile (iOS/Android):** Limit to 3-5 layers, use shader-based parallax, update slow layers at 30fps. Fragment fillrate critical - avoid overdraw. Target <2ms for parallax rendering.
- **Desktop (Windows/Mac/Linux):** 5-10 layers comfortable, can use CPU-based with optimization. Higher resolution textures (2048-4096px) for 4K displays.
- **Console (Switch/PS/Xbox):** Switch: 4-6 layers, optimize via shader. PS5/Xbox: 10+ layers fine. Fixed performance budget easier to optimize.
- **WebGL:** Shader-based essential, CPU transforms too slow. Limit to 4-5 layers, use compressed textures.
- **VRAM:** Each 2048×1024 layer = ~8MB RGBA. Use compressed textures (DXT1/ETC2) to reduce by 75%.

**Godot-Specific Notes:**
- `ParallaxBackground` automatically handles parallax math, use `motion_scale` and `motion_offset`
- Set `ParallaxLayer.motion_mirroring` to texture size for infinite scroll
- Use `limit_begin` and `limit_end` on Camera2D to constrain parallax bounds
- Godot 4.x: Shader-based parallax more efficient than node-based
- Monitor with Profiler → Visual → CanvasItem Updates
- Use `CanvasLayer` instead of `ParallaxLayer` for UI elements that shouldn't scroll
- `ignore_camera_zoom` property prevents layers from scaling with camera zoom

**Synergies:**
- **Sprite Batching:** Parallax layers batch together if same texture/shader
- **Dirty Rectangles:** Only redraw parallax when camera moves
- **Canvas Layer Management:** Separate parallax into background CanvasLayer (z=-100)
- **Tilemap Chunking:** Combine with parallax for massive scrolling worlds
- **2D Lighting:** Apply lights to foreground parallax layers only for depth

**Measurement/Profiling:**
- **Render Time:** Monitor `Performance.TIME_RENDER` - parallax should be <10% of frame
- **Update Frequency:** Track camera movement delta, verify layers skip updates when static
- **Overdraw:** Use Godot overdraw visualizer - parallax can cause 3-5× overdraw
- **Memory:** Track texture memory with `Performance.RENDER_TEXTURE_MEM_USED`
- **Profiling Pattern:**
```gdscript
var updates_this_second = 0
var last_second = Time.get_ticks_msec()

func _process(_delta):
    if camera_moved:
        updates_this_second += 1

    var current_time = Time.get_ticks_msec()
    if current_time - last_second > 1000:
        print("Parallax updates/sec: ", updates_this_second)
        updates_this_second = 0
        last_second = current_time
```

**Advanced Optimization Techniques:**
```gdscript
# Technique 1: LOD for parallax layers
func update_layer_lod(layer: ParallaxLayer, fps: float):
    if fps < 50:  # Performance struggling
        if layer.motion_scale.x < 0.3:  # Far layers
            layer.visible = false  # Disable far layers
    elif fps < 55:
        layer.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST  # Lower quality
    else:
        layer.texture_filter = CanvasItem.TEXTURE_FILTER_LINEAR

# Technique 2: Predictive loading for fast movement
func predict_needed_layers(camera_velocity: Vector2):
    var prediction_distance = camera_velocity * 0.5  # 0.5 seconds ahead
    for layer in all_layers:
        var layer_bounds = get_layer_bounds(layer)
        var predicted_viewport = current_viewport.grow(prediction_distance.length())
        layer.visible = predicted_viewport.intersects(layer_bounds)

# Technique 3: Cached shader uniforms
var cached_camera_pos: Vector2
var shader_update_needed: bool = false

func _process(_delta):
    var new_pos = camera.get_screen_center_position()
    if new_pos.distance_squared_to(cached_camera_pos) > 1.0:
        cached_camera_pos = new_pos
        shader_update_needed = true

    if shader_update_needed:
        for layer in shader_layers:
            layer.material.set_shader_parameter("camera_pos", cached_camera_pos)
        shader_update_needed = false
```
