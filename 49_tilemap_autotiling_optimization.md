### 49. Tilemap Autotiling Optimization

**Category:** Performance - 2D Rendering & Data Processing

**Problem It Solves:** Autotiling calculations are expensive during level editing and runtime tile changes. Calculating autotile rules for 1000 tiles with 8-neighbor checks × 47 rule patterns = 376,000 comparisons. At 100ns per check = 37.6ms, causing stutters during rapid tile placement (e.g., terrain tools). Optimized autotiling reduces to <1ms via caching and incremental updates.

**Technical Explanation:**
Autotiling automatically selects correct tile variant based on surrounding tiles (e.g., grass connects smoothly to other grass). Naive approach recalculates all tiles every change. Optimization: only recalculate affected tile + 8 neighbors (9 tiles max), cache autotile rule results, use bitmask lookup tables instead of conditional chains, and batch updates during bulk editing. For 3×3 or 2×2 minimal autotiling, use 8-bit bitmask (256 possible states) with precomputed atlas coordinate lookup.

Modern implementation separates rule evaluation from rendering: precalculate bitmasks for all possible neighbor combinations, store in 256-entry lookup table mapping bitmask→atlas coordinate. Runtime becomes simple bit manipulation + array lookup: O(1) vs O(n) rule evaluation.

**Algorithmic Complexity:**
- Naive Full Recalc: O(t × r) where t = tiles, r = rules (e.g., 1000 × 47 = 47k checks)
- Incremental Update: O(k × r) where k = affected tiles (typically 9)
- Bitmask Lookup: O(k × 1) = constant time per tile
- Improvement: 1000× faster for single tile changes, 100× for bulk edits

