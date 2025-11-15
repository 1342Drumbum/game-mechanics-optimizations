### 63. Virtual Texturing (Mega-Textures)

**Category:** Performance - Memory Management / Texture Streaming

**Problem It Solves:** Texture memory explosion from high-resolution materials. A large open-world game with 1000 unique 4096² textures (diffuse, normal, roughness, AO = 4 textures per material) requires 1000 × 4 × 64MB = 256GB of texture data. Even with compression (4:1) = 64GB, far exceeding console VRAM (8GB). Loading all textures causes 30+ second load times and frequent streaming hitches. Virtual texturing enables unlimited texture resolution with constant memory usage (512MB-2GB).

**Technical Explanation:**
Virtual texturing treats all textures as one massive virtual texture address space. Only visible texture data is loaded into physical texture cache (similar to virtual memory paging). Implementation: 1) Create page table mapping virtual texture coordinates to physical cache tiles. 2) Render scene, track which tiles are visible via feedback buffer. 3) Stream required tiles from disk to physical cache (128×128 pixel tiles). 4) Update page table. 5) Re-render with high-resolution tiles. Enables 128K×128K textures with only 1-2GB VRAM usage. Indirection overhead: 1 extra texture lookup per sample.

**Algorithmic Complexity:**
- Traditional texturing: O(visible_textures), limited by VRAM (8GB)
- Virtual texturing: O(visible_tiles), constant VRAM (1-2GB regardless of total texture size)
- Page table lookup: O(1) with 2-level indirection
- Streaming bandwidth: 50-200MB/sec from SSD
- Memory: Constant 1-2GB vs potentially hundreds of GB
- Indirection cost: +1-2 texture lookups per fragment (~5-10 GPU cycles)

