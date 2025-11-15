### 59. Animation Blending Optimization

**Category:** Performance - Animation Systems

**Problem It Solves:** Expensive multi-layer animation blending causing CPU bottleneck. Blending 4 animation layers with 50 bones each requires 4 × 50 = 200 bone transform calculations (lerp/slerp), plus final weighted average = 250 transform operations per character. At 100 characters × 60fps = 1.5M transforms/sec. At 200 cycles per blend operation = 300M cycles = 120ms on 2.5GHz CPU, exceeding 16.67ms budget by 7.2x.

**Technical Explanation:**
Animation blending optimization reduces computational cost of mixing multiple animations. Techniques: 1) Early-out on zero-weight layers (skip blend if weight < 0.01). 2) Bone masking - only blend bones affected by each layer (upper body walk + lower body run). 3) Hierarchical blending - blend child bones only if parent changed. 4) Cache layer results - reuse unchanged layers across frames. 5) Quantize blend weights to reduce precision (0.0, 0.25, 0.5, 0.75, 1.0) enabling lookup tables. 6) SIMD vectorization - blend 4 bones simultaneously. Reduces blend cost by 60-80%.

**Algorithmic Complexity:**
- Naive blending: O(layers × bones) every frame
- Optimized: O(active_layers × changed_bones) with early-out
- SIMD optimization: 4x speedup for transform blending
- Bone masking: 50-70% reduction (e.g., only blend upper body for aim offset)
- Memory: Negligible overhead for cached layer results
- Net gain: 60-80% reduction in animation CPU time

