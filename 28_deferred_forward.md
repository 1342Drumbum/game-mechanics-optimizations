### 28. Deferred vs Forward Rendering

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Many-light scenarios bottleneck. Forward rendering: O(lights × fragments) = 100 lights × 2M pixels × 50 cycles = 10B cycles = 4000ms on mid-range GPU. Deferred rendering: O(fragments) + O(lights × pixels_affected) = 2M × 20 cycles + 100 lights × 50K pixels × 30 cycles = 190M cycles = 76ms (50x speedup for many lights).

**Technical Explanation:**
**Forward Rendering**: Renders geometry once, calculating all lighting per-fragment. Each object shaded with all affecting lights in single pass. Simple but inefficient for many lights (lighting calculated even for occluded pixels before depth test). Good for: few lights (<10), transparent objects, simple scenes. Cost: O(geometry × lights).

**Deferred Rendering**: Separates geometry from lighting. Pass 1: Render geometry to G-Buffer (position, normal, albedo, roughness, metallic). Pass 2: Full-screen pass applies all lights using G-Buffer data. Only shades visible pixels. Good for: many lights (100+), complex lighting, opaque scenes. Cost: O(geometry) + O(lights × affected_pixels). Drawbacks: No hardware MSAA, transparency requires separate pass, more memory (G-Buffer = 128-256 bits/pixel).

**Modern Approach**: Hybrid systems. Forward+ (tiled forward): Divides screen into tiles, culls lights per-tile. Clustered rendering: 3D grid of tiles for precise light assignment.

**Algorithmic Complexity:**
- Forward: O(n × l) where n=geometry, l=lights per object
- Deferred: O(n) + O(l × p) where p=pixels affected
- Forward+: O(n) + O(t × l) where t=tiles, much smaller than n×l
- Clustered: O(n) + O(c × l) where c=clusters, optimal light assignment

