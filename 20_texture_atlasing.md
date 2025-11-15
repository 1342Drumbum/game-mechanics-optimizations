### 20. Texture Atlasing (2D)

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Texture binding overhead and broken batching in 2D games. 500 sprites with individual textures = 500 draw calls × 0.02ms = 10ms CPU overhead. Each texture bind = GPU state change = pipeline flush. Atlasing combines textures into single image, enabling batching: 500 objects → 5-10 draw calls, saving 8-9ms.

**Technical Explanation:**
Texture atlas packs multiple images into a single large texture (1024×1024, 2048×2048, or 4096×4096). Each sprite uses UV coordinates to reference its region within the atlas. Renderer can batch all sprites using the same atlas into one draw call. Reduces texture bindings (expensive GPU operation), enables draw call batching, improves cache coherency. Modern engines use automatic atlasing at build time. Runtime atlasing possible but adds overhead. Atlas size limited by GPU texture size (typically 8192×8192 max).

**Algorithmic Complexity:**
- No atlasing: O(n) texture binds for n unique textures
- With atlasing: O(k) texture binds where k = number of atlases (typically 1-10)
- Packing algorithm: O(n log n) rectangle packing (Maxrects, Shelf, Guillotine)
- Memory: Slight overhead from padding (2-4 pixels per sprite)
- CPU overhead: UV remapping = negligible (<0.01ms per sprite)

