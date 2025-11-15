### 41. 2D Sprite Batching

**Category:** Performance - 2D Rendering Optimization

**Problem It Solves:** Draw call overhead causing GPU bottlenecks. Rendering 1000 sprites individually requires 1000 draw calls at ~50μs each = 50ms GPU time, exceeding 16.67ms frame budget by 3x. Batching reduces to 10-50 draw calls = 0.5-2.5ms, a 95%+ reduction.

**Technical Explanation:**
Sprite batching combines multiple sprites sharing the same texture/material into a single draw call. Instead of sending 1000 separate render commands (each with setup overhead), the GPU receives one command with 1000 sprites worth of vertex data. The renderer sorts sprites by texture/shader, groups consecutive compatible sprites, and uploads combined vertex buffers. Modern engines use dynamic batching (runtime grouping) or static batching (pre-combined meshes).

GPU state changes are expensive: binding textures, setting shaders, updating uniforms. Each draw call requires CPU-GPU synchronization. Batching minimizes state changes by processing all sprites with identical rendering state together. The vertex shader processes all sprites in parallel, maximizing GPU utilization.

**Algorithmic Complexity:**
- Unbatched: O(n) draw calls for n sprites
- Batched: O(k) draw calls where k = unique texture/material combinations
- Best case: 1000 sprites, 1 texture = 1 draw call (99.9% reduction)
- Worst case: 1000 sprites, 1000 textures = 1000 draw calls (no benefit)

**Implementation Pattern:**
```gdscript
# Godot automatic batching - ensure these conditions are met
class_name SpriteBatcher
extends Node2D

# 1. Use same texture atlas for related sprites
const TEXTURE_ATLAS = preload("res://sprites/atlas.png")

# 2. Keep sprites using same material together
var sprite_material: CanvasItemMaterial

func _ready():
    # Create shared material
    sprite_material = CanvasItemMaterial.new()
    sprite_material.blend_mode = CanvasItemMaterial.BLEND_MODE_MIX

    # Apply to all child sprites
    for child in get_children():
        if child is Sprite2D:
            child.texture = TEXTURE_ATLAS
            child.material = sprite_material
            child.region_enabled = true  # Use atlas regions

# Manual batching with MultiMeshInstance2D for static sprites
class_name StaticSpriteBatch
extends MultiMeshInstance2D

func setup_batch(sprite_positions: Array, texture_rects: Array):
    multimesh = MultiMesh.new()
    multimesh.transform_format = MultiMesh.TRANSFORM_2D
    multimesh.instance_count = sprite_positions.size()

    # Set transforms
    for i in sprite_positions.size():
        var t = Transform2D()
        t.origin = sprite_positions[i]
        multimesh.set_instance_transform_2d(i, t)

        # Set custom data for UV coordinates if needed
        if texture_rects.size() > i:
            multimesh.set_instance_custom_data(i,
                Color(texture_rects[i].position.x,
                      texture_rects[i].position.y,
                      texture_rects[i].size.x,
                      texture_rects[i].size.y))

# Dynamic sprite sorter for batching efficiency
class_name SpriteLayerManager
extends Node2D

var layers: Dictionary = {}  # texture -> array of sprites

func add_sprite(sprite: Sprite2D):
    var tex = sprite.texture
    if not layers.has(tex):
        layers[tex] = []
    layers[tex].append(sprite)

    # Z-index based on batch to minimize state changes
    var batch_index = layers.keys().find(tex)
    sprite.z_index = batch_index

func optimize_draw_order():
    # Sort children by texture to maximize batching
    var sorted_children = []
    for texture in layers.keys():
        sorted_children.append_array(layers[texture])

    for i in sorted_children.size():
        move_child(sorted_children[i], i)
```

**Key Parameters:**
- Texture Atlas Size: 2048x2048 or 4096x4096 (balance memory vs batching)
- Max Sprites Per Batch: 10,000-50,000 vertices (40-200KB vertex buffer)
- Material Compatibility: Same blend mode, shader, render priority
- Z-Index Range: Keep batched sprites in consecutive z-order layers
- Update Frequency: Static batches never rebuild, dynamic rebuild when sprite set changes

**Edge Cases:**
- Material Changes: Different blend modes break batches (mix, add, multiply)
- Shader Variations: Custom shaders break batching unless identical
- Texture Swapping: Animated sprites changing textures break batches
- Large Sprite Count: >100k sprites may need hierarchical batching
- Transform Updates: Frequent position changes invalidate static batches
- Depth Sorting: Semi-transparent sprites need back-to-front order

**When NOT to Use:**
- Few sprites (<50 on screen) - overhead exceeds benefit
- Each sprite unique texture - no batching possible
- Heavy per-sprite custom shaders - defeats batching purpose
- Particles with physics - instability from constant updates
- Development/prototyping - premature optimization

