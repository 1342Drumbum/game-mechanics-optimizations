### 33. Render Target Pooling

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Render target allocation overhead and memory waste. Creating/destroying render targets mid-frame = GPU pipeline stall = 0.5-2ms per allocation. Post-processing chain with 10 effects × 2ms = 20ms overhead (exceeds budget). RT pooling pre-allocates targets, reuses across effects: 10 effects share 3-4 pooled targets = 20ms → 0.1ms overhead, saving 19.9ms. Also prevents fragmentation and reduces memory by 40-60% through reuse.

**Technical Explanation:**
Render target pool maintains pre-allocated textures of various sizes/formats. When effect needs RT, request from pool (instant); when done, return to pool. Pool tracks which RTs are in-use vs available. Smart pooling matches request to smallest suitable RT (saves memory). Multiple effects share same physical RT if they don't overlap temporally. Critical for post-processing (bloom, DOF, SSR, etc.) which each need temp buffers. Modern engines pool all temporary RTs: shadow maps, G-buffers, post-process chains. Prevents memory spikes and allocation stalls.

**Algorithmic Complexity:**
- No pooling: O(n) allocations, O(n) stalls, O(n × size) memory
- With pooling: O(1) get/return, O(0) stalls, O(max_concurrent × size) memory
- Memory savings: 40-70% typical (reuse across effects)
- Time savings: 1-10ms per frame (allocation overhead eliminated)
- Management overhead: <0.1ms for pool lookups

**Implementation Pattern:**
```gdscript
# Godot Render Target Pool
class_name RenderTargetPool
extends RefCounted

class PooledRT:
    var rid: RID
    var texture: Texture2D
    var size: Vector2i
    var format: int
    var in_use: bool = false
    var last_used_frame: int = 0

var pool: Array[PooledRT] = []
var frame_count: int = 0
var max_pool_size: int = 20
var eviction_frame_threshold: int = 60  # Evict after 60 frames unused

func _init():
    pass

func get_render_target(size: Vector2i, format: int = RenderingDevice.DATA_FORMAT_R8G8B8A8_UNORM) -> PooledRT:
    # Try to find existing unused RT with matching specs
    var best_match: PooledRT = null
    var best_waste: int = 999999

    for rt in pool:
        if rt.in_use:
            continue

        # Check if RT is suitable (size >= requested, format matches)
        if rt.size.x >= size.x and rt.size.y >= size.y and rt.format == format:
            var waste = (rt.size.x - size.x) * (rt.size.y - size.y)

            if waste < best_waste:
                best_waste = waste
                best_match = rt

    # Found a match
    if best_match:
        best_match.in_use = true
        best_match.last_used_frame = frame_count
        return best_match

    # No match, create new RT
    return _create_new_rt(size, format)

func _create_new_rt(size: Vector2i, format: int) -> PooledRT:
    var rd = RenderingServer.get_rendering_device()
    if not rd:
        push_error("RenderingDevice not available")
        return null

    # Create texture
    var texture_format = RDTextureFormat.new()
    texture_format.width = size.x
    texture_format.height = size.y
    texture_format.format = format
    texture_format.usage_bits = RenderingDevice.TEXTURE_USAGE_SAMPLING_BIT | \
                                 RenderingDevice.TEXTURE_USAGE_COLOR_ATTACHMENT_BIT | \
                                 RenderingDevice.TEXTURE_USAGE_STORAGE_BIT

    var view = RDTextureView.new()
    var rid = rd.texture_create(texture_format, view, [])

    # Create pooled entry
    var pooled = PooledRT.new()
    pooled.rid = rid
    pooled.size = size
    pooled.format = format
    pooled.in_use = true
    pooled.last_used_frame = frame_count

    pool.append(pooled)

    print("Created new RT: ", size, " Format: ", format)
    return pooled

func return_render_target(rt: PooledRT):
    if rt and rt in pool:
        rt.in_use = false

func end_frame():
    frame_count += 1

    # Evict old unused RTs
    var to_remove = []
    for i in range(pool.size()):
        var rt = pool[i]

        if not rt.in_use and (frame_count - rt.last_used_frame) > eviction_frame_threshold:
            to_remove.append(i)

    # Remove in reverse order
    to_remove.reverse()
    for i in to_remove:
        var rt = pool[i]
        _free_rt(rt)
        pool.remove_at(i)

func _free_rt(rt: PooledRT):
    var rd = RenderingServer.get_rendering_device()
    if rd and rt.rid.is_valid():
        rd.free_rid(rt.rid)

    print("Freed RT: ", rt.size)

func get_stats() -> Dictionary:
    var total_memory = 0
    var in_use_count = 0
    var available_count = 0

    for rt in pool:
        var bytes = rt.size.x * rt.size.y * 4  # Assume RGBA8
        total_memory += bytes

        if rt.in_use:
            in_use_count += 1
        else:
            available_count += 1

    return {
        "total_rts": pool.size(),
        "in_use": in_use_count,
        "available": available_count,
        "memory_mb": total_memory / (1024.0 * 1024.0)
    }

# Post-processing with RT pooling
class PostProcessChain:
    var rt_pool: RenderTargetPool

    func _init(pool: RenderTargetPool):
        rt_pool = pool

    func apply_effects(source: Texture2D, size: Vector2i) -> Texture2D:
        # Effect 1: Bloom
        var bloom_rt = rt_pool.get_render_target(size / 2)
        _apply_bloom(source, bloom_rt)

        # Effect 2: Tone mapping
        var tonemap_rt = rt_pool.get_render_target(size)
        _apply_tonemap(source, bloom_rt, tonemap_rt)

        # Return intermediate RTs to pool
        rt_pool.return_render_target(bloom_rt)

        # Final RT contains result
        return tonemap_rt.texture

    func _apply_bloom(source: Texture2D, target: RenderTargetPool.PooledRT):
        # Render bloom to target
        pass

    func _apply_tonemap(source: Texture2D, bloom: RenderTargetPool.PooledRT, target: RenderTargetPool.PooledRT):
        # Combine source + bloom to target
        pass

# Hierarchical RT pooling (multiple size buckets)
class HierarchicalRTPool:
    var pools: Dictionary = {}  # Size bucket -> Array[RT]
    var size_buckets: Array[Vector2i] = [
        Vector2i(256, 256),
        Vector2i(512, 512),
        Vector2i(1024, 1024),
        Vector2i(2048, 2048),
        Vector2i(4096, 4096),
    ]

    func get_rt(requested_size: Vector2i) -> RenderTargetPool.PooledRT:
        # Find smallest bucket that fits
        var bucket = _find_bucket(requested_size)

        if not bucket in pools:
            pools[bucket] = []

        # Try to find free RT in bucket
        for rt in pools[bucket]:
            if not rt.in_use:
                rt.in_use = true
                return rt

        # Create new RT in this bucket
        return _create_in_bucket(bucket)

    func _find_bucket(size: Vector2i) -> Vector2i:
        for bucket in size_buckets:
            if bucket.x >= size.x and bucket.y >= size.y:
                return bucket

        return size_buckets[-1]  # Largest bucket

    func _create_in_bucket(bucket: Vector2i) -> RenderTargetPool.PooledRT:
        # Create new RT
        pass
        return null
```