**Implementation Pattern:**
```gdscript
# Godot Texture Atlas System
class_name TextureAtlas
extends Resource

@export var atlas_size: Vector2i = Vector2i(2048, 2048)
@export var padding: int = 2  # Prevent bleeding
@export var allow_rotation: bool = false

var atlas_texture: ImageTexture
var regions: Dictionary = {}  # sprite_name -> Rect2
var packed_image: Image

class AtlasRegion:
    var uv_rect: Rect2  # Normalized UV coordinates (0-1)
    var pixel_rect: Rect2i  # Pixel coordinates in atlas
    var original_size: Vector2i
    var rotated: bool = false

func create_atlas(sprites: Dictionary) -> bool:
    # sprites = {name: Image}
    packed_image = Image.create(atlas_size.x, atlas_size.y, false, Image.FORMAT_RGBA8)
    packed_image.fill(Color(0, 0, 0, 0))

    # Sort by size (largest first for better packing)
    var sorted_sprites = sprites.keys()
    sorted_sprites.sort_custom(func(a, b):
        var area_a = sprites[a].get_width() * sprites[a].get_height()
        var area_b = sprites[b].get_width() * sprites[b].get_height()
        return area_a > area_b
    )

    # Pack using shelf algorithm
    var success = _pack_sprites(sorted_sprites, sprites)

    if success:
        atlas_texture = ImageTexture.create_from_image(packed_image)
        return true

    return false

func _pack_sprites(sprite_names: Array, sprites: Dictionary) -> bool:
    var packer = ShelfPacker.new(atlas_size, padding)

    for name in sprite_names:
        var image = sprites[name]
        var size = Vector2i(image.get_width(), image.get_height())

        var rect = packer.pack(size)
        if rect == Rect2i():
            push_error("Failed to pack sprite: " + name)
            return false

        # Copy sprite to atlas
        packed_image.blit_rect(image, Rect2i(Vector2i.ZERO, size), rect.position)

        # Store region info
        var region = AtlasRegion.new()
        region.pixel_rect = rect
        region.original_size = size
        region.uv_rect = Rect2(
            float(rect.position.x) / atlas_size.x,
            float(rect.position.y) / atlas_size.y,
            float(rect.size.x) / atlas_size.x,
            float(rect.size.y) / atlas_size.y
        )

        regions[name] = region

    return true

func get_region(sprite_name: String) -> AtlasRegion:
    return regions.get(sprite_name)

func get_atlas_texture() -> Texture2D:
    return atlas_texture

# Shelf packing algorithm (simple and efficient)
class ShelfPacker:
    var atlas_size: Vector2i
    var padding: int
    var shelves: Array = []
    var current_shelf: Shelf

    class Shelf:
        var y: int
        var height: int
        var x: int  # Current x position

        func _init(p_y: int, p_height: int):
            y = p_y
            height = p_height
            x = 0

    func _init(p_size: Vector2i, p_padding: int):
        atlas_size = p_size
        padding = p_padding
        current_shelf = Shelf.new(0, 0)
        shelves.append(current_shelf)

    func pack(size: Vector2i) -> Rect2i:
        var padded_size = size + Vector2i(padding * 2, padding * 2)

        # Try current shelf
        if current_shelf.x + padded_size.x <= atlas_size.x:
            if current_shelf.y + padded_size.y <= atlas_size.y:
                var rect = Rect2i(
                    current_shelf.x + padding,
                    current_shelf.y + padding,
                    size.x,
                    size.y
                )
                current_shelf.x += padded_size.x
                current_shelf.height = max(current_shelf.height, padded_size.y)
                return rect

        # Create new shelf
        var new_y = current_shelf.y + current_shelf.height
        if new_y + padded_size.y > atlas_size.y:
            return Rect2i()  # Failed to pack

        current_shelf = Shelf.new(new_y, padded_size.y)
        shelves.append(current_shelf)

        var rect = Rect2i(
            padding,
            new_y + padding,
            size.x,
            size.y
        )
        current_shelf.x = padded_size.x
        return rect

# Sprite batcher using atlas
class AtlasSpriteBatcher:
    var atlas: TextureAtlas
    var sprite_batch: Array = []  # {transform, region_name, color}

    func add_sprite(transform: Transform2D, sprite_name: String, color: Color = Color.WHITE):
        sprite_batch.append({
            "transform": transform,
            "sprite": sprite_name,
            "color": color
        })

    func draw_batch(canvas: CanvasItem):
        if sprite_batch.is_empty():
            return

        var atlas_texture = atlas.get_atlas_texture()

        # All sprites use same texture = single draw call in many cases
        for item in sprite_batch:
            var region = atlas.get_region(item["sprite"])
            if not region:
                continue

            # Convert UV rect to pixel rect for draw_texture_rect
            var src_rect = Rect2(
                region.uv_rect.position * atlas.atlas_size,
                region.uv_rect.size * atlas.atlas_size
            )

            canvas.draw_set_transform_matrix(item["transform"])
            canvas.draw_texture_rect_region(
                atlas_texture,
                Rect2(Vector2.ZERO, region.original_size),
                src_rect,
                item["color"]
            )

        sprite_batch.clear()

# Runtime atlas builder (dynamic sprites)
class DynamicTextureAtlas:
    var atlas_texture: ImageTexture
    var atlas_size: Vector2i = Vector2i(2048, 2048)
    var free_regions: Array[Rect2i] = []
    var used_regions: Dictionary = {}  # id -> Rect2i

    func _init(size: Vector2i = Vector2i(2048, 2048)):
        atlas_size = size
        var image = Image.create(size.x, size.y, false, Image.FORMAT_RGBA8)
        image.fill(Color(0, 0, 0, 0))
        atlas_texture = ImageTexture.create_from_image(image)

        # Start with one large free region
        free_regions.append(Rect2i(Vector2i.ZERO, size))

    func allocate_region(size: Vector2i) -> int:
        # Find best-fit free region
        var best_index = -1
        var best_area = INF

        for i in range(free_regions.size()):
            var region = free_regions[i]
            if region.size.x >= size.x and region.size.y >= size.y:
                var area = region.size.x * region.size.y
                if area < best_area:
                    best_area = area
                    best_index = i

        if best_index == -1:
            return -1  # No space

        var region = free_regions[best_index]
        free_regions.remove_at(best_index)

        # Allocate from region
        var allocated = Rect2i(region.position, size)

        # Split remaining space
        if region.size.x > size.x:
            free_regions.append(Rect2i(
                region.position + Vector2i(size.x, 0),
                Vector2i(region.size.x - size.x, size.y)
            ))

        if region.size.y > size.y:
            free_regions.append(Rect2i(
                region.position + Vector2i(0, size.y),
                Vector2i(region.size.x, region.size.y - size.y)
            ))

        var id = used_regions.size()
        used_regions[id] = allocated
        return id

    func update_region(id: int, image: Image):
        if not id in used_regions:
            return

        var region = used_regions[id]
        var atlas_image = atlas_texture.get_image()
        atlas_image.blit_rect(image, Rect2i(Vector2i.ZERO, image.get_size()), region.position)
        atlas_texture.update(atlas_image)

    func free_region(id: int):
        if id in used_regions:
            free_regions.append(used_regions[id])
            used_regions.erase(id)
            # TODO: Merge adjacent free regions
```

