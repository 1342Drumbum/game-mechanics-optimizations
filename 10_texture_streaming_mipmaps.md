### 10. Texture Streaming & Mipmaps

**Category:** Memory & Data Management

**Problem It Solves:** Loading full-resolution textures for all objects wastes 70-85% of VRAM on textures never seen at high detail. A 4K texture (16MB) viewed from distance only needs 256×256 mipmap (256KB), wasting 15.75MB (98%) of memory. In open-world games with thousands of textures, this leads to VRAM exhaustion, causing texture thrashing (loading from disk mid-frame), resulting in 50-200ms stutter spikes.

**Technical Explanation:**
Texture streaming dynamically loads/unloads texture mipmap levels based on distance from camera and screen-space size. Mipmaps are pre-generated texture pyramid (1/4 size per level). System calculates required detail level using texture size, distance, and FOV. High-priority textures (near camera) stream in highest mips first. Low-priority textures remain at low mips or unload entirely. Uses asynchronous I/O to avoid blocking main thread. Distance-based LOD: Mip 0 = 0-10m, Mip 1 = 10-25m, Mip 2 = 25-50m, Mip 3+ = 50m+.

**Algorithmic Complexity:**
- Texture lookup: O(1) regardless of streaming status (GPU handles missing mips)
- Streaming decision: O(n) per frame where n = visible textures (typically 100-500)
- Mip calculation: O(1) per texture using distance formula
- I/O streaming: Asynchronous, non-blocking background threads

**Implementation Pattern:**
```gdscript
class_name TextureStreamingManager
extends Node

const MAX_VRAM_MB: int = 2048  # Target VRAM budget
const STREAM_PRIORITY_LEVELS: int = 4
const BYTES_PER_FRAME: int = 16 * 1024 * 1024  # 16MB/frame streaming budget

var _texture_registry: Dictionary = {}  # Texture -> StreamingInfo
var _streaming_queue: Array[StreamingRequest] = []
var _current_vram_usage: int = 0

class StreamingInfo:
    var texture_path: String
    var current_mip_level: int = 4  # Start at lowest quality
    var target_mip_level: int = 4
    var priority: float = 0.0
    var size_bytes: int = 0
    var last_visible_frame: int = 0

class StreamingRequest:
    var info: StreamingInfo
    var target_mip: int
    var priority: float

func _ready() -> void:
    # Register all streaming textures in scene
    for node in get_tree().get_nodes_in_group("streaming_textures"):
        _register_texture(node)

func _process(_delta: float) -> void:
    _update_streaming_priorities()
    _process_streaming_queue()
    _evict_unused_textures()

func _update_streaming_priorities() -> void:
    var camera = get_viewport().get_camera_3d()
    if not camera:
        return

    var camera_pos = camera.global_position

    for node in get_tree().get_nodes_in_group("streaming_textures"):
        if not node.visible:
            continue

        var info = _texture_registry.get(node)
        if not info:
            continue

        # Calculate required mip level based on distance and screen size
        var distance = node.global_position.distance_to(camera_pos)
        var screen_size = _calculate_screen_size(node, camera)
        var required_mip = _calculate_required_mip(distance, screen_size)

        info.target_mip_level = required_mip
        info.priority = _calculate_priority(distance, screen_size)
        info.last_visible_frame = Engine.get_frames_drawn()

        # Queue streaming request if mip level needs to change
        if info.current_mip_level != required_mip:
            var request = StreamingRequest.new()
            request.info = info
            request.target_mip = required_mip
            request.priority = info.priority
            _streaming_queue.append(request)

    # Sort queue by priority (highest first)
    _streaming_queue.sort_custom(func(a, b): return a.priority > b.priority)

func _calculate_required_mip(distance: float, screen_size: float) -> int:
    # Formula: Closer objects and larger screen size need higher detail (lower mip)
    var pixels_per_meter = screen_size / max(distance, 1.0)

    # Mip 0 = >100 pixels/meter, Mip 1 = 50-100, Mip 2 = 25-50, etc.
    var mip_level = clamp(int(-log(pixels_per_meter / 100.0) / log(2.0)), 0, 4)
    return mip_level

func _calculate_screen_size(node: Node3D, camera: Camera3D) -> float:
    # Estimate screen-space size based on distance and bounds
    var aabb = node.get_aabb() if node.has_method("get_aabb") else AABB(Vector3.ZERO, Vector3.ONE)
    var radius = aabb.size.length() / 2.0
    var distance = node.global_position.distance_to(camera.global_position)

    # Simple screen-space size approximation
    var fov_factor = tan(deg_to_rad(camera.fov / 2.0))
    var screen_height = get_viewport().size.y
    var screen_size = (radius / distance) * screen_height / fov_factor

    return screen_size

func _calculate_priority(distance: float, screen_size: float) -> float:
    # Priority = (screen_size / distance) - closer & larger = higher priority
    return screen_size / max(distance, 1.0)

func _process_streaming_queue() -> void:
    var bytes_streamed = 0

    while not _streaming_queue.is_empty() and bytes_streamed < BYTES_PER_FRAME:
        var request = _streaming_queue.pop_front()

        # Calculate size difference
        var current_size = _calculate_mip_size(request.info.size_bytes, request.info.current_mip_level)
        var target_size = _calculate_mip_size(request.info.size_bytes, request.target_mip)
        var size_delta = target_size - current_size

        # Check VRAM budget
        if size_delta > 0 and _current_vram_usage + size_delta > MAX_VRAM_MB * 1024 * 1024:
            continue  # Skip if would exceed budget

        # Stream texture (simulate - actual implementation would use ResourceLoader)
        _stream_texture_mip(request.info, request.target_mip)
        request.info.current_mip_level = request.target_mip
        _current_vram_usage += size_delta
        bytes_streamed += abs(size_delta)

func _stream_texture_mip(info: StreamingInfo, mip_level: int) -> void:
    # In production, this would:
    # 1. Load mip level from disk asynchronously (ResourceLoader.load_threaded_request)
    # 2. Upload to GPU when ready
    # 3. Update material to use new texture
    pass

func _calculate_mip_size(base_size: int, mip_level: int) -> int:
    # Each mip level is 1/4 the size of previous (half width, half height)
    return base_size / int(pow(4, mip_level))

func _evict_unused_textures() -> void:
    # Unload textures not seen in last 120 frames (2 seconds at 60fps)
    var current_frame = Engine.get_frames_drawn()
    var eviction_threshold = 120

    for info in _texture_registry.values():
        if current_frame - info.last_visible_frame > eviction_threshold:
            if info.current_mip_level < 4:
                # Downgrade to lowest mip to save VRAM
                var old_size = _calculate_mip_size(info.size_bytes, info.current_mip_level)
                info.current_mip_level = 4
                var new_size = _calculate_mip_size(info.size_bytes, 4)
                _current_vram_usage -= (old_size - new_size)

func _register_texture(node: Node) -> void:
    var info = StreamingInfo.new()
    info.texture_path = node.texture.resource_path if node.get("texture") else ""
    info.size_bytes = 16 * 1024 * 1024  # Example: 4K texture = ~16MB
    _texture_registry[node] = info
```