**Key Parameters:**
- Pool size: 10-50 RTs depending on complexity
- Eviction threshold: 30-120 frames (1-2 seconds at 60fps)
- Size buckets: Powers of 2 (256, 512, 1K, 2K, 4K)
- Format variety: RGBA8, RGBA16F, R11G11B10F, depth formats
- Memory budget: 100-500MB for RT pool

**Edge Cases:**
- Pool exhaustion: Grow pool or reuse oldest
- Large variety of sizes: Use flexible allocation
- Sudden memory spike: Evict unused RTs immediately
- Format mismatches: Maintain separate pools per format
- Resolution changes: Clear pool on viewport resize

**When NOT to Use:**
- Simple games with 1-2 post effects
- Fixed render targets never released
- Very predictable RT usage
- Extremely limited RAM (<2GB)

**Examples from Shipped Games:**
1. **Unreal Engine 4/5**: Extensive RT pooling, 100+ RTs in pool. Saves 200MB+ memory, eliminates allocation stalls.
2. **Unity**: RT pooling standard, RenderTexture.GetTemporary() API. 40-60% memory savings in post-heavy scenes.
3. **DOOM (2016)**: Custom RT manager, 50+ pooled targets. Critical for 60fps (eliminates stalls).
4. **Decima Engine (Horizon, Death Stranding)**: Hierarchical RT pooling, size buckets. Efficient memory use on PS4.
5. **Frostbite**: Smart RT pooling with LRU eviction. Handles complex post-processing chains efficiently.

**Platform Considerations:**
- **PC**: Large pool (30-50 RTs), generous memory
- **Console**: Moderate pool (20-30 RTs), careful sizing
- **Mobile**: Small pool (10-15 RTs), aggressive eviction
- **VR**: Per-eye pooling, doubled requirements

**Godot-Specific Notes:**
- Godot has internal RT management but custom pooling beneficial
- Use RenderingDevice for explicit control
- Viewport textures can be pooled for SubViewport usage
- Godot 4.x: Better automatic management but manual still useful

**Synergies:**
- Essential for **Post-Processing Effects** (reuse RTs)
- Pairs with **Shadow Mapping** (pool shadow RTs)
- Works with **Screen-Space Effects** (temp buffers)
- Enables **Dynamic Resolution** (different sized RTs)

**Measurement:**
- Track allocations: Should be near zero mid-frame
- Monitor memory: 40-60% reduction typical
- Measure stalls: <0.1ms with pooling vs 1-10ms without
- Profile pool usage: Hit rate should be >80%

**Additional Resources:**
- GDC 2016: "Rendering Technology in Quantum Break"
- GPU Gems: Render Target Management