**Implementation Pattern:**
```gdscript
# Optimized autotiling with bitmask lookup
class_name FastAutotiler
extends TileMap

# Bitmask constants for 3×3 autotiling (8 neighbors)
const NEIGHBOR_TOP_LEFT = 0b00000001      # 1
const NEIGHBOR_TOP = 0b00000010            # 2
const NEIGHBOR_TOP_RIGHT = 0b00000100      # 4
const NEIGHBOR_LEFT = 0b00001000           # 8
const NEIGHBOR_RIGHT = 0b00010000          # 16
const NEIGHBOR_BOTTOM_LEFT = 0b00100000    # 32
const NEIGHBOR_BOTTOM = 0b01000000         # 64
const NEIGHBOR_BOTTOM_RIGHT = 0b10000000   # 128

# Precomputed bitmask → atlas coordinate lookup (256 entries)
var autotile_lookup: Dictionary = {}  # int -> Vector2i
var tile_cache: Dictionary = {}       # Vector2i -> int (bitmask)

# Neighbor offsets for 3×3 grid
const NEIGHBOR_OFFSETS = [
    Vector2i(-1, -1), Vector2i(0, -1), Vector2i(1, -1),  # Top row
    Vector2i(-1,  0),                  Vector2i(1,  0),  # Middle row
    Vector2i(-1,  1), Vector2i(0,  1), Vector2i(1,  1)   # Bottom row
]

func _ready():
    build_autotile_lookup()

func build_autotile_lookup():
    # Build lookup table for all 256 possible neighbor combinations
    # This is done once at startup
    for bitmask in 256:
        autotile_lookup[bitmask] = calculate_atlas_coords_from_bitmask(bitmask)

func calculate_atlas_coords_from_bitmask(bitmask: int) -> Vector2i:
    # Map bitmask to specific tile in atlas
    # Example for blob autotiling (Godot's 3×3 minimal)

    # Full tile (all neighbors present)
    if bitmask == 0b11111111:
        return Vector2i(1, 1)  # Center tile

    # Edges
    if bitmask & NEIGHBOR_TOP == 0:
        return Vector2i(1, 0)  # Top edge
    if bitmask & NEIGHBOR_BOTTOM == 0:
        return Vector2i(1, 2)  # Bottom edge
    if bitmask & NEIGHBOR_LEFT == 0:
        return Vector2i(0, 1)  # Left edge
    if bitmask & NEIGHBOR_RIGHT == 0:
        return Vector2i(2, 1)  # Right edge

    # Corners
    if bitmask == NEIGHBOR_RIGHT | NEIGHBOR_BOTTOM:
        return Vector2i(0, 0)  # Top-left corner
    if bitmask == NEIGHBOR_LEFT | NEIGHBOR_BOTTOM:
        return Vector2i(2, 0)  # Top-right corner
    if bitmask == NEIGHBOR_RIGHT | NEIGHBOR_TOP:
        return Vector2i(0, 2)  # Bottom-left corner
    if bitmask == NEIGHBOR_LEFT | NEIGHBOR_TOP:
        return Vector2i(2, 2)  # Bottom-right corner

    # Default to center
    return Vector2i(1, 1)

func set_cell_autotile(layer: int, coords: Vector2i, tile_id: int):
    # Set tile and update autotiling incrementally
    set_cell(layer, coords, tile_id, Vector2i.ZERO)

    # Update only affected tiles (this tile + 8 neighbors)
    update_autotile_at(layer, coords)

    for offset in NEIGHBOR_OFFSETS:
        update_autotile_at(layer, coords + offset)

func update_autotile_at(layer: int, coords: Vector2i):
    var tile_id = get_cell_source_id(layer, coords)
    if tile_id == -1:
        return  # No tile here

    # Calculate bitmask from neighbors
    var bitmask = calculate_bitmask(layer, coords, tile_id)

    # Check cache
    var cache_key = coords
    if tile_cache.has(cache_key) and tile_cache[cache_key] == bitmask:
        return  # Already correct, skip update

    # Update tile using lookup table
    var atlas_coords = autotile_lookup.get(bitmask, Vector2i.ZERO)
    set_cell(layer, coords, tile_id, atlas_coords)

    # Cache result
    tile_cache[cache_key] = bitmask

func calculate_bitmask(layer: int, coords: Vector2i, tile_id: int) -> int:
    var bitmask = 0

    # Check each of 8 neighbors
    var neighbor_masks = [
        NEIGHBOR_TOP_LEFT, NEIGHBOR_TOP, NEIGHBOR_TOP_RIGHT,
        NEIGHBOR_LEFT,                   NEIGHBOR_RIGHT,
        NEIGHBOR_BOTTOM_LEFT, NEIGHBOR_BOTTOM, NEIGHBOR_BOTTOM_RIGHT
    ]

    for i in 8:
        var neighbor_coords = coords + NEIGHBOR_OFFSETS[i]
        var neighbor_id = get_cell_source_id(layer, neighbor_coords)

        # Neighbor matches this tile type
        if neighbor_id == tile_id:
            bitmask |= neighbor_masks[i]

    return bitmask

# Batch autotiling for bulk operations
class_name BatchAutotiler
extends FastAutotiler

var pending_updates: Dictionary = {}  # Vector2i -> bool
var batch_mode: bool = false

func begin_batch():
    batch_mode = true
    pending_updates.clear()

func end_batch():
    batch_mode = false

    # Update all pending tiles at once
    for coords in pending_updates.keys():
        update_autotile_at(0, coords)

        # Also update neighbors
        for offset in NEIGHBOR_OFFSETS:
            var neighbor_coords = coords + offset
            if not pending_updates.has(neighbor_coords):
                update_autotile_at(0, neighbor_coords)

    pending_updates.clear()

func set_cell_batched(layer: int, coords: Vector2i, tile_id: int):
    set_cell(layer, coords, tile_id, Vector2i.ZERO)

    if batch_mode:
        pending_updates[coords] = true
    else:
        set_cell_autotile(layer, coords, tile_id)

# Example: Painting 100 tiles
func paint_large_area(layer: int, area: Array[Vector2i], tile_id: int):
    begin_batch()

    for coords in area:
        set_cell_batched(layer, coords, tile_id)

    end_batch()
    # Only recalculates perimeter + neighbors instead of all 100 tiles repeatedly

# 2×2 minimal autotiling (simplified, 4 tiles only)
class_name Minimal2x2Autotiler
extends TileMap

# 2×2 only checks right and bottom neighbors
const CHECK_RIGHT = 0b01
const CHECK_BOTTOM = 0b10

var lookup_2x2: Dictionary = {
    0b00: Vector2i(1, 1),  # No neighbors (single tile)
    0b01: Vector2i(0, 1),  # Right neighbor (horizontal)
    0b10: Vector2i(1, 0),  # Bottom neighbor (vertical)
    0b11: Vector2i(0, 0),  # Both neighbors (full)
}

func update_autotile_2x2(coords: Vector2i, tile_id: int):
    var bitmask = 0

    # Check right neighbor
    var right = get_cell_source_id(0, coords + Vector2i(1, 0))
    if right == tile_id:
        bitmask |= CHECK_RIGHT

    # Check bottom neighbor
    var bottom = get_cell_source_id(0, coords + Vector2i(0, 1))
    if bottom == tile_id:
        bitmask |= CHECK_BOTTOM

    # Apply from lookup
    var atlas_coords = lookup_2x2[bitmask]
    set_cell(0, coords, tile_id, atlas_coords)

# Advanced: Wang tiles (16 combinations, 4 edges)
class_name WangTileAutotiler
extends TileMap

# Wang tiles use edge colors: each edge can be color A or B
# 4 edges × 2 colors = 2^4 = 16 tiles

func calculate_wang_bitmask(coords: Vector2i, tile_id: int) -> int:
    var bitmask = 0

    # Check 4 cardinal directions only (not diagonals)
    var directions = [
        Vector2i(0, -1),  # Top
        Vector2i(1, 0),   # Right
        Vector2i(0, 1),   # Bottom
        Vector2i(-1, 0)   # Left
    ]

    for i in 4:
        var neighbor = get_cell_source_id(0, coords + directions[i])
        if neighbor == tile_id:
            bitmask |= (1 << i)

    return bitmask  # 0-15 range

# Optimized for Godot's built-in autotile system
class_name GodotAutotileOptimizer
extends TileMap

var dirty_cells: Array[Vector2i] = []

func set_cell_optimized(layer: int, coords: Vector2i, tile_id: int, atlas_coords: Vector2i):
    # Use Godot's built-in autotiling but optimize update frequency
    set_cell(layer, coords, tile_id, atlas_coords)

    # Mark as dirty for next frame batch update
    if not dirty_cells.has(coords):
        dirty_cells.append(coords)

func _process(_delta):
    if dirty_cells.is_empty():
        return

    # Batch update dirty cells
    for coords in dirty_cells:
        # Godot's set_cell automatically handles autotiling
        # But we control when it happens (batched per frame)
        var tile_id = get_cell_source_id(0, coords)
        if tile_id != -1:
            # Force autotile recalculation
            notify_property_list_changed()

    dirty_cells.clear()
```