**Implementation Pattern:**
```gdscript
# Godot implementation
class_name VirtualTexturingSystem
extends Node

# Virtual texture parameters
const PAGE_SIZE = 128  # Physical tile size in pixels
const CACHE_SIZE = 8192  # Physical texture cache size (8192×8192)
const PAGE_TABLE_SIZE = 2048  # Page table resolution

var physical_cache: ImageTexture  # Physical texture containing cached tiles
var page_table: ImageTexture  # Indirection texture mapping virtual→physical
var feedback_buffer: Image  # Records which tiles were accessed

var tile_cache: Dictionary = {}  # Active tiles in physical cache
var streaming_queue: Array = []  # Tiles to load from disk

@export var virtual_texture_size: int = 131072  # 128K×128K virtual texture
@export var max_streaming_per_frame: int = 8  # Tiles to stream per frame

# Virtual texture shader
const VIRTUAL_TEXTURE_SHADER = """
shader_type spatial;

uniform sampler2D physical_cache : hint_default_white;
uniform sampler2D page_table : hint_default_black;
uniform sampler2D feedback_buffer : hint_default_black;

uniform vec2 virtual_texture_size = vec2(131072.0);
uniform vec2 page_table_size = vec2(2048.0);
uniform vec2 physical_cache_size = vec2(8192.0);
uniform float page_size = 128.0;

varying vec2 virtual_uv;

void vertex() {
    virtual_uv = UV;
}

void fragment() {
    // Step 1: Map virtual UV to page table coordinates
    vec2 page_table_uv = virtual_uv;

    // Step 2: Look up page table to find physical tile
    vec4 page_info = texture(page_table, page_table_uv);

    // Page info encoding:
    // RG: physical cache coordinates (tile position in cache)
    // B: mip level
    // A: valid bit (1.0 = tile is loaded, 0.0 = not loaded)

    if (page_info.a < 0.5) {
        // Tile not loaded - write to feedback buffer
        imageStore(feedback_buffer, ivec2(page_table_uv * page_table_size), vec4(1.0));

        // Use low-res fallback
        ALBEDO = vec3(0.5);  // Gray placeholder
        return;
    }

    // Step 3: Calculate position within tile
    vec2 tile_local_uv = fract(virtual_uv * virtual_texture_size / page_size);

    // Step 4: Map to physical cache
    vec2 physical_uv = (page_info.rg * page_size + tile_local_uv * page_size) / physical_cache_size;

    // Step 5: Sample physical cache
    vec4 color = texture(physical_cache, physical_uv);

    ALBEDO = color.rgb;
}
"""

func _ready():
    _initialize_virtual_texturing()

func _initialize_virtual_texturing():
    # Create physical texture cache (8192×8192 RGBA)
    var cache_image = Image.create(CACHE_SIZE, CACHE_SIZE, false, Image.FORMAT_RGBA8)
    physical_cache = ImageTexture.create_from_image(cache_image)

    # Create page table (2048×2048 RGBA16F for precise coordinates)
    var page_table_image = Image.create(PAGE_TABLE_SIZE, PAGE_TABLE_SIZE, false, Image.FORMAT_RGBAH)
    page_table = ImageTexture.create_from_image(page_table_image)

    # Create feedback buffer (track accessed tiles)
    feedback_buffer = Image.create(PAGE_TABLE_SIZE, PAGE_TABLE_SIZE, false, Image.FORMAT_R8)

func _process(_delta):
    # Step 1: Read feedback buffer to find requested tiles
    _analyze_feedback_buffer()

    # Step 2: Stream tiles from disk/memory
    _stream_tiles()

    # Step 3: Update page table with newly loaded tiles
    _update_page_table()

    # Step 4: Clear feedback buffer for next frame
    feedback_buffer.fill(Color(0, 0, 0, 0))

func _analyze_feedback_buffer():
    # Scan feedback buffer to find which tiles were accessed
    streaming_queue.clear()

    for y in range(PAGE_TABLE_SIZE):
        for x in range(PAGE_TABLE_SIZE):
            var pixel = feedback_buffer.get_pixel(x, y)

            if pixel.r > 0.5:  # Tile was requested
                var tile_id = Vector2i(x, y)

                # Check if tile already loaded
                if tile_id not in tile_cache:
                    streaming_queue.append(tile_id)

    # Sort by distance to camera (priority streaming)
    _sort_streaming_queue_by_priority()

func _sort_streaming_queue_by_priority():
    # Sort tiles by importance (distance to camera, mip level, etc.)
    var camera = get_viewport().get_camera_3d()

    if camera == null:
        return

    streaming_queue.sort_custom(func(a, b):
        # Tiles closer to camera center have higher priority
        var center = Vector2(PAGE_TABLE_SIZE, PAGE_TABLE_SIZE) / 2.0
        var dist_a = a.distance_to(center)
        var dist_b = b.distance_to(center)
        return dist_a < dist_b
    )

func _stream_tiles():
    # Stream up to N tiles per frame to avoid hitches
    var tiles_loaded = 0

    while tiles_loaded < max_streaming_per_frame and not streaming_queue.is_empty():
        var tile_id = streaming_queue.pop_front()

        _load_tile_from_disk(tile_id)
        tiles_loaded += 1

func _load_tile_from_disk(tile_id: Vector2i):
    # Load 128×128 tile from disk
    var tile_path = _get_tile_path(tile_id)

    # In production, use async loading
    var tile_image = Image.load_from_file(tile_path)

    if tile_image == null:
        # Generate tile procedurally or use fallback
        tile_image = _generate_fallback_tile(tile_id)

    # Find empty slot in physical cache
    var cache_slot = _allocate_cache_slot()

    # Copy tile to physical cache
    _blit_tile_to_cache(tile_image, cache_slot)

    # Update tile cache registry
    tile_cache[tile_id] = cache_slot

func _allocate_cache_slot() -> Vector2i:
    # Simple LRU cache eviction
    var tiles_per_row = CACHE_SIZE / PAGE_SIZE
    var total_tiles = tiles_per_row * tiles_per_row

    # Find least recently used tile
    if tile_cache.size() >= total_tiles:
        # Evict oldest tile
        var oldest_tile = null
        for tile_id in tile_cache.keys():
            oldest_tile = tile_id
            break

        tile_cache.erase(oldest_tile)

    # Return next available slot
    var slot_index = tile_cache.size()
    var slot_x = slot_index % tiles_per_row
    var slot_y = slot_index / tiles_per_row

    return Vector2i(slot_x, slot_y)

func _blit_tile_to_cache(tile_image: Image, cache_slot: Vector2i):
    # Copy tile to physical cache at specific position
    var cache_image = physical_cache.get_image()

    var dst_x = cache_slot.x * PAGE_SIZE
    var dst_y = cache_slot.y * PAGE_SIZE

    cache_image.blit_rect(tile_image, Rect2i(0, 0, PAGE_SIZE, PAGE_SIZE), Vector2i(dst_x, dst_y))

    # Update texture
    physical_cache.update(cache_image)

func _update_page_table():
    # Update page table with newly loaded tiles
    var page_table_image = page_table.get_image()

    for tile_id in tile_cache.keys():
        var cache_slot = tile_cache[tile_id]

        # Encode cache position in page table
        var page_info = Color(
            cache_slot.x / float(CACHE_SIZE / PAGE_SIZE),  # R: cache X
            cache_slot.y / float(CACHE_SIZE / PAGE_SIZE),  # G: cache Y
            0.0,  # B: mip level
            1.0   # A: valid bit
        )

        page_table_image.set_pixel(tile_id.x, tile_id.y, page_info)

    page_table.update(page_table_image)

func _get_tile_path(tile_id: Vector2i) -> String:
    # Map tile ID to file path
    # In production, use hierarchical directory structure
    return "res://virtual_texture/tiles/tile_%d_%d.png" % [tile_id.x, tile_id.y]

func _generate_fallback_tile(tile_id: Vector2i) -> Image:
    # Generate low-res procedural tile as fallback
    var image = Image.create(PAGE_SIZE, PAGE_SIZE, false, Image.FORMAT_RGBA8)

    # Checkerboard pattern for debugging
    var color = Color.WHITE if (tile_id.x + tile_id.y) % 2 == 0 else Color.BLACK
    image.fill(color)

    return image

# Tile pre-caching based on camera movement prediction
class_name TilePrecacher

var camera_velocity: Vector3
var predicted_tiles: Array[Vector2i] = []

func predict_and_precache(camera: Camera3D, vt_system: VirtualTexturingSystem):
    # Calculate camera velocity
    var current_pos = camera.global_position
    camera_velocity = (current_pos - camera_velocity) # Previous position tracking

    # Predict future camera position
    var predicted_pos = current_pos + camera_velocity * 0.5  # 0.5 sec ahead

    # Find tiles visible from predicted position
    predicted_tiles = _find_visible_tiles(predicted_pos)

    # Queue for streaming
    for tile_id in predicted_tiles:
        if tile_id not in vt_system.tile_cache:
            vt_system.streaming_queue.append(tile_id)

func _find_visible_tiles(position: Vector3) -> Array[Vector2i]:
    # Calculate which tiles are visible from given position
    # Simplified - in production, use proper frustum calculation
    var visible = []

    # Implementation based on camera frustum and texture projection

    return visible

# Mip-mapping support for virtual textures
class_name VirtualTextureMipMap

var mip_levels: int = 8
var mip_page_tables: Array[ImageTexture] = []

func _initialize_mipmaps():
    # Create page tables for each mip level
    var size = PAGE_TABLE_SIZE

    for mip in range(mip_levels):
        var mip_table = Image.create(size, size, false, Image.FORMAT_RGBAH)
        mip_page_tables.append(ImageTexture.create_from_image(mip_table))

        size /= 2
        size = max(size, 1)

func get_mip_level_for_distance(distance: float) -> int:
    # Calculate appropriate mip level based on distance
    var mip = log(distance) / log(2.0)
    return clampi(int(mip), 0, mip_levels - 1)
```

