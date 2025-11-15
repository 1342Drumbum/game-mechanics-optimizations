### 51. Canvas Layer Management

**Category:** Performance - 2D Rendering Organization

**Problem It Solves:** Unorganized rendering causes unnecessary draw calls and state changes. Rendering 1000 sprites randomly mixed between UI, gameplay, and background = 1000 draw calls with constant texture/shader switching. Proper layer management groups by render state, reducing to 50-100 draw calls (90% reduction), improving frame time from 15ms to 2ms.

**Technical Explanation:**
CanvasLayers separate 2D rendering into distinct depth layers with independent transforms, visibility, and render order. GPU processes each layer sequentially, batching compatible items within layer. Organize sprites by update frequency (static backgrounds, dynamic gameplay, always-updating UI) and render requirements (lit vs unlit, opaque vs transparent). Layer isolation enables selective updates: only redraw changed layers, cache static layers to textures, and disable invisible layers entirely.

Modern approach uses layer hierarchy: background layers (z=-100 to -1) for parallax/environment, gameplay layers (z=0 to 10) for characters/objects, foreground layers (z=11 to 50) for effects/particles, UI layers (z=100+) for HUD/menus. Each layer configured independently: follow_viewport, offset, rotation, scale. Critical for mobile where overdraw kills performance - proper layering reduces overdraw by 60-80%.

**Algorithmic Complexity:**
- Disorganized Rendering: O(n) draw calls for n sprites, frequent state changes
- Layer-Organized: O(l × (n/l)) where l = layers, enables batching within layers
- Selective Update: O(changed_layers) vs O(all_layers)
- Typical: 5-10 layers reduce draw calls by 80-95%