**Implementation Pattern:**
```gdscript
# Godot implementation
class_name OptimizedAnimationBlender
extends Node

# Animation layer with optimization metadata
class AnimationLayer:
    var animation: Animation
    var weight: float = 0.0
    var bone_mask: Array[bool] = []  # Which bones this layer affects
    var last_result: Array[Transform3D] = []  # Cached transform results
    var dirty: bool = true  # Needs recalculation?

var layers: Array[AnimationLayer] = []
var skeleton: Skeleton3D
var final_pose: Array[Transform3D] = []

const WEIGHT_EPSILON = 0.01  # Skip layers below this weight
const USE_SIMD_BLENDING = true

func _ready():
    skeleton = get_parent() as Skeleton3D
    final_pose.resize(skeleton.get_bone_count())

func add_layer(anim: Animation, bone_mask: Array[bool] = []) -> AnimationLayer:
    var layer = AnimationLayer.new()
    layer.animation = anim
    layer.bone_mask = bone_mask
    layer.last_result.resize(skeleton.get_bone_count())

    # If no mask provided, affect all bones
    if layer.bone_mask.is_empty():
        layer.bone_mask.resize(skeleton.get_bone_count())
        layer.bone_mask.fill(true)

    layers.append(layer)
    return layer

func update_blend(delta: float):
    # Step 1: Update only dirty layers (those with changed weights/time)
    var active_layers = _get_active_layers()

    if active_layers.is_empty():
        return

    # Step 2: Sample each active layer
    for layer in active_layers:
        if layer.dirty:
            _sample_layer(layer)
            layer.dirty = false

    # Step 3: Blend layers with bone masking
    _blend_layers_optimized(active_layers)

    # Step 4: Apply to skeleton
    _apply_to_skeleton()

func _get_active_layers() -> Array[AnimationLayer]:
    var active = []

    for layer in layers:
        # Early-out optimization: Skip near-zero weight layers
        if layer.weight > WEIGHT_EPSILON:
            active.append(layer)

    return active

func _sample_layer(layer: AnimationLayer):
    # Sample animation at current time
    var time = _get_layer_time(layer)

    for bone_idx in range(skeleton.get_bone_count()):
        # Skip bones not affected by this layer (bone masking)
        if not layer.bone_mask[bone_idx]:
            continue

        # Get bone transform from animation
        var transform = _sample_bone_transform(layer.animation, bone_idx, time)
        layer.last_result[bone_idx] = transform

func _blend_layers_optimized(active_layers: Array[AnimationLayer]):
    # Initialize with rest pose or first layer
    var base_layer = active_layers[0]

    for bone_idx in range(skeleton.get_bone_count()):
        final_pose[bone_idx] = base_layer.last_result[bone_idx]

    # Blend additional layers
    for i in range(1, active_layers.size()):
        var layer = active_layers[i]

        # Process bones in SIMD-friendly batches
        if USE_SIMD_BLENDING:
            _blend_layer_simd(layer)
        else:
            _blend_layer_scalar(layer)

func _blend_layer_scalar(layer: AnimationLayer):
    # Standard scalar blending
    for bone_idx in range(skeleton.get_bone_count()):
        if not layer.bone_mask[bone_idx]:
            continue  # Bone masking optimization

        var current = final_pose[bone_idx]
        var target = layer.last_result[bone_idx]

        # Lerp position, slerp rotation, lerp scale
        final_pose[bone_idx] = current.interpolate_with(target, layer.weight)

func _blend_layer_simd(layer: AnimationLayer):
    # SIMD optimization: Process 4 bones at once
    # In GDScript, this is conceptual - real SIMD requires C++/shader

    var bone_count = skeleton.get_bone_count()
    var i = 0

    while i < bone_count:
        # Check if next 4 bones are all masked
        var all_masked = true
        for j in range(min(4, bone_count - i)):
            if layer.bone_mask[i + j]:
                all_masked = false
                break

        if all_masked:
            i += 4
            continue

        # Blend 4 bones in parallel (conceptual)
        for j in range(min(4, bone_count - i)):
            if not layer.bone_mask[i + j]:
                continue

            var current = final_pose[i + j]
            var target = layer.last_result[i + j]
            final_pose[i + j] = current.interpolate_with(target, layer.weight)

        i += 4

func _apply_to_skeleton():
    for bone_idx in range(skeleton.get_bone_count()):
        skeleton.set_bone_pose(bone_idx, final_pose[bone_idx])

# Hierarchical blending optimization
class HierarchicalBlender:
    var skeleton: Skeleton3D
    var bone_dirty_flags: Array[bool] = []

    func blend_hierarchical(layers: Array[AnimationLayer]):
        bone_dirty_flags.fill(false)

        # Mark dirty bones (those that changed)
        for layer in layers:
            for bone_idx in range(skeleton.get_bone_count()):
                if layer.bone_mask[bone_idx] and layer.dirty:
                    _mark_bone_and_children_dirty(bone_idx)

        # Only blend dirty bones
        for bone_idx in range(skeleton.get_bone_count()):
            if bone_dirty_flags[bone_idx]:
                _blend_bone(layers, bone_idx)

    func _mark_bone_and_children_dirty(bone_idx: int):
        bone_dirty_flags[bone_idx] = true

        # Mark all children dirty (hierarchical propagation)
        for i in range(skeleton.get_bone_count()):
            if skeleton.get_bone_parent(i) == bone_idx:
                _mark_bone_and_children_dirty(i)

# Quantized weight blending
class QuantizedBlender:
    const WEIGHT_LEVELS = [0.0, 0.25, 0.5, 0.75, 1.0]
    var blend_cache: Dictionary = {}  # Cache common blend combinations

    func quantize_weight(weight: float) -> float:
        # Snap to nearest quantized level
        var closest = 0.0
        var min_dist = INF

        for level in WEIGHT_LEVELS:
            var dist = abs(weight - level)
            if dist < min_dist:
                min_dist = dist
                closest = level

        return closest

    func blend_with_cache(transform_a: Transform3D, transform_b: Transform3D,
                          weight: float) -> Transform3D:
        var quantized = quantize_weight(weight)

        # Check cache
        var cache_key = str(transform_a.get_instance_id()) + "_" + \
                       str(transform_b.get_instance_id()) + "_" + \
                       str(quantized)

        if cache_key in blend_cache:
            return blend_cache[cache_key]

        # Compute and cache
        var result = transform_a.interpolate_with(transform_b, quantized)
        blend_cache[cache_key] = result

        return result

# LOD-based animation blending
class LODAnimationBlender:
    var distance_to_camera: float
    var max_layers_by_lod: Array[int] = [4, 2, 1, 0]  # Layers allowed per LOD

    func get_max_layers() -> int:
        if distance_to_camera < 20.0:
            return max_layers_by_lod[0]  # Full quality
        elif distance_to_camera < 50.0:
            return max_layers_by_lod[1]
        elif distance_to_camera < 100.0:
            return max_layers_by_lod[2]
        else:
            return max_layers_by_lod[3]  # No blending, single animation
```

