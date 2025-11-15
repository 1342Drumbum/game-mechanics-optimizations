### 42. Tilemap Chunking

**Category:** Performance - 2D Rendering & Memory Management

**Problem It Solves:** Rendering massive tilemaps causes overdraw and memory waste. A 10,000x10,000 tile world = 100M tiles. At 16 bytes per tile = 1.6GB memory. Rendering all tiles = 100M draw operations = 5000ms at 50ns each, destroying performance. Chunking reduces visible tiles by 99%+, rendering only ~2000 tiles in viewport.

**Technical Explanation:**
Divide large tilemap into fixed-size chunks (typically 16x16 to 64x64 tiles). Load/render only chunks intersecting the camera viewport. Each chunk is a separate TileMap node or mesh. Use spatial hash to quickly find visible chunks. Unload distant chunks to free memory. Stream chunks from disk as player moves. GPU culls entire chunks outside viewport in single operation vs per-tile culling.

Chunking exploits spatial locality - players only interact with nearby tiles. By subdividing the world, we convert O(n) operations on entire map to O(k) operations on visible chunks where k << n. Memory usage becomes proportional to viewport size rather than world size.

**Algorithmic Complexity:**
- Full Map: O(n) tiles rendered, O(n) memory where n = total tiles
- Chunked: O(c × k) visible tiles where c = visible chunks, k = tiles per chunk
- Typical: 100M tiles → 2000 visible tiles (99.998% reduction)
- Chunk Lookup: O(1) via spatial hash or grid array