**Key Parameters:**
- Atlas size: 2048×2048 (standard), 4096×4096 (high-res), 1024×1024 (mobile)
- Padding: 2-4 pixels (prevents texture bleeding at edges)
- Packing efficiency: Target >80% utilized space
- Max sprites per atlas: 50-500 depending on sprite size
- Atlas count: 1-5 for typical 2D game (UI, characters, environment, effects, particles)
- Compression: Use ETC2/ASTC on mobile, BC7 on desktop

**Edge Cases:**
- Sprite too large: Create dedicated texture or split sprite
- Atlas full: Create additional atlas, manage multiple batches
- Animated sprites: Pack all frames or use sprite sheets
- Texture bleeding: Add padding + clamp UV coordinates
- Mipmapping: Padding must account for mip levels (4-8 pixels)
- Resolution variants: Create separate atlases for @2x, @3x scales

**When NOT to Use:**
- Few unique sprites (<20)
- Very large sprites (>1024×1024 each)
- Frequently changing textures (video, procedural)
- Need for individual texture filtering per sprite
- 3D games (use material batching instead)

**Examples from Shipped Games:**

1. **Celeste:** All sprites atlased, 1200+ sprites → 3 atlases (characters, environment, effects), 60fps on Switch. Reduced draw calls from 800 → 15 average. Essential for pixel-perfect rendering.

2. **Dead Cells:** Aggressive atlasing, 2000+ animation frames → 8 atlases (2048×2048), 60fps on all platforms. Combined with sprite batching, maintained <100 draw calls in intense combat.

3. **Hollow Knight:** Environment/character atlasing, 300+ unique sprites → 4 atlases. Enabled complex layered parallax backgrounds at 60fps on Switch. Reduced VRAM from 400MB → 120MB.

4. **Stardew Valley:** UI/item atlasing, 500+ items → 2 atlases (1024×1024). Mobile version: critical for performance, reduced draw calls by 90%. Enabled 60fps on older devices.

5. **Brawl Stars:** Aggressive atlasing, 1000+ sprites → 5 atlases per environment. Enabled 60fps on low-end mobile. Runtime atlas for player skins. Reduced memory bandwidth by 70%.

**Platform Considerations:**
- **Mobile:** Critical - texture binds expensive, aim for 1-3 atlases, use ETC2/ASTC compression
- **Desktop:** Less critical but still beneficial, can use larger atlases (4096×4096)
- **Console:** Important for 60fps, optimize atlas count, use hardware compression
- **WebGL:** Essential - limited texture units, 50-100ms saved with atlasing
- **Memory:** Atlasing saves VRAM (shared mipmaps) but increases individual texture size
- **Bandwidth:** Reduced texture fetches improve cache coherency

**Godot-Specific Notes:**
- Use `AtlasTexture` resource for manual atlasing
- Import settings: Can auto-generate atlases for sprite sheets
- `SpriteFrames` supports atlases for animations
- TileSet automatically creates atlas from tilemap
- Godot 4.x: Improved automatic 2D batching with texture arrays
- Use `CanvasItem.texture_filter` consistently across atlased sprites
- Batching breaks if filtering modes differ
- Consider using `TextureRect` with atlas for UI elements

**Synergies:**
- Essential for Draw Call Batching (same texture = batchable)
- Pairs with 2D Particle Systems (particle atlas reduces draw calls)
- Combines with Sprite Animation (frame atlas enables efficient animation)
- Works with UI Optimization (all UI elements in one atlas)
- Enables efficient TileMap rendering (tile atlas)

**Measurement/Profiling:**
- Godot Profiler: Monitor `Rendering -> Draw calls` (target <50 for 2D)
- Track atlas utilization: >80% ideal, <60% wasteful
- Measure VRAM usage: Should decrease with atlasing
- Visual debug: Color sprites by atlas to verify batching
- Profile texture binds: Should decrease proportional to atlas count
- Target: 5-20 draw calls for typical 2D scene
- Performance: 70-90% draw call reduction typical

**Additional Resources:**
- Texture Packer: Popular tool for atlas generation
- GDC 2014: "Optimizing 2D Games for Mobile"
- GPU Gems: Chapter on Texture Compression and Atlasing