**Examples from Shipped Games:**

1. **Celeste:** All level tiles in 2048x2048 atlas, 1000+ sprites batched to 5-10 draw calls. Critical for maintaining 60fps with pixel-perfect rendering. Madeline sprite separate for animations.

2. **Dead Cells:** Enemy sprites in shared atlases by biome, 50-100 enemies on screen batched to 3-5 calls. Used MultiMesh for background decoration (torches, chains), reducing 500 draw calls to 1.

3. **Hollow Knight:** Environmental sprites heavily batched, 2000+ background elements rendered in <10 draw calls. Character and enemy animations kept separate for flexibility, but effects particles batched.

4. **Terraria:** Tile rendering uses mega-batching, 10,000+ visible tiles → 1-2 draw calls per layer. Items and entities use texture packing, 200 items on ground → 15-20 batches by item type.

5. **Stardew Valley:** All crop sprites in single atlas, farm with 1000 crops = 1 draw call for all crops. Animals and tools separate but batched by type. UI elements heavily batched.

**Platform Considerations:**
- **Mobile (iOS/Android):** Critical - draw calls 2-3x more expensive. Target <20 draw calls total. Use ETC2/ASTC texture compression for atlases.
- **Desktop (Windows/Mac/Linux):** Important but less critical. Can handle 50-100 draw calls at 60fps. Larger atlases (4096x4096) acceptable.
- **Console (Switch/PS/Xbox):** Essential - fixed GPU budget. Switch especially sensitive, target <30 draw calls. Pre-bake static geometry.
- **WebGL:** Absolutely critical - draw call overhead massive. Target <15 draw calls. Aggressive texture atlasing required.
- **Memory Trade-off:** Atlasing uses 2-4x more VRAM than individual textures (padding, unused space).

**Godot-Specific Notes:**
- Godot 4.x automatically batches compatible sprites in same CanvasItem layer
- Use `CanvasItem.texture_filter` and `texture_repeat` consistently for batching
- `z_index` and `z_as_relative` affect batch breaking - keep consecutive
- Monitor batching in Godot debugger: Monitors → Rendering → Draw Calls
- `MultiMeshInstance2D` for static sprites gives best batching (10,000+ instances)
- Godot's batching prefers Row-Major order in scene tree
- Use `YSort` carefully - can break batches if sprites overlap different textures

**Synergies:**
- **Texture Atlasing:** Essential prerequisite for effective batching
- **Sprite Animation Compression:** Keep animation frames in same atlas
- **Canvas Layer Management:** Group sprites by render layer for better batching
- **Dirty Rectangles:** Selective updates preserve batch validity
- **2D Lighting:** Separate light-affected sprites into consistent batches

**Measurement/Profiling:**
- **Godot Monitor:** `Performance.RENDER_2D_DRAW_CALLS_IN_FRAME` - target <30
- **Frame Time:** Monitor `Performance.TIME_RENDER` - batching should show 40-60% reduction
- **Visual Profiler:** Check "Render 2D" section for draw call breakdown
- **Debug Draw:** Use `--verbose` flag to see batch formation logs
- **Baseline Test:** Disable batching, measure draw calls, re-enable and compare
- **Target Metrics:**
  - Desktop: <0.5ms render time for 1000 sprites
  - Mobile: <1.5ms render time for 500 sprites
  - Draw call reduction: 90%+ for atlas-based games

**Common Batching Patterns:**
```gdscript
# Pattern 1: Layer-based batching
# Group sprites by visual layer
var background_layer = CanvasLayer.new()  # z_index = -10
var gameplay_layer = CanvasLayer.new()    # z_index = 0
var ui_layer = CanvasLayer.new()          # z_index = 10

# Pattern 2: Material-based grouping
var opaque_material = CanvasItemMaterial.new()
opaque_material.blend_mode = CanvasItemMaterial.BLEND_MODE_MIX

var additive_material = CanvasItemMaterial.new()
additive_material.blend_mode = CanvasItemMaterial.BLEND_MODE_ADD

# Separate sprites by material to maintain batches

# Pattern 3: Atlas region sharing
var atlas_texture = preload("res://atlas.png")
for sprite in sprites:
    sprite.texture = atlas_texture
    sprite.region_enabled = true
    sprite.region_rect = calculate_region(sprite.type)
```

**Advanced Optimization:**
Use sprite sheets with power-of-2 dimensions, minimize alpha blending, disable `light_mode` when unnecessary, keep sprite count per batch under vertex buffer limits (65k vertices typical), and profile with different atlas configurations to find sweet spot between texture memory and batching efficiency.