**Implementation Pattern:**
```gdscript
# TileMap Chunk Manager
class_name ChunkManager
extends Node2D

const CHUNK_SIZE = 32  # 32x32 tiles per chunk
const TILE_SIZE = 16   # 16 pixels per tile
const CHUNK_PIXEL_SIZE = CHUNK_SIZE * TILE_SIZE  # 512 pixels

const LOAD_DISTANCE = 2    # Load chunks 2 chunks away from camera
const UNLOAD_DISTANCE = 3  # Unload chunks 3 chunks away

var active_chunks: Dictionary = {}  # Vector2i -> TileMapChunk
var chunk_pool: Array[TileMap] = []
var world_data: Dictionary = {}  # Persistent chunk data

class TileMapChunk:
    var tilemap: TileMap
    var chunk_pos: Vector2i
    var is_dirty: bool = false

func _ready():
    # Pre-pool some TileMap nodes
    for i in range(20):
        var tm = TileMap.new()
        tm.tile_set = load("res://tilesets/main_tileset.tres")
        tm.cell_quadrant_size = 16  # Optimize for batching
        chunk_pool.append(tm)

func _process(_delta):
    var camera_pos = get_viewport().get_camera_2d().get_screen_center_position()
    var camera_chunk = world_to_chunk(camera_pos)

    # Load visible chunks
    for x in range(-LOAD_DISTANCE, LOAD_DISTANCE + 1):
        for y in range(-LOAD_DISTANCE, LOAD_DISTANCE + 1):
            var chunk_coord = camera_chunk + Vector2i(x, y)
            if not active_chunks.has(chunk_coord):
                load_chunk(chunk_coord)

    # Unload distant chunks
    var chunks_to_unload = []
    for chunk_coord in active_chunks.keys():
        var dist = chunk_coord - camera_chunk
        if abs(dist.x) > UNLOAD_DISTANCE or abs(dist.y) > UNLOAD_DISTANCE:
            chunks_to_unload.append(chunk_coord)

    for coord in chunks_to_unload:
        unload_chunk(coord)

func world_to_chunk(world_pos: Vector2) -> Vector2i:
    return Vector2i(
        int(floor(world_pos.x / CHUNK_PIXEL_SIZE)),
        int(floor(world_pos.y / CHUNK_PIXEL_SIZE))
    )

func load_chunk(chunk_coord: Vector2i):
    var chunk = TileMapChunk.new()

    # Get or create TileMap
    if chunk_pool.is_empty():
        chunk.tilemap = TileMap.new()
        chunk.tilemap.tile_set = load("res://tilesets/main_tileset.tres")
    else:
        chunk.tilemap = chunk_pool.pop_back()

    chunk.chunk_pos = chunk_coord
    chunk.tilemap.position = Vector2(
        chunk_coord.x * CHUNK_PIXEL_SIZE,
        chunk_coord.y * CHUNK_PIXEL_SIZE
    )

    add_child(chunk.tilemap)

    # Load tile data from storage
    if world_data.has(chunk_coord):
        populate_chunk(chunk, world_data[chunk_coord])
    else:
        generate_chunk(chunk)  # Procedural generation

    active_chunks[chunk_coord] = chunk

func unload_chunk(chunk_coord: Vector2i):
    var chunk = active_chunks[chunk_coord]

    # Save if modified
    if chunk.is_dirty:
        save_chunk_data(chunk)

    # Return to pool
    remove_child(chunk.tilemap)
    chunk.tilemap.clear()
    chunk_pool.append(chunk.tilemap)

    active_chunks.erase(chunk_coord)

func populate_chunk(chunk: TileMapChunk, tile_data: Array):
    # tile_data is array of {pos: Vector2i, tile_id: int, atlas_coord: Vector2i}
    for tile_info in tile_data:
        chunk.tilemap.set_cell(
            0,  # layer
            tile_info.pos,
            tile_info.tile_id,
            tile_info.atlas_coord
        )

func generate_chunk(chunk: TileMapChunk):
    # Procedural generation example
    var noise = FastNoiseLite.new()
    noise.seed = hash(chunk.chunk_pos)

    for x in range(CHUNK_SIZE):
        for y in range(CHUNK_SIZE):
            var world_pos = chunk.chunk_pos * CHUNK_SIZE + Vector2i(x, y)
            var noise_val = noise.get_noise_2d(world_pos.x, world_pos.y)

            var tile_id = 0
            if noise_val > 0.2:
                tile_id = 1  # Grass
            elif noise_val > -0.2:
                tile_id = 2  # Dirt
            else:
                tile_id = 3  # Stone

            chunk.tilemap.set_cell(0, Vector2i(x, y), tile_id, Vector2i.ZERO)

func save_chunk_data(chunk: TileMapChunk):
    var tile_data = []
    for cell in chunk.tilemap.get_used_cells(0):
        var tile_id = chunk.tilemap.get_cell_source_id(0, cell)
        var atlas_coord = chunk.tilemap.get_cell_atlas_coords(0, cell)
        tile_data.append({
            "pos": cell,
            "tile_id": tile_id,
            "atlas_coord": atlas_coord
        })
    world_data[chunk.chunk_pos] = tile_data
```

**Key Parameters:**
- **Chunk Size:** 16x16 (small worlds), 32x32 (medium), 64x64 (large procedural)
- **Load Distance:** 1-2 chunks (platformers), 3-5 (open world exploration)
- **Unload Distance:** Load distance + 1 or 2 (hysteresis prevents thrashing)
- **Pool Size:** 20-100 TileMap nodes (depends on visible chunk count)
- **Memory Budget:** ~1-10MB per chunk loaded (depends on tile complexity)

**Edge Cases:**
- **Chunk Boundaries:** Ensure seamless rendering at edges, no gaps or overlaps
- **Fast Movement:** Load ahead in movement direction, predictive loading
- **Teleportation:** Unload all chunks, immediate load at new position
- **Chunk Thrashing:** Player at chunk boundary causes load/unload spam - use hysteresis
- **Modification Tracking:** Mark chunks dirty when tiles modified, batch saves
- **Multi-layer Tilemaps:** Each chunk needs all layers, coordinate saves

**When NOT to Use:**
- Small static levels (<1000x1000 tiles) - simpler to load entire map
- Level-based games with discrete screens - load per screen instead
- Highly dynamic worlds where all tiles update - chunking overhead not worth it
- Memory not constrained (desktop with 16GB+ RAM and small world)

**Examples from Shipped Games:**