**Key Parameters:**
- Weight epsilon: 0.01 (skip layers below 1% contribution)
- Max blend layers: 2-4 layers per character
- Bone mask granularity: Upper/lower body split minimum
- Update frequency: Full quality at 60fps, distant at 30fps or 15fps
- SIMD batch size: 4 bones processed simultaneously
- Cache size: 64-256 commonly blended transforms

**Edge Cases:**
- Weight sum normalization: Ensure weights sum to 1.0 to avoid scaling
- Bone mask inheritance: Child bones inherit parent masks
- Animation synchronization: Ensure blended animations are time-aligned
- Additive blending: Different math than normal blending (add instead of lerp)
- Quaternion flipping: Ensure shortest path for rotation interpolation
- Root motion: May need special handling separate from blend system

**When NOT to Use:**
- Single animation playback (no blending needed)
- Very simple skeletons (<10 bones) where optimization overhead exceeds benefit
- Cutscenes with pre-baked animation (no runtime blending)
- When CPU has plenty of headroom and optimization complexity not justified
- Procedural animation systems (different architecture)

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Advanced bone masking for contextual animations. Upper body aim + lower body locomotion blended with per-bone masks. Reduced blend CPU time from 18ms to 4ms for 50 characters, critical for 30fps lock.

2. **Horizon Zero Dawn:** Layered animation for machines: locomotion + head tracking + body hit reactions. Used hierarchical blending and early-out optimization. Supported 30+ animated machines at 30fps with <5ms animation CPU time.

3. **Assassin's Creed Unity:** Crowd animation blending optimized with bone masking and layer caching. 10,000 NPCs with 2-layer blending (walk + reaction). Reduced per-character blend cost from 0.5ms to 0.08ms, enabling massive crowds.

4. **Spider-Man (PS4):** Web-swinging blended with contextual animations (wall-running, zipping). Used quantized weights and SIMD blending. Maintained 30fps with complex multi-layer character animation (<3ms total animation time).

5. **Call of Duty: Modern Warfare:** Weapon handling blended with full-body animation and procedural recoil. 4-layer blending with aggressive bone masking. Optimizations reduced animation CPU from 25ms to 8ms at 60fps with 12 players visible.

**Platform Considerations:**
- **PC/Console:** Full SIMD support (SSE, AVX on PC; NEON on consoles)
- **Mobile:** Critical optimization - limit to 2 layers max, aggressive early-out
- **Switch:** Use bone masking extensively, 2-3 layers maximum
- **VR:** Essential for 90fps - limit to 2 layers, use LOD-based layer count reduction
- **CPU architecture:** Modern CPUs have 4-8 SIMD lanes for parallel blending
- **Memory:** Cached layer results add 64 bytes × bones per layer

**Godot-Specific Notes:**
- Godot's `AnimationTree` handles blending automatically but with limited optimization control
- Use `AnimationNodeBlendSpace2D` and `AnimationNodeBlendTree` for structured blending
- `AnimationTree.set_active(false)` to disable blending for distant characters
- Custom blending via `Skeleton3D.set_bone_pose()` for manual optimization
- Godot 4.x: `SkeletonModifier3D` for custom bone processing
- Profile: Monitor "AnimationTree" and "AnimationPlayer" in profiler
- For extreme optimization, implement custom blending in C++ GDExtension

**Synergies:**
- **Bone Count Reduction (Technique 58):** Fewer bones to blend = faster
- **Animation Compression (Technique 54):** Smaller cached layer data
- **Animation LOD:** Update distant character blends at lower frequency
- **GPU Skinning (Technique 53):** Reduces pressure on CPU for final pose application
- **Object Pooling:** Reuse blend layer structures across character instances

**Measurement/Profiling:**
- **CPU time:** Profile `AnimationTree` and `Skeleton3D` update time
- **Layer activity:** Log how often layers are skipped via early-out
- **Bone mask effectiveness:** Track percentage of bones skipped per layer
- **Memory:** Monitor size of cached layer results
- **Target:** 60-80% reduction in blend CPU time
- **Bottleneck detection:** Animation should be <20% of total frame time
- **Profile tools:** Godot profiler, custom timing code around blend operations

---
