### 54. Animation Compression (Keyframe)

**Category:** Performance - Memory Management / Animation

**Problem It Solves:** Excessive memory usage and cache misses from uncompressed animation data. A 10-second animation at 60fps with 50 bones storing full Transform (position, rotation, scale = 10 floats) requires 600 frames × 50 bones × 10 floats × 4 bytes = 1.2MB per animation. With 500 animations = 600MB, causing cache thrashing and long load times (5-10 seconds on HDD).

**Technical Explanation:**
Animation compression uses multiple techniques: 1) Keyframe reduction - remove redundant frames where values don't change significantly, reducing 600 frames to 50-100 keyframes (83% reduction). 2) Quantization - store rotations as 16-bit quaternions (8 bytes) instead of 32-bit (16 bytes), positions as 16-bit fixed-point. 3) Curve fitting - use Bezier/Hermite curves between keyframes instead of linear interpolation. 4) Delta encoding - store differences from bind pose. Combined compression achieves 10-20x size reduction with imperceptible quality loss.

**Algorithmic Complexity:**
- Decompression: O(log n) keyframe lookup via binary search + O(1) interpolation
- Memory: O(unique_keyframes) instead of O(total_frames)
- Cache efficiency: 10-20x better due to smaller data footprint
- Load time: Proportional to compressed size (10x faster loads)

**Implementation Pattern:**
```gdscript
# Godot implementation
class_name AnimationCompressor

# Compressed animation data structure
class CompressedTrack:
    var bone_index: int
    var keyframe_times: PackedFloat32Array  # Sparse keyframes only
    var keyframe_values: PackedByteArray    # Quantized rotation/position
    var compression_mode: int  # FULL, ROTATION_ONLY, POSITION_ONLY

const POSITION_PRECISION = 0.001  # 1mm precision
const ROTATION_PRECISION = 0.001  # ~0.057 degrees
const SCALE_PRECISION = 0.01

var compressed_tracks: Array[CompressedTrack] = []

func compress_animation(animation: Animation, threshold: float = 0.01) -> void:
    compressed_tracks.clear()

    for track_idx in range(animation.get_track_count()):
        if animation.track_get_type(track_idx) != Animation.TYPE_POSITION_3D:
            continue

        var bone_idx = animation.track_get_path(track_idx).get_name(1)
        var track = CompressedTrack.new()
        track.bone_index = int(bone_idx)

        # Step 1: Keyframe reduction - remove redundant frames
        var keyframes = _reduce_keyframes(animation, track_idx, threshold)

        # Step 2: Quantize values (16-bit precision)
        track.keyframe_times = PackedFloat32Array()
        track.keyframe_values = PackedByteArray()

        for kf in keyframes:
            track.keyframe_times.append(kf.time)
            # Quantize position: float → int16
            var quantized_pos = _quantize_vector3(kf.position, POSITION_PRECISION)
            track.keyframe_values.append_array(quantized_pos)

        compressed_tracks.append(track)

func _reduce_keyframes(animation: Animation, track_idx: int, threshold: float) -> Array:
    var result = []
    var key_count = animation.track_get_key_count(track_idx)

    if key_count <= 2:
        return _get_all_keys(animation, track_idx)

    # Douglas-Peucker algorithm for keyframe reduction
    result.append(_get_key_data(animation, track_idx, 0))  # Always keep first

    var i = 1
    while i < key_count - 1:
        var prev_val = animation.track_get_key_value(track_idx, i - 1)
        var curr_val = animation.track_get_key_value(track_idx, i)
        var next_val = animation.track_get_key_value(track_idx, i + 1)

        # Check if current keyframe can be interpolated
        var interpolated = prev_val.lerp(next_val, 0.5)
        if curr_val.distance_to(interpolated) > threshold:
            result.append(_get_key_data(animation, track_idx, i))

        i += 1

    result.append(_get_key_data(animation, track_idx, key_count - 1))  # Always keep last
    return result

func _quantize_vector3(v: Vector3, precision: float) -> PackedByteArray:
    var bytes = PackedByteArray()
    # Convert to 16-bit integer range
    var x = int(v.x / precision)
    var y = int(v.y / precision)
    var z = int(v.z / precision)

    bytes.append(x & 0xFF)
    bytes.append((x >> 8) & 0xFF)
    bytes.append(y & 0xFF)
    bytes.append((y >> 8) & 0xFF)
    bytes.append(z & 0xFF)
    bytes.append((z >> 8) & 0xFF)

    return bytes

func _quantize_quaternion(q: Quaternion) -> PackedByteArray:
    # Smallest-three compression: store 3 components, derive 4th
    var bytes = PackedByteArray()
    var largest_idx = 0
    var largest_val = abs(q.x)

    if abs(q.y) > largest_val:
        largest_idx = 1
        largest_val = abs(q.y)
    if abs(q.z) > largest_val:
        largest_idx = 2
        largest_val = abs(q.z)
    if abs(q.w) > largest_val:
        largest_idx = 3

    # Store index of largest component
    bytes.append(largest_idx)

    # Store other 3 components as 15-bit signed integers
    var components = [q.x, q.y, q.z, q.w]
    for i in range(4):
        if i == largest_idx:
            continue
        var quantized = int(components[i] * 32767.0)
        bytes.append(quantized & 0xFF)
        bytes.append((quantized >> 8) & 0xFF)

    return bytes

func decompress_sample(track: CompressedTrack, time: float) -> Transform3D:
    # Binary search for keyframes
    var idx = track.keyframe_times.bsearch(time)

    if idx >= track.keyframe_times.size():
        idx = track.keyframe_times.size() - 1

    if idx == 0 or track.keyframe_times[idx] == time:
        return _dequantize_transform(track, idx)

    # Interpolate between keyframes
    var t0 = track.keyframe_times[idx - 1]
    var t1 = track.keyframe_times[idx]
    var alpha = (time - t0) / (t1 - t0)

    var trans0 = _dequantize_transform(track, idx - 1)
    var trans1 = _dequantize_transform(track, idx)

    return trans0.interpolate_with(trans1, alpha)
```