**Implementation Pattern:**
```gdscript
# Godot Forward vs Deferred Rendering Comparison
# Note: Godot 4.x uses Forward+ by default, but we can demonstrate concepts

class_name RenderingPathComparison
extends Node3D

enum RenderPath {
    FORWARD,
    DEFERRED,
    FORWARD_PLUS,
}

@export var current_path: RenderPath = RenderPath.FORWARD_PLUS
@export var light_count: int = 100
@export var use_shadow_atlas: bool = true

var viewport: Viewport
var environment: Environment

func _ready():
    viewport = get_viewport()
    _setup_rendering_path()

func _setup_rendering_path():
    # Godot 4.x automatically uses Forward+ (clustered forward)
    # We can configure it for different scenarios

    match current_path:
        RenderPath.FORWARD:
            _configure_forward()
        RenderPath.DEFERRED:
            _configure_deferred()
        RenderPath.FORWARD_PLUS:
            _configure_forward_plus()

func _configure_forward():
    # Simple forward rendering (limited lights)
    # Godot doesn't have explicit "forward" mode in 4.x
    # This simulates by limiting lights and disabling clustering

    ProjectSettings.set_setting("rendering/limits/cluster_builder/max_clustered_elements", 32)
    print("Configured: Simple Forward (limited lights)")

func _configure_deferred():
    # Deferred rendering simulation
    # Godot 4.x doesn't have traditional deferred, but we can approximate

    # Use GI for similar effect (precomputed lighting)
    # In a true deferred implementation, we'd use custom shaders
    print("Configured: Deferred-style rendering")

func _configure_forward_plus():
    # Forward+ (Tiled/Clustered Forward) - Godot 4.x default
    ProjectSettings.set_setting("rendering/limits/cluster_builder/max_clustered_elements", 512)

    # Configure cluster parameters
    # Godot automatically handles tiling and light clustering
    print("Configured: Forward+ (Clustered)")

# Custom Deferred Renderer (conceptual - requires custom RenderingServer usage)
class CustomDeferredRenderer:
    # G-Buffer textures
    var gbuffer_position: RID
    var gbuffer_normal: RID
    var gbuffer_albedo: RID
    var gbuffer_material: RID  # Roughness, Metallic, etc.
    var gbuffer_depth: RID

    var screen_size: Vector2i
    var rd: RenderingDevice

    func _init(size: Vector2i):
        screen_size = size
        rd = RenderingServer.get_rendering_device()
        if rd:
            _create_gbuffers()

    func _create_gbuffers():
        # Create G-Buffer render targets
        var format_rgba16f = RenderingDevice.DATA_FORMAT_R16G16B16A16_SFLOAT
        var format_rgba8 = RenderingDevice.DATA_FORMAT_R8G8B8A8_UNORM
        var format_depth = RenderingDevice.DATA_FORMAT_D32_SFLOAT

        # Position buffer (world space position)
        gbuffer_position = _create_texture(screen_size, format_rgba16f)

        # Normal buffer (world space normal)
        gbuffer_normal = _create_texture(screen_size, format_rgba16f)

        # Albedo buffer (base color)
        gbuffer_albedo = _create_texture(screen_size, format_rgba8)

        # Material buffer (roughness, metallic, AO, etc.)
        gbuffer_material = _create_texture(screen_size, format_rgba8)

        # Depth buffer
        gbuffer_depth = _create_texture(screen_size, format_depth)

    func _create_texture(size: Vector2i, format: int) -> RID:
        if not rd:
            return RID()

        var texture_format = RDTextureFormat.new()
        texture_format.width = size.x
        texture_format.height = size.y
        texture_format.format = format
        texture_format.usage_bits = RenderingDevice.TEXTURE_USAGE_SAMPLING_BIT | \
                                     RenderingDevice.TEXTURE_USAGE_COLOR_ATTACHMENT_BIT

        var view = RDTextureView.new()
        return rd.texture_create(texture_format, view, [])

    func render_geometry_pass(objects: Array):
        # Pass 1: Render geometry to G-Buffer
        # Each object writes: position, normal, albedo, material properties
        pass

    func render_lighting_pass(lights: Array):
        # Pass 2: Full-screen lighting pass
        # Read G-Buffer, accumulate lighting for all lights
        pass

# Forward+ Implementation (conceptual)
class ForwardPlusRenderer:
    var tile_size: Vector2i = Vector2i(16, 16)
    var light_grid: Array = []  # 2D grid of light lists
    var screen_tiles: Vector2i

    func _init(screen_size: Vector2i):
        screen_tiles = Vector2i(
            (screen_size.x + tile_size.x - 1) / tile_size.x,
            (screen_size.y + tile_size.y - 1) / tile_size.y
        )
        _initialize_light_grid()

    func _initialize_light_grid():
        light_grid.resize(screen_tiles.x * screen_tiles.y)
        for i in range(light_grid.size()):
            light_grid[i] = []

    func cull_lights_to_tiles(lights: Array, camera: Camera3D):
        # Reset grid
        for tile in light_grid:
            tile.clear()

        # For each light, determine which tiles it affects
        for light in lights:
            var affected_tiles = _calculate_light_tiles(light, camera)

            for tile_idx in affected_tiles:
                light_grid[tile_idx].append(light)

    func _calculate_light_tiles(light: Light3D, camera: Camera3D) -> Array:
        var affected = []

        # Project light bounds to screen space
        var light_pos = light.global_position
        var light_range = _get_light_range(light)

        # Simplified: Test all tiles (real implementation uses bounding volume)
        for ty in range(screen_tiles.y):
            for tx in range(screen_tiles.x):
                if _tile_intersects_light(tx, ty, light_pos, light_range, camera):
                    affected.append(ty * screen_tiles.x + tx)

        return affected

    func _tile_intersects_light(tile_x: int, tile_y: int, light_pos: Vector3, range: float, camera: Camera3D) -> bool:
        # Calculate tile frustum in world space
        # Test if light sphere intersects frustum
        # Simplified for now
        return true

    func _get_light_range(light: Light3D) -> float:
        if light is OmniLight3D:
            return light.omni_range
        elif light is SpotLight3D:
            return light.spot_range
        return 100.0  # Default

# Clustered Rendering (3D grid)
class ClusteredRenderer:
    var cluster_size: Vector3i = Vector3i(16, 16, 16)
    var cluster_grid: Array = []  # 3D grid of light lists
    var num_clusters: Vector3i
    var near_plane: float
    var far_plane: float

    func _init(screen_size: Vector2i, p_near: float, p_far: float):
        near_plane = p_near
        far_plane = p_far

        num_clusters = Vector3i(
            (screen_size.x + cluster_size.x - 1) / cluster_size.x,
            (screen_size.y + cluster_size.y - 1) / cluster_size.y,
            cluster_size.z
        )

        var total_clusters = num_clusters.x * num_clusters.y * num_clusters.z
        cluster_grid.resize(total_clusters)

        for i in range(total_clusters):
            cluster_grid[i] = []

    func build_light_clusters(lights: Array, camera: Camera3D):
        # Reset clusters
        for cluster in cluster_grid:
            cluster.clear()

        # Assign lights to clusters
        for light in lights:
            var affected = _calculate_light_clusters(light, camera)

            for cluster_idx in affected:
                cluster_grid[cluster_idx].append(light)

    func _calculate_light_clusters(light: Light3D, camera: Camera3D) -> Array:
        # Determine which 3D clusters the light affects
        # More precise than 2D tiles - handles depth
        var affected = []

        # Simplified implementation
        return affected

    func get_cluster_index(screen_pos: Vector2, depth: float) -> int:
        # Convert screen position and depth to cluster index
        var tile_x = int(screen_pos.x / cluster_size.x)
        var tile_y = int(screen_pos.y / cluster_size.y)

        # Logarithmic depth slicing for better distribution
        var depth_normalized = (depth - near_plane) / (far_plane - near_plane)
        var depth_slice = int(log(depth_normalized * (cluster_size.z - 1) + 1) / log(cluster_size.z))

        depth_slice = clamp(depth_slice, 0, cluster_size.z - 1)

        return (depth_slice * num_clusters.y + tile_y) * num_clusters.x + tile_x

# Performance comparison tool
class RenderPathBenchmark:
    func compare_paths(scene_config: Dictionary) -> Dictionary:
        var results = {}

        # Simulate different rendering paths
        results["forward"] = _benchmark_forward(scene_config)
        results["deferred"] = _benchmark_deferred(scene_config)
        results["forward_plus"] = _benchmark_forward_plus(scene_config)

        return results

    func _benchmark_forward(config: Dictionary) -> Dictionary:
        var lights = config.get("lights", 10)
        var objects = config.get("objects", 1000)
        var pixels = config.get("pixels", 1920 * 1080)

        # Simplified cost model
        var cost = objects * lights * 50  # 50 cycles per object per light

        return {
            "cost_cycles": cost,
            "time_ms": cost / 2500000.0,  # 2.5 GHz GPU
            "best_for": "Few lights (<10), transparency, MSAA"
        }

    func _benchmark_deferred(config: Dictionary) -> Dictionary:
        var lights = config.get("lights", 10)
        var objects = config.get("objects", 1000)
        var pixels = config.get("pixels", 1920 * 1080)

        # G-Buffer pass + lighting pass
        var gbuffer_cost = objects * 100  # Write G-Buffer
        var lighting_cost = pixels * lights * 0.1  # Per-pixel lighting

        var total_cost = gbuffer_cost + lighting_cost

        return {
            "cost_cycles": total_cost,
            "time_ms": total_cost / 2500000.0,
            "best_for": "Many lights (50+), opaque geometry, complex lighting"
        }

    func _benchmark_forward_plus(config: Dictionary) -> Dictionary:
        var lights = config.get("lights", 10)
        var objects = config.get("objects", 1000)
        var pixels = config.get("pixels", 1920 * 1080)

        var tiles = (1920 / 16) * (1080 / 16)  # 16x16 tiles

        # Render pass + per-tile lighting
        var render_cost = objects * 50
        var culling_cost = lights * tiles * 5  # Light culling per tile
        var lighting_cost = objects * 3 * 30  # Average 3 lights per object

        var total_cost = render_cost + culling_cost + lighting_cost

        return {
            "cost_cycles": total_cost,
            "time_ms": total_cost / 2500000.0,
            "best_for": "Medium-many lights (10-100), modern hardware, flexibility"
        }
```