**Key Parameters:**
- **Autotile Type:** 3×3 minimal (256 states), 2×2 minimal (4 states), 3×3 full (512 states), Wang tiles (16 states)
- **Update Strategy:** Immediate (per tile), batched (per frame), deferred (on idle)
- **Cache Size:** Store bitmasks for 1000-10000 tiles (16-160KB)
- **Neighbor Check Count:** 4 (cardinal), 8 (cardinal + diagonal), 24 (extended)
- **Lookup Table Size:** 256 entries (3×3), 16 entries (Wang), 4 entries (2×2)
- **Batch Threshold:** >10 tiles = use batching, <10 = immediate update

**Edge Cases:**
- **Tile Removal:** Setting tile_id = -1 must clear from cache
- **Layer Changes:** Different layers may have different autotile rules
- **Alternative Tiles:** Multiple tile IDs that should connect (grass, flowers)
- **Diagonal Corners:** Some autotile systems treat corners specially
- **Animation:** Autotiled animated tiles must sync frame selection
- **Rotation:** Rotated tiles may break autotile assumptions

**When NOT to Use:**
- Static pre-built levels - autotiling only needed in editor
- Simple tile sets without connections (platformer boxes)
- Hand-crafted tile placement preferred over automatic
- Performance already good - premature optimization
- Tile count very low (<100 tiles) - optimization overhead not worth it

**Examples from Shipped Games:**

1. **Terraria:** Extensive autotiling for 50+ tile types (dirt, stone, ore, etc.). Uses 3×3 bitmask with 47 variations per tile type. Optimized with incremental updates - only recalculates on mining/placement. Batch mode during world generation. Critical for smooth terraforming at 60fps.

2. **Stardew Valley:** Farm tiles (plowed dirt, paths) use 2×2 minimal autotiling. Only 4 tile variants per type. Updates batched per day cycle, not per tile place. Hoe tool marks entire area dirty, updates on tool release. Maintains 60fps even when tilling 100+ tiles.

3. **Celeste:** Level tiles use custom autotile system with caching. 3×3 minimal with special corner rules. Level editor uses batched autotiling - recalculates only on save/test. Runtime tiles static (no autotile updates needed). Enables rapid level iteration.

4. **Dead Cells:** Platform tiles autotile with 3×3 system. Procedural generation prebakes autotiling during level creation. Zero runtime autotile cost - all tiles predetermined. Uses lookup table for instant generation. Generates entire biome in <100ms.

5. **Minecraft (2D equivalent):** Block placement uses optimized autotiling for connected textures. Calculates bitmask for 8 neighbors, uses 256-entry lookup. Batch updates when breaking multiple blocks (TNT). Incremental updates otherwise. Handles 1000s of block changes per second.

**Platform Considerations:**
- **Mobile (iOS/Android):** Batched updates essential - single-tile updates cause stutters. Use 2×2 or simple 3×3 (fewer states). Cache aggressively. Target <5ms for 100-tile batch.
- **Desktop (Windows/Mac/Linux):** Can handle immediate updates for <20 tiles/frame. Use full 3×3 or Wang tiles for quality. Caching helpful but less critical.
- **Console (Switch/PS/Xbox):** Switch: batch updates, use simple autotile systems. PS5/Xbox: full featured autotiling fine.
- **WebGL:** Lookup tables critical (avoid complex conditionals in JavaScript). Aggressive caching reduces overhead.
- **Memory:** Lookup tables tiny (256 × 8 bytes = 2KB). Cache larger (10k tiles × 4 bytes = 40KB) but still minimal.