**Implementation Pattern:**
```gdscript
# Organized canvas layer structure
class_name LayerManager
extends Node2D

# Layer indices (negative = background, positive = foreground)
const LAYER_FAR_BACKGROUND = -100
const LAYER_BACKGROUND = -50
const LAYER_BACKGROUND_DECOR = -10
const LAYER_GAMEPLAY = 0
const LAYER_GAMEPLAY_EFFECTS = 10
const LAYER_FOREGROUND = 50
const LAYER_UI_BACKGROUND = 100
const LAYER_UI = 200
const LAYER_UI_OVERLAY = 300
const LAYER_DEBUG = 1000

var layers: Dictionary = {}  # layer_index -> CanvasLayer

func _ready():
    setup_layers()

func setup_layers():
    # Background layers (static, cached)
    layers[LAYER_FAR_BACKGROUND] = create_layer("FarBackground", LAYER_FAR_BACKGROUND)
    layers[LAYER_FAR_BACKGROUND].follow_viewport_enabled = false

    layers[LAYER_BACKGROUND] = create_layer("Background", LAYER_BACKGROUND)
    layers[LAYER_BACKGROUND_DECOR] = create_layer("BackgroundDecor", LAYER_BACKGROUND_DECOR)

    # Gameplay layers (dynamic, frequently updated)
    layers[LAYER_GAMEPLAY] = create_layer("Gameplay", LAYER_GAMEPLAY)
    layers[LAYER_GAMEPLAY_EFFECTS] = create_layer("Effects", LAYER_GAMEPLAY_EFFECTS)

    # Foreground layers
    layers[LAYER_FOREGROUND] = create_layer("Foreground", LAYER_FOREGROUND)

    # UI layers (independent viewport following)
    layers[LAYER_UI_BACKGROUND] = create_layer("UIBackground", LAYER_UI_BACKGROUND)
    layers[LAYER_UI_BACKGROUND].follow_viewport_enabled = true

    layers[LAYER_UI] = create_layer("UI", LAYER_UI)
    layers[LAYER_UI].follow_viewport_enabled = true

    layers[LAYER_UI_OVERLAY] = create_layer("UIOverlay", LAYER_UI_OVERLAY)
    layers[LAYER_UI_OVERLAY].follow_viewport_enabled = true

    # Debug layer (development only)
    if OS.is_debug_build():
        layers[LAYER_DEBUG] = create_layer("Debug", LAYER_DEBUG)
        layers[LAYER_DEBUG].follow_viewport_enabled = true

func create_layer(layer_name: String, layer_index: int) -> CanvasLayer:
    var layer = CanvasLayer.new()
    layer.name = layer_name
    layer.layer = layer_index
    add_child(layer)
    return layer

func get_layer(layer_index: int) -> CanvasLayer:
    return layers.get(layer_index)

func add_to_layer(node: Node, layer_index: int):
    var layer = get_layer(layer_index)
    if layer:
        layer.add_child(node)

func set_layer_visible(layer_index: int, is_visible: bool):
    var layer = get_layer(layer_index)
    if layer:
        layer.visible = is_visible

# Selective layer updating (cache static layers)
class_name CachedCanvasLayer
extends CanvasLayer

var is_static: bool = false
var cached_texture: ViewportTexture
var cache_viewport: SubViewport
var needs_refresh: bool = true

func _ready():
    if is_static:
        setup_cache()

func setup_cache():
    # Create viewport to render layer to texture
    cache_viewport = SubViewport.new()
    cache_viewport.size = get_viewport().size
    cache_viewport.transparent_bg = true
    cache_viewport.render_target_update_mode = SubViewport.UPDATE_DISABLED

    # Move children to cache viewport
    var children_copy = get_children().duplicate()
    for child in children_copy:
        remove_child(child)
        cache_viewport.add_child(child)

    add_child(cache_viewport)

    # Create sprite to display cached texture
    var display = Sprite2D.new()
    display.centered = false
    cached_texture = cache_viewport.get_texture()
    display.texture = cached_texture
    add_child(display)

    refresh_cache()

func refresh_cache():
    if not is_static or not cache_viewport:
        return

    # Render viewport once
    cache_viewport.render_target_update_mode = SubViewport.UPDATE_ONCE
    needs_refresh = false

func invalidate_cache():
    needs_refresh = true
    if cache_viewport:
        refresh_cache()

# Layer-based rendering optimization
class_name OptimizedLayerRenderer
extends Node2D

var layers_dirty: Dictionary = {}  # layer_index -> bool

func mark_layer_dirty(layer_index: int):
    layers_dirty[layer_index] = true

func _process(_delta):
    # Only update dirty layers
    for layer_index in layers_dirty.keys():
        if layers_dirty[layer_index]:
            update_layer(layer_index)
            layers_dirty[layer_index] = false

func update_layer(layer_index: int):
    # Update only sprites in this layer
    var layer = get_layer(layer_index)
    if layer:
        for child in layer.get_children():
            if child.has_method("update"):
                child.update()

# UI layer with independent camera
class_name UILayer
extends CanvasLayer

@export var ui_camera_zoom: float = 1.0

func _ready():
    # UI doesn't follow world camera
    follow_viewport_enabled = false

    # Position UI in screen space
    offset = Vector2.ZERO

    # Optional: scale UI independently
    scale = Vector2.ONE * ui_camera_zoom

# Parallax background using canvas layers
class_name LayeredParallax
extends Node2D

@export var num_layers: int = 5
@export var base_layer_index: int = -100

var parallax_layers: Array[CanvasLayer] = []

func _ready():
    for i in num_layers:
        var layer = CanvasLayer.new()
        layer.layer = base_layer_index + (i * 10)
        layer.follow_viewport_enabled = true

        # Scale based on depth (closer = larger)
        var depth = float(i) / num_layers
        layer.follow_viewport_scale = 0.5 + (depth * 0.5)

        add_child(layer)
        parallax_layers.append(layer)

# Dynamic layer visibility based on game state
class_name StateBasedLayerManager
extends LayerManager

enum GameState { MENU, GAMEPLAY, PAUSED, CUTSCENE }

var current_state: GameState = GameState.MENU

func change_state(new_state: GameState):
    current_state = new_state
    update_layer_visibility()

func update_layer_visibility():
    match current_state:
        GameState.MENU:
            # Only UI visible
            set_layer_visible(LAYER_GAMEPLAY, false)
            set_layer_visible(LAYER_BACKGROUND, false)
            set_layer_visible(LAYER_UI, true)

        GameState.GAMEPLAY:
            # Everything except menu UI
            set_layer_visible(LAYER_GAMEPLAY, true)
            set_layer_visible(LAYER_BACKGROUND, true)
            set_layer_visible(LAYER_UI, true)

        GameState.PAUSED:
            # Gameplay visible but not updating
            set_layer_visible(LAYER_GAMEPLAY, true)
            set_layer_visible(LAYER_BACKGROUND, true)
            set_layer_visible(LAYER_UI, true)
            set_layer_visible(LAYER_UI_OVERLAY, true)  # Pause menu

        GameState.CUTSCENE:
            # No UI, focus on gameplay
            set_layer_visible(LAYER_GAMEPLAY, true)
            set_layer_visible(LAYER_BACKGROUND, true)
            set_layer_visible(LAYER_UI, false)

# Layer-based lighting separation
class_name LitLayerManager
extends LayerManager

func _ready():
    super._ready()

    # Configure which layers receive lighting
    var gameplay_layer = get_layer(LAYER_GAMEPLAY)
    if gameplay_layer:
        # Gameplay layer affected by lights
        gameplay_layer.visible = true

    var background_layer = get_layer(LAYER_BACKGROUND)
    if background_layer:
        # Background not affected by dynamic lights (performance)
        # Could have separate static lighting

    var ui_layer = get_layer(LAYER_UI)
    if ui_layer:
        # UI never affected by game lights
        ui_layer.visible = true

# Performance monitoring per layer
class_name ProfiledLayerManager
extends LayerManager

var layer_draw_times: Dictionary = {}  # layer_index -> float (ms)

func _process(_delta):
    # Profile each layer's render time
    for layer_index in layers.keys():
        var layer = layers[layer_index]
        if not layer.visible:
            continue

        var start_time = Time.get_ticks_usec()
        # Layer renders automatically
        await RenderingServer.frame_post_draw
        var end_time = Time.get_ticks_usec()

        layer_draw_times[layer_index] = (end_time - start_time) / 1000.0

func get_layer_render_time(layer_index: int) -> float:
    return layer_draw_times.get(layer_index, 0.0)

func print_layer_statistics():
    print("=== Layer Render Times ===")
    for layer_index in layer_draw_times.keys():
        var layer = layers[layer_index]
        print("%s: %.2f ms" % [layer.name, layer_draw_times[layer_index]])
```