1. **Terraria:** 8400x2400 tile world split into 200x200 tile chunks. Only loads ~16-25 chunks around player. Unmodified chunks regenerate procedurally on demand. Enables massive worlds on limited hardware (original Xbox 360: 512MB RAM).

2. **Stardew Valley:** Farm uses chunked rendering despite static size. Optimization allows unlimited farms in multiplayer without memory scaling. Chunks unload when players far away, critical for 4-player split-screen.

3. **Dead Cells:** Each biome level chunked into rooms. Loads 3-5 rooms ahead, unloads 2+ rooms behind. Enables seamless procedural generation without loading screens. Chunks are 40x30 tiles, perfect for room-based design.

4. **Celeste:** Levels chunked into screens (40x23 tiles each). Loads adjacent screens for seamless scrolling. Unloads screens 2+ away. Strawberries and secrets persist via chunk save data. Enables B-sides/C-sides without memory duplication.

5. **Minecraft (2D equivalent):** Gold standard of chunking. 16x16 tile chunks, loads 6-12 chunk radius (view distance). Infinite worlds possible through chunk streaming from disk. Each chunk ~10KB compressed.

**Platform Considerations:**
- **Mobile:** Essential - RAM limited to 1-4GB. Use smaller chunks (16x16), aggressive unloading (distance 2). Target <50 chunks loaded.
- **Desktop:** Helpful for large worlds - can keep more chunks (100+) with 8GB+ RAM. Use larger chunks (64x64) for better batching.
- **Console (Switch):** Critical - 4GB RAM shared with OS. Target 30-50 chunks max. Stream from fast storage (SSD/cartridge).
- **WebGL:** Absolutely necessary - browser memory limits strict. Small chunks (16x16), minimal load distance (1-2).
- **Storage:** Save only modified chunks to disk, procedural regeneration for pristine chunks saves space.

**Godot-Specific Notes:**
- Godot's TileMap has built-in culling but still allocates memory for entire map
- Use multiple TileMap nodes (one per chunk) rather than single massive TileMap
- Set `TileMap.cell_quadrant_size` to match chunk size for optimal batching
- Use `TileMap.get_used_cells()` efficiently - cache results per chunk
- Godot 4.x TileMap layers: All layers in same chunk TileMap node
- Consider RuntimeTileMapData for pure data storage vs full TileMap nodes
- Use `ResourceLoader.load_threaded_*` for async chunk loading

**Synergies:**
- **Object Pooling:** Pool TileMap nodes to avoid allocation overhead
- **Sprite Batching:** Chunks batch internally, each chunk is one batch
- **Dirty Rectangles:** Only re-render modified chunks
- **Frustum Culling:** Cull entire chunks outside viewport in one check
- **Procedural Generation:** Generate chunks on-demand instead of storing all data

**Measurement/Profiling:**
- **Memory Usage:** Monitor `Performance.MEMORY_STATIC` - should scale with viewport not world size
- **Chunk Load Time:** Target <5ms per chunk load (16-33ms for 3-6 chunks)
- **Active Chunk Count:** Track `active_chunks.size()` - should be 9-25 typically
- **Draw Calls:** Each chunk = 1-3 draw calls, monitor `RENDER_2D_DRAW_CALLS_IN_FRAME`
- **Frame Time:** Chunk loading should not cause spikes >2ms
- **Profiling Code:**
```gdscript
var chunk_load_time = Time.get_ticks_usec()
load_chunk(coord)
chunk_load_time = Time.get_ticks_usec() - chunk_load_time
print("Chunk load: ", chunk_load_time / 1000.0, "ms")
```

**Advanced Patterns:**
- **Predictive Loading:** Analyze player velocity, preload chunks in movement direction
- **Priority Queue:** Load closest chunks first, deprioritize behind player
- **Chunk LOD:** Distant visible chunks use simplified tile sets (fewer atlas coords)
- **Compression:** RLE or bit-packing for repetitive chunk data
- **Worker Threads:** Generate/load chunks on background thread (Godot 4.x threading)