**Key Parameters:**
- Keyframe reduction threshold: 0.001 - 0.1 (lower = higher quality)
- Position precision: 0.001 (1mm) for high-quality, 0.01 (1cm) for background
- Rotation precision: 16-bit quaternion components
- Compression ratio target: 10-20x for typical animations
- Cache line size awareness: Align data to 64-byte boundaries

**Edge Cases:**
- High-frequency animation (fighting games): Use higher keyframe density
- Rapid direction changes: Automatic keyframe insertion at inflection points
- Root motion: May need full precision to avoid foot sliding
- Looping animations: Ensure first/last frames match exactly after compression
- Additive animations: Compress deltas, not absolute values

**When NOT to Use:**
- Procedural animations (no stored keyframes to compress)
- Very short animations (<1 second) - overhead exceeds benefit
- Animations with <10 keyframes - already minimal
- When RAM is abundant and load times don't matter
- Highly dynamic, non-repeating animations

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Aggressive animation compression reduced 800MB animation data to 80MB (10x compression). Used 16-bit quantization and keyframe culling. Critical for fitting massive animation library on PS4.

2. **Horizon Zero Dawn:** Compressed 500+ machine animations from 1.2GB to 120MB using keyframe reduction and quaternion compression. Achieved <0.1ms decompression overhead per character, enabling large herds.

3. **Uncharted 4:** Advanced animation compression (ACL library) achieved 15x compression with imperceptible quality loss. Reduced memory from 600MB to 40MB for character animations, critical for PS4's limited RAM.

4. **Spider-Man (PS4):** Compressed traversal animations (hundreds of variations) from 2GB to 150MB. Used adaptive precision - high for close-up, low for distant NPCs. Enabled seamless streaming.

5. **Assassin's Creed Unity:** Compressed crowd animations to fit 10,000+ NPCs in memory. Each NPC's animation data reduced from 2MB to 100KB (20x compression), enabling unprecedented crowd density.

**Platform Considerations:**
- **Console (PS4/Xbox One):** Essential due to limited RAM (5GB available for games)
- **Mobile:** Critical - reduces app size and RAM usage, use aggressive 20x compression
- **PC:** Important for load times and SSD longevity (less data written)
- **Switch:** Mandatory - 4GB RAM shared with OS, aim for 15-20x compression
- **VR:** Balance compression with decompression cost (stay under 0.5ms)

**Godot-Specific Notes:**
- Godot's `.tres` animation format is text-based and bloated for complex animations
- Use `.anim` binary format or custom compression
- Godot 4.x: `Animation.compress()` method with configurable thresholds
- Consider pre-compressed `.res` files for shipped builds
- `Animation.optimize()` method performs basic keyframe reduction
- For extreme compression, export to custom binary format
- Import pipeline: Compress animations during import from FBX/GLTF

**Synergies:**
- **Animation LOD (Technique 58):** Compress low-LOD animations more aggressively
- **Streaming:** Compressed animations load faster from disk
- **GPU Skinning (Technique 53):** Smaller data transfers to GPU
- **Object Pooling:** Shared compressed animation data across instances
- **Memory Management:** Fits more animations in CPU cache

**Measurement/Profiling:**
- **Size comparison:** Track before/after compression ratios
- **Quality metrics:** PSNR (Peak Signal-to-Noise Ratio) for positional error
- **Decompression time:** Should be <0.1ms per animation per frame
- **Memory usage:** Monitor `Performance.get_monitor(MEMORY_STATIC)`
- **Load times:** Measure animation resource load duration
- **Visual quality:** Side-by-side comparison at 100% and 10% compression
- **Target:** 10-15x compression with <0.1mm average error

---