**Key Parameters:**
- Page/tile size: 128×128 to 256×256 pixels (128 is typical)
- Physical cache size: 4096² to 16384² (8192² is sweet spot)
- Page table size: 1024² to 4096² (2048² typical)
- Virtual texture size: 32K×32K to 128K×128K
- Streaming rate: 4-16 tiles per frame (depends on disk speed)
- Cache size: 1-2GB VRAM for physical cache
- Indirection levels: 2-level (page table + physical cache)

**Edge Cases:**
- Tile thrashing: Camera moving too fast, constant cache eviction
- Seams between tiles: Use 1-2 pixel borders with texture padding
- Mip-mapping: Requires separate page tables for each mip level
- Compression: Use per-tile compression (BC7/ASTC) for disk storage
- Feedback buffer latency: 1-frame delay before tiles load (use low-res fallback)
- SSD speed: HDD too slow for real-time streaming, SSD mandatory

**When NOT to Use:**
- Small games with limited texture variety (<2GB total textures)
- Indoor games with limited view distance
- Stylized games with low-resolution textures
- Platforms without shader support for indirection
- When VRAM is abundant (>16GB) and textures fit in memory

**Examples from Shipped Games:**

1. **Rage (id Software):** First major game using mega-textures. Entire game world (16GB compressed) streamed from 8192² physical cache. Enabled unique texturing for entire outdoor environment. Required SSD for optimal performance.