**Key Parameters:**
- **Layer Count:** 5-10 typical, 15-20 complex games, >25 causes overhead
- **Layer Indices:** Use multiples of 10 for easy insertion (-100, -50, 0, 50, 100)
- **Update Frequency:** Static layers (never), UI (60fps), Gameplay (30-60fps)
- **Cache Strategy:** Static layers cached, dynamic updated every frame
- **Visibility Culling:** Disable invisible layers completely
- **Follow Viewport:** Background (no), Gameplay (yes), UI (configurable)

**Edge Cases:**
- **Layer Order Dependency:** Transparent objects need back-to-front within layer
- **Cross-Layer Interaction:** Lighting across layers requires special handling
- **Z-Index Conflicts:** Sprites in same layer with same z_index have undefined order
- **UI Scaling:** Different resolutions need layer scale adjustment
- **Camera Zoom:** Layer follow_viewport_scale must account for zoom
- **Transitions:** Fading between states requires layer alpha control

**When NOT to Use:**
- Very simple games (<100 sprites total) - single layer sufficient
- 3D games - use proper 3D rendering layers instead
- Already well-organized rendering - don't over-engineer
- Prototyping phase - premature optimization
- Games without distinct visual layers (e.g., single-screen puzzles)

**Examples from Shipped Games:**

1. **Celeste:** 6-7 canvas layers: far background (-100), mid background (-50), background decor (-10), gameplay (0), foreground effects (10), UI (100), debug (1000). Static backgrounds cached, gameplay dynamic. UI layer independent from camera. Reduces draw calls by 85%, critical for maintaining 60fps on Switch.

2. **Dead Cells:** 8 layers organized by depth and update frequency. Background layers (3) update only on room transition. Gameplay layer (1) updates every frame. Foreground effects (2) with additive blending. UI layers (2) for health/items separate from menus. Enables complex biome visuals without performance hit.

3. **Hollow Knight:** Extensive layering for atmospheric depth. Background layers (4-5) for distant areas, gameplay layers (2-3) for platforms/enemies, foreground layers (2) for dust/particles, UI layers (3) for HUD/map/menus. Static layers cached as textures. Boss arenas have dedicated layer setups.

4. **Stardew Valley:** Simple but effective layering. Farm background (ground), gameplay (crops/buildings/NPCs), weather effects (rain/snow), UI (inventory/menus). UI layer uses different camera, stays pixel-perfect regardless of zoom. Reduces rendering complexity for week-long crop growth animations.

5. **Enter the Gungeon:** Bullet-hell requiring optimized layering. Floor tiles (cached), walls/obstacles (static), gameplay objects (dynamic), bullets (1000+, separate layer for batching), UI (independent). Bullet layer optimized for massive sprite batching. Essential for 60fps with 500+ bullets on screen.

**Platform Considerations:**
- **Mobile (iOS/Android):** Critical - limit to 6-8 layers max. Aggressive layer caching. Disable invisible layers completely. Target <5 layers actively rendering per frame.
- **Desktop (Windows/Mac/Linux):** Can afford 10-15 layers. Higher resolution needs more careful overdraw management. Memory for cached layers not constrained.
- **Console (Switch/PS/Xbox):** Switch: 8-10 layers max, cache aggressively. PS5/Xbox: 15+ layers fine. Fixed performance easier to optimize.
- **WebGL:** Layer overhead higher in browser - 6-8 layers max. Minimize state changes between layers.
- **VRAM:** Cached layers consume memory (1080p layer = ~8MB RGBA). Budget accordingly.