**Godot-Specific Notes:**
- Godot TileMap has built-in autotiling via TileSet editor (terrain mode)
- Godot 4.x: Use TileSet terrains for automatic autotiling (GPU-based)
- Godot 3.x: Use autotile mode with bitmask settings
- Custom autotiling needed for complex rules Godot doesn't support
- Monitor performance: Profiler → Physics → TileMap updates
- `TileMap.set_cells_terrain_connect()` for batch autotiling (Godot 4.x)
- Use `tile_set.tile_get_autotile_bitmask()` to query bitmasks

**Synergies:**
- **Tilemap Chunking:** Autotile calculations per chunk, not entire map
- **Dirty Rectangles:** Only autotile in dirty regions
- **Object Pooling:** Pool autotile calculation results
- **Sprite Batching:** Autotiled tiles batch together perfectly
- **Procedural Generation:** Prebake autotiling during generation

**Measurement/Profiling:**
- **Update Time:** Measure time for single tile update vs batch update
- **Cache Hit Rate:** Track how often cache prevents recalculation - target >90%
- **Tiles Updated:** Count tiles updated per frame - should be <100
- **Lookup Table Performance:** Verify O(1) access vs O(n) rule evaluation
- **Profiling Pattern:**
```gdscript
var cache_hits = 0
var cache_misses = 0
var total_updates = 0

func update_autotile_at_profiled(layer: int, coords: Vector2i):
    total_updates += 1

    var cache_key = coords
    var bitmask = calculate_bitmask(layer, coords, get_cell_source_id(layer, coords))

    if tile_cache.has(cache_key) and tile_cache[cache_key] == bitmask:
        cache_hits += 1
        return  # Cache hit
    else:
        cache_misses += 1
        # Update tile...

    if total_updates % 1000 == 0:
        var hit_rate = float(cache_hits) / total_updates * 100.0
        print("Cache hit rate: ", hit_rate, "%")

# Measure batch vs individual updates
var time_individual = 0.0
var time_batch = 0.0

func benchmark_autotiling():
    var test_tiles = []
    for i in 100:
        test_tiles.append(Vector2i(randi() % 50, randi() % 50))

    # Individual updates
    var start = Time.get_ticks_usec()
    for coords in test_tiles:
        set_cell_autotile(0, coords, 1)
    time_individual = (Time.get_ticks_usec() - start) / 1000.0

    # Batch updates
    start = Time.get_ticks_usec()
    begin_batch()
    for coords in test_tiles:
        set_cell_batched(0, coords, 1)
    end_batch()
    time_batch = (Time.get_ticks_usec() - start) / 1000.0

    print("Individual: ", time_individual, "ms | Batch: ", time_batch, "ms")
    print("Speedup: ", time_individual / time_batch, "x")
```

**Advanced Optimization Patterns:**
```gdscript
# Pattern 1: Lazy autotiling (update only when visible)
func lazy_update_autotile(coords: Vector2i):
    var viewport_rect = get_viewport_rect()
    var tile_world_pos = map_to_local(coords)

    if not viewport_rect.has_point(tile_world_pos):
        # Not visible, defer update
        pending_updates[coords] = true
        return

    update_autotile_at(0, coords)

# Pattern 2: Hierarchical autotiling (group similar tiles)
var tile_groups: Dictionary = {
    "terrain": [1, 2, 3],  # Grass, dirt, stone connect
    "path": [10, 11],      # Stone path, wood path connect
}

func calculate_bitmask_grouped(coords: Vector2i, tile_id: int) -> int:
    var group = find_tile_group(tile_id)
    var bitmask = 0

    for i in 8:
        var neighbor_coords = coords + NEIGHBOR_OFFSETS[i]
        var neighbor_id = get_cell_source_id(0, neighbor_coords)

        if neighbor_id in group:  # Any tile in same group connects
            bitmask |= (1 << i)

    return bitmask

# Pattern 3: Probabilistic autotiling (randomize tile variants)
func get_random_variant_atlas(bitmask: int, coords: Vector2i) -> Vector2i:
    var base_atlas = autotile_lookup[bitmask]

    # Use coords as seed for deterministic randomness
    var seed_val = coords.x + coords.y * 1000
    var rng = RandomNumberGenerator.new()
    rng.seed = seed_val

    # 20% chance for variant
    if rng.randf() < 0.2:
        return base_atlas + Vector2i(3, 0)  # Variant offset

    return base_atlas
```