**Key Parameters:**
- VRAM Budget: 1-4GB depending on platform (mobile: 512MB-1GB, desktop: 2-8GB)
- Streaming Bandwidth: 10-50MB/frame (depends on storage speed: SSD vs HDD)
- Mip Levels: 4-6 levels typical (4K → 2K → 1K → 512 → 256 → 128)
- Priority Threshold: Stream textures >50 pixels on screen first
- Eviction Time: Unload textures not seen for 2-5 seconds

**Edge Cases:**
- Rapid camera movement: Predictive streaming based on camera velocity
- Teleportation: Immediate stream all nearby textures (may cause hitch)
- Texture thrashing: Too aggressive eviction causes reload loops
- SSD vs HDD: Adjust streaming bandwidth based on storage speed
- First frame: All textures at low mip until streaming catches up

**When NOT to Use:**
- Small games with <500MB total textures (all fit in VRAM)
- Fixed camera games (no distance variation)
- Stylized/low-poly games with small textures
- VR games requiring max quality (causes visible pop-in)

**Examples from Shipped Games:**

1. **Unreal Engine 5 (Nanite/Virtual Texturing):** Handles 100GB+ texture data, only 4-8GB in VRAM, enables photorealistic open worlds
2. **Unity Viking Village Demo:** 25-30% VRAM savings with texture streaming, 8GB → 5.6GB texture memory
3. **Forza Horizon 5:** Streams 100GB+ of world textures, maintains consistent 60fps with 4GB VRAM budget
4. **Microsoft Flight Simulator:** Streams terrain textures from cloud (2PB total data), only 2-4GB local cache, enables planet-scale detail
5. **Gears of War (Xbox 360):** Pioneered texture streaming on console with 512MB total RAM, enabled high-res textures impossible otherwise

**Platform Considerations:**
- **PC (Desktop):** Aggressive streaming, 2-8GB VRAM budgets, fast SSD enables 50MB/frame streaming
- **Mobile (iOS/Android):** Critical optimization, 512MB-2GB VRAM, slower storage requires 5-10MB/frame limit
- **Consoles (PS5/Xbox):** Built-in I/O decompression hardware, 100GB/s+ streaming possible with DirectStorage
- **VR:** Challenging due to visible pop-in, consider pre-loading nearby areas, needs higher base mip levels

**Godot-Specific Notes:**
- Godot 4.x lacks built-in texture streaming system (Unreal/Unity have it)
- Implement custom system using ResourceLoader.load_threaded_request()
- Use RenderingServer for low-level texture upload control
- Consider preloading common textures in _ready() to avoid first-frame hitches
- Compress textures with VRAM compression (BC7/ASTC) to reduce streaming size
- Use `Image.generate_mipmaps()` to create mipmap chain at import time

**Synergies:**
- Essential for Open World games with Level-of-Detail (LOD) Systems
- Pairs with Frustum Culling (only stream visible textures)
- Combines with Occlusion Culling (don't stream occluded objects)
- Works with Object Pooling (stream textures as pooled objects activate)
- Enables Virtual Texturing for terrain (mega-textures)

**Measurement/Profiling:**
```gdscript
# Monitor VRAM usage
func _profile_vram():
    var vram_used = Performance.get_monitor(Performance.RENDER_VIDEO_MEM_USED)
    var vram_mb = vram_used / 1024.0 / 1024.0
    print("VRAM Used: %.2f MB" % vram_mb)

    var texture_mem = Performance.get_monitor(Performance.RENDER_TEXTURE_MEM_USED)
    var texture_mb = texture_mem / 1024.0 / 1024.0
    print("Texture Memory: %.2f MB" % texture_mb)

# Track streaming hitches
func _detect_streaming_hitch(frame_time_ms: float):
    if frame_time_ms > 20.0:  # 16.67ms = 60fps target
        push_warning("Streaming hitch detected: %.2f ms" % frame_time_ms)
```

**Target Metrics:**
- VRAM usage: 50-70% of budget (leaves headroom for spikes)
- Streaming bandwidth: <10% of frame time budget (<1.6ms at 60fps)
- Texture pop-in: Should occur >50m from camera (not visible)
- Mip transitions: Smooth, not visible to player (use trilinear filtering)