**Key Parameters:**
- **Forward**: Max lights per object: 4-8, use forward for <10 total lights
- **Deferred**: G-Buffer size: 128-256 bits/pixel, supports 100+ lights efficiently
- **Forward+**: Tile size: 8×8 to 32×32 pixels (16×16 typical), max 512 lights
- **Clustered**: Cluster grid: 16×16×16 typical, logarithmic depth slices

**Edge Cases:**
- **Transparency**: Deferred requires separate forward pass (major limitation)
- **MSAA**: Hardware MSAA doesn't work with deferred (use FXAA/TAA instead)
- **Bandwidth**: Deferred uses more bandwidth (G-Buffer reads/writes)
- **Skinned meshes**: Same cost in all paths (vertex processing)
- **Many materials**: Forward can batch better, deferred always full-screen

**When NOT to Use:**
- **Deferred**: Heavy transparency, need MSAA, limited memory/bandwidth
- **Forward**: Many lights (>20), complex per-pixel lighting
- **Forward+**: Very simple scenes (<10 lights), older hardware

**Examples from Shipped Games:**

1. **Battlefield 3/4**: Deferred rendering, 100+ dynamic lights, 30-60fps. G-Buffer: Position (RGB16F), Normal (RGB10), Albedo (RGB8), Material (RGBA8) = 136 bits/pixel. Enabled dense urban lighting.