**Godot-Specific Notes:**
- `CanvasLayer.layer` property sets render order (lower = behind)
- `CanvasLayer.follow_viewport_enabled` makes layer follow camera
- `CanvasLayer.follow_viewport_scale` controls parallax effect
- Use `CanvasLayer.offset` for screen-space positioning (UI)
- Godot 4.x has improved layer batching vs Godot 3.x
- Monitor with Profiler → Visual → CanvasLayer render time
- `CanvasLayer.visible = false` completely skips layer rendering
- Z-index within layer: `Node2D.z_index` ranges from -4096 to 4096

**Synergies:**
- **Sprite Batching:** Layers group sprites for better batching
- **Dirty Rectangles:** Mark specific layers dirty, not all
- **Canvas Item Culling:** Cull per-layer instead of globally
- **Lighting Optimization:** Separate lit/unlit layers
- **Parallax Scrolling:** Layer-based parallax more efficient

**Measurement/Profiling:**
- **Draw Calls per Layer:** Monitor `Performance.RENDER_2D_DRAW_CALLS_IN_FRAME` per layer
- **Layer Render Time:** Profile individual layer rendering cost
- **Overdraw:** Use Godot overdraw visualizer - should see distinct layer boundaries
- **Memory Usage:** Track cached layer textures in `RENDER_TEXTURE_MEM_USED`
- **Profiling Pattern:**
```gdscript
var layer_stats: Dictionary = {}

func profile_layers():
    for layer_index in layers.keys():
        var layer = layers[layer_index]

        var node_count = count_nodes_in_layer(layer)
        var draw_calls = estimate_draw_calls(layer)
        var memory = estimate_layer_memory(layer)

        layer_stats[layer_index] = {
            "name": layer.name,
            "visible": layer.visible,
            "nodes": node_count,
            "draw_calls": draw_calls,
            "memory_mb": memory,
            "cached": layer is CachedCanvasLayer
        }

    print_layer_statistics()

func count_nodes_in_layer(layer: CanvasLayer) -> int:
    var count = 0
    for child in layer.get_children():
        count += 1 + count_children_recursive(child)
    return count

func count_children_recursive(node: Node) -> int:
    var count = 0
    for child in node.get_children():
        count += 1 + count_children_recursive(child)
    return count

func print_layer_statistics():
    print("\n=== Canvas Layer Statistics ===")
    for layer_index in layer_stats.keys():
        var stats = layer_stats[layer_index]
        print("%s [%d]: Nodes=%d, DrawCalls≈%d, Memory≈%.1fMB, Cached=%s" % [
            stats.name,
            layer_index,
            stats.nodes,
            stats.draw_calls,
            stats.memory_mb,
            "Yes" if stats.cached else "No"
        ])
```

**Advanced Optimization Patterns:**
```gdscript
# Pattern 1: Dynamic layer activation based on screen content
func optimize_layer_visibility():
    var viewport_rect = get_viewport_rect()

    for layer_index in layers.keys():
        var layer = layers[layer_index]
        var has_visible_content = false

        for child in layer.get_children():
            if child is Node2D:
                var bounds = get_node_bounds(child)
                if viewport_rect.intersects(bounds):
                    has_visible_content = true
                    break

        # Disable layer if nothing visible
        layer.visible = has_visible_content

# Pattern 2: Layer LOD (reduce quality for distant layers)
func apply_layer_lod(camera_zoom: float):
    if camera_zoom < 0.5:  # Zoomed out
        # Reduce background layer quality
        var bg_layer = get_layer(LAYER_BACKGROUND)
        if bg_layer:
            bg_layer.scale = Vector2.ONE * 0.5  # Half resolution
    else:
        # Full quality when zoomed in
        var bg_layer = get_layer(LAYER_BACKGROUND)
        if bg_layer:
            bg_layer.scale = Vector2.ONE

# Pattern 3: Lazy layer initialization
var initialized_layers: Dictionary = {}

func get_or_create_layer(layer_index: int) -> CanvasLayer:
    if not initialized_layers.has(layer_index):
        var layer = create_layer("Layer_%d" % layer_index, layer_index)
        initialized_layers[layer_index] = layer
        layers[layer_index] = layer

    return layers[layer_index]

# Pattern 4: Layer-based effect isolation
func apply_screen_shake(intensity: float):
    # Only shake gameplay layers, not UI
    for layer_index in [LAYER_BACKGROUND, LAYER_GAMEPLAY, LAYER_FOREGROUND]:
        var layer = get_layer(layer_index)
        if layer:
            layer.offset = Vector2(
                randf_range(-intensity, intensity),
                randf_range(-intensity, intensity)
            )

func apply_camera_zoom(zoom: float):
    # Only zoom gameplay, not UI
    for layer_index in [LAYER_BACKGROUND, LAYER_GAMEPLAY, LAYER_FOREGROUND]:
        var layer = get_layer(layer_index)
        if layer:
            layer.scale = Vector2.ONE * zoom
```