2. **Doom (2016):** Refined virtual texturing system. 128K×128K virtual textures with 1.5GB physical cache. Streamed textures at 60fps with <1 frame latency. Enabled photogrammetry-quality textures across entire game.

3. **The Surge:** Used virtual texturing for large industrial environments. Reduced texture memory from 12GB to 1.2GB while maintaining high-resolution detail. Critical for fitting in console memory budget.

4. **Call of Duty: Ghosts:** Implemented virtual texturing for campaign levels. Enabled 4K texture resolution on all surfaces with 2GB VRAM usage. Streamed from HDD with predictive caching.

5. **Halo Infinite:** Procedural virtual texturing for open-world terrain. 256K×256K virtual texture generated procedurally and cached. Enabled massive draw distances with consistent texture quality.

**Platform Considerations:**
- **PC:** Full support, benefits from SSD, 2-4GB VRAM cache
- **Console (PS5/XSX):** SSD mandatory, excellent performance with 2GB cache
- **Last-gen (PS4/XBO):** Limited by HDD speed, requires aggressive prediction
- **Mobile:** Limited shader support, smaller cache (512MB-1GB)
- **Switch:** Possible but limited by 32GB/s memory bandwidth
- **VR:** Excellent for VR - constant memory usage regardless of world size

**Godot-Specific Notes:**
- Godot doesn't have built-in virtual texturing support
- Implement using custom shaders and RenderingDevice for compute-based updates
- Use `Image` and `ImageTexture` for runtime texture updates
- Async loading: Use `ResourceLoader.load_threaded_request()` for tile streaming
- Monitor: Track VRAM usage via `RenderingServer.get_rendering_info()`
- Consider texture arrays for physical cache (reduces bind changes)
- Profile: Use GPU profilers to measure indirection overhead

**Synergies:**
- **Texture Streaming:** Virtual texturing is advanced form of streaming
- **LOD Systems:** Combine with mesh LOD for complete detail management
- **Occlusion Culling:** Don't stream tiles for occluded geometry
- **Procedural Generation:** Generate tiles procedurally instead of loading from disk
- **Compression:** Per-tile BC7/ASTC compression reduces disk space

**Measurement/Profiling:**
- **Memory usage:** Should be constant (1-2GB) regardless of world size
- **Streaming rate:** Monitor tiles loaded per frame (target: 8-16)
- **Cache hit rate:** Should be >95% (low = thrashing)
- **Indirection overhead:** Measure fragment shader cost (+5-10% typical)
- **Load times:** Eliminated or minimal (<1 sec) vs traditional (30+ sec)
- **Visual quality:** Compare to full-resolution textures (should be identical)
- **Profile tools:** GPU profilers, custom streaming statistics

---