2. **DOOM (2016)**: Forward+ with clustered shading, 500+ lights in some scenes, 60fps. 16×16×16 clusters, logarithmic depth. Better than deferred for id Tech's requirements (lots of particles/transparency).

3. **Uncharted 4**: Forward+ on PS4, custom clustered system. 50-100 lights typical, maintained 30fps. Reduced G-Buffer memory (PS4 limited), allowed MSAA for quality.

4. **Detroit: Become Human**: Deferred rendering, complex indoor lighting. 200+ lights in some scenes, 30fps on PS4. G-Buffer optimizations: packed normals, shared Z-buffer.

5. **Fortnite**: Forward+ (originally deferred), switched for better performance on all platforms. Mobile-friendly, supports wide hardware range. 30-120fps depending on platform.

**Platform Considerations:**
- **PC High-End**: Deferred or Forward+ both excellent, choose based on needs
- **PC Low-End**: Forward+ better (less memory, scales down gracefully)
- **Console (PS5/XSX):** Forward+ preferred (Godot default), excellent performance
- **Last-Gen Console**: Forward+ or limited deferred, memory constraints
- **Mobile**: Forward or simple Forward+ (tile-based GPUs benefit less from deferred)
- **VR**: Forward+ excellent (need MSAA-like quality, but more performance-friendly)

**Godot-Specific Notes:**
- **Godot 4.x**: Uses Forward+ (clustered forward) by default - best of both worlds
- **Godot 3.x**: Forward rendering only (limited lights)
- Forward+ automatically handles light culling (no manual work needed)
- `rendering/limits/cluster_builder/*` settings control clustering
- Godot's Forward+ supports 512+ lights efficiently
- No traditional deferred mode (Forward+ is superior for most cases)
- Transparent objects automatically use forward pass

**Synergies:**
- Forward+ with **Light Culling** (built-in clustering)
- Deferred with **Screen-Space Effects** (G-Buffer provides data)
- Both with **Shadow Mapping** (independent technique)
- Forward+ with **MSAA** (hardware support maintained)
- Deferred with **Decal Systems** (easy G-Buffer manipulation)

**Measurement/Profiling:**
- GPU profiler: Geometry pass vs lighting pass time
- Track light counts: How many lights in scene?
- Measure bandwidth: G-Buffer writes (deferred) vs light calculations (forward)
- Visual debug: Show which lights affect each pixel
- Target: <5ms lighting time for 50 lights (Forward+), <10ms for 200 lights (Deferred)
- Bottleneck: Forward = vertex shader × lights, Deferred = bandwidth + fragment shader

**Additional Resources:**
- GDC 2010: "Deferred Rendering in Battlefield 3"
- Siggraph 2012: "Forward+: Bringing Deferred Lighting to the Next Level"
- GDC 2016: "Optimizing the Graphics Pipeline with Compute" (DOOM)
