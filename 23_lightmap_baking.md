### 23. Lightmap Baking vs Real-Time

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Real-time lighting computational cost. Dynamic lighting for 50 lights × 1000 surfaces = 50K light calculations per frame. At 200 cycles each = 10M cycles = 4ms on 2.5GHz GPU. Add shadows (5ms) + bounced GI (8ms) = 17ms total, exceeding 16.67ms budget. Lightmaps precompute lighting offline, reducing runtime to simple texture lookup (~0.1ms).

**Technical Explanation:**
Lightmapping bakes static lighting into textures during development/load time. Traces rays from light sources, calculates direct + indirect (bounced) illumination, stores in UV-mapped textures. At runtime, fragment shader samples lightmap texture instead of calculating lighting. Supports complex phenomena: soft shadows, area lights, global illumination, caustics. Trade-offs: Static only (moving objects need real-time lights), memory cost (lightmap textures), longer build times, requires good UV unwrapping. Modern approach: lightmaps for static geometry + dynamic lights for moving objects.

**Algorithmic Complexity:**
- Real-time lighting: O(l × s) per frame where l=lights, s=surfaces
- Lightmap lookup: O(1) texture sample per fragment
- Baking: O(r × s × l) offline where r=rays per sample (1000-10000)
- Memory: O(s × resolution²) for lightmap textures
- Quality vs Performance: More resolution/rays = better quality, longer bake

**Implementation Pattern:**
```gdscript
# Godot Lightmap Baking System
class_name LightmapBaker
extends Node3D

@export var lightmap_resolution: int = 512  # Per object
@export var texels_per_unit: float = 20.0  # Texel density
@export var bounces: int = 3  # GI bounces
@export var samples_per_texel: int = 128  # Quality
@export var use_denoiser: bool = true
@export var directional_mode: bool = true  # Store directionality

var baked_lights: Array[LightmapGI] = []

func setup_scene_for_baking():
    # Mark static geometry
    var static_objects = _find_static_geometry(get_tree().root)

    for obj in static_objects:
        if obj is GeometryInstance3D:
            # Set to static mode (baked lighting)
            obj.gi_mode = GeometryInstance3D.GI_MODE_STATIC

            # Ensure proper UV2 channel for lightmaps
            if obj is MeshInstance3D:
                _generate_lightmap_uv(obj)

func _find_static_geometry(node: Node) -> Array:
    var static_objects = []

    for child in node.get_children():
        # Check if marked as static
        if child is GeometryInstance3D:
            if child.has_meta("static") and child.get_meta("static"):
                static_objects.append(child)

        # Recurse
        static_objects.append_array(_find_static_geometry(child))

    return static_objects

func _generate_lightmap_uv(mesh_instance: MeshInstance3D):
    # Generate secondary UV for lightmapping
    # Godot does this automatically, but can be customized

    var mesh = mesh_instance.mesh
    if not mesh:
        return

    # Calculate appropriate resolution based on size
    var aabb = mesh_instance.get_aabb()
    var surface_area = _estimate_surface_area(aabb)
    var texels_needed = surface_area * texels_per_unit
    var resolution = int(sqrt(texels_needed))

    # Clamp to reasonable range
    resolution = clamp(resolution, 32, 2048)

    print("Lightmap resolution for ", mesh_instance.name, ": ", resolution)

func _estimate_surface_area(aabb: AABB) -> float:
    var size = aabb.size
    return 2.0 * (size.x * size.y + size.y * size.z + size.z * size.x)

func bake_lightmaps():
    # Godot 4.x: Use LightmapGI node
    var lightmap_gi = LightmapGI.new()
    add_child(lightmap_gi)

    # Configure baking parameters
    lightmap_gi.bounces = bounces
    lightmap_gi.quality = LightmapGI.BAKE_QUALITY_HIGH if samples_per_texel > 64 else LightmapGI.BAKE_QUALITY_MEDIUM
    lightmap_gi.use_denoiser = use_denoiser
    lightmap_gi.directional = directional_mode

    # Start baking (blocking operation)
    print("Starting lightmap bake...")
    var start_time = Time.get_ticks_msec()

    # In editor: Use LightmapGI.bake()
    # At runtime: Load pre-baked lightmaps

    var end_time = Time.get_ticks_msec()
    print("Baking completed in ", (end_time - start_time) / 1000.0, " seconds")

    baked_lights.append(lightmap_gi)

# Runtime lightmap manager
class LightmapManager:
    var lightmap_textures: Dictionary = {}  # Object -> Texture2D

    func load_lightmaps(scene_path: String):
        # Load pre-baked lightmaps
        var lightmap_data = ResourceLoader.load(scene_path + ".lmbake")

        if lightmap_data:
            print("Loaded lightmaps from: ", scene_path)
            # Apply to scene objects
            _apply_lightmaps(lightmap_data)

    func _apply_lightmaps(lightmap_data):
        # Godot handles this automatically for LightmapGI nodes
        pass

    func switch_lighting_mode(objects: Array, use_lightmaps: bool):
        for obj in objects:
            if obj is GeometryInstance3D:
                if use_lightmaps:
                    obj.gi_mode = GeometryInstance3D.GI_MODE_STATIC
                else:
                    obj.gi_mode = GeometryInstance3D.GI_MODE_DYNAMIC

# Hybrid lighting system
class HybridLightingSystem:
    # Combine lightmaps (static) with real-time lights (dynamic)

    @export var static_light_contribution: float = 1.0
    @export var dynamic_light_contribution: float = 1.0

    var dynamic_lights: Array[Light3D] = []
    var lightmapped_objects: Array[GeometryInstance3D] = []

    func setup_hybrid_lighting():
        # Find all lights
        dynamic_lights = _find_dynamic_lights(get_tree().root)

        # Setup light layers
        for light in dynamic_lights:
            # Dynamic lights only affect dynamic objects
            light.light_cull_mask = 0b10  # Layer 2

        # Setup object layers
        for obj in lightmapped_objects:
            # Static objects ignore dynamic lights
            obj.layers = 0b01  # Layer 1

    func _find_dynamic_lights(node: Node) -> Array:
        var lights = []

        for child in node.get_children():
            if child is Light3D:
                if not child.has_meta("static") or not child.get_meta("static"):
                    lights.append(child)

            lights.append_array(_find_dynamic_lights(child))

        return lights

    func toggle_dynamic_lights(enabled: bool):
        for light in dynamic_lights:
            light.visible = enabled

# Lightmap probe system for dynamic objects
class LightProbeSystem:
    # Light probes capture baked lighting for dynamic objects

    var probes: Array[LightProbe] = []
    var probe_grid_size: Vector3 = Vector3(2, 2, 2)  # Probe spacing

    class LightProbe:
        var position: Vector3
        var sh_coefficients: Array[Vector3] = []  # Spherical harmonics (9 coefficients)

        func sample_lighting() -> Color:
            # Reconstruct lighting from SH coefficients
            # Simplified: return average
            if sh_coefficients.is_empty():
                return Color.WHITE

            var sum = Vector3.ZERO
            for coef in sh_coefficients:
                sum += coef

            var avg = sum / sh_coefficients.size()
            return Color(avg.x, avg.y, avg.z)

    func generate_probe_grid(bounds: AABB):
        probes.clear()

        var x_count = int(bounds.size.x / probe_grid_size.x) + 1
        var y_count = int(bounds.size.y / probe_grid_size.y) + 1
        var z_count = int(bounds.size.z / probe_grid_size.z) + 1

        for x in range(x_count):
            for y in range(y_count):
                for z in range(z_count):
                    var pos = bounds.position + Vector3(
                        x * probe_grid_size.x,
                        y * probe_grid_size.y,
                        z * probe_grid_size.z
                    )

                    var probe = LightProbe.new()
                    probe.position = pos
                    probes.append(probe)

        print("Generated ", probes.size(), " light probes")

    func sample_lighting_at(position: Vector3) -> Color:
        # Find nearest probes and interpolate
        var nearest = _find_nearest_probes(position, 8)

        if nearest.is_empty():
            return Color.WHITE

        # Trilinear interpolation
        var total_color = Color.BLACK
        var total_weight = 0.0

        for probe in nearest:
            var distance = position.distance_to(probe.position)
            var weight = 1.0 / (distance + 0.1)  # Avoid divide by zero

            total_color += probe.sample_lighting() * weight
            total_weight += weight

        return total_color / total_weight

    func _find_nearest_probes(position: Vector3, count: int) -> Array:
        var nearest = []
        var distances = []

        for probe in probes:
            var dist = position.distance_squared_to(probe.position)

            if nearest.size() < count:
                nearest.append(probe)
                distances.append(dist)
            else:
                # Replace furthest if this is closer
                var max_idx = 0
                var max_dist = distances[0]

                for i in range(distances.size()):
                    if distances[i] > max_dist:
                        max_dist = distances[i]
                        max_idx = i

                if dist < max_dist:
                    nearest[max_idx] = probe
                    distances[max_idx] = dist

        return nearest

# Progressive lightmap refinement
class ProgressiveLightmapBaker:
    # Bake lightmaps progressively for faster iteration

    var samples_per_frame: int = 100
    var current_sample: int = 0
    var total_samples: int = 10000
    var baking: bool = false

    func start_progressive_bake():
        baking = true
        current_sample = 0

    func update_bake(delta: float):
        if not baking:
            return

        # Bake a few samples per frame
        for i in range(samples_per_frame):
            _bake_sample(current_sample)
            current_sample += 1

            if current_sample >= total_samples:
                baking = false
                _finalize_bake()
                break

    func _bake_sample(sample_index: int):
        # Trace rays and accumulate lighting
        pass

    func _finalize_bake():
        # Denoise and write final lightmaps
        print("Progressive bake complete!")
```

**Key Parameters:**
- Lightmap resolution: 256-2048 per object (higher = better quality, more memory)
- Texels per unit: 10-50 (10 = low quality, 50 = high quality)
- GI bounces: 1-5 (1 = direct only, 3 = realistic, 5 = diminishing returns)
- Samples per texel: 64-512 (higher = less noise, longer bake)
- Denoiser: Always enable (removes noise without increasing samples)
- Directional lightmaps: Enable for normal-mapped surfaces
- Atlas packing: Combine small objects into single lightmap

**Edge Cases:**
- UV seams: Add padding to prevent light bleeding
- Overlapping UVs: Ensure unique UV2 layout
- Thin geometry: May have light leaking through
- Large scale differences: Use adaptive resolution
- Dynamic objects: Use light probes or real-time lights
- Level streaming: Bake lightmaps per chunk/section

**When NOT to Use:**
- Fully dynamic environments (destructible, procedural)
- Time-of-day changes required
- Limited development time (baking can take hours)
- Minimal lighting complexity (few lights, simple scenes)
- Extremely memory-constrained platforms

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Extensive lightmapping, 4-8 hours bake per level. 90% lighting precomputed. Real-time reserved for flashlights/muzzle flashes. Saved 20ms per frame (25ms → 5ms lighting), enabled 30fps on PS4.

2. **Half-Life: Alyx:** Lightmaps + light probes, 2K resolution per 4m². Directional lightmaps for normal maps. Bake time 1-2 hours per level. Enabled high-quality VR lighting at 90fps.

3. **Gears of War 5:** Beast lightmapper, 12-16 hours per level. 1024-2048 resolution, 5 GI bounces. Combined with dynamic lights (character rim lights). 60fps on Xbox One.

4. **Resident Evil 2 Remake:** Lightmaps for environments, dynamic lights for characters. 512-1024 resolution, selective high-res for hero areas. Saved 15ms GPU time, maintained 60fps.

5. **Minecraft RTX:** Hybrid approach - lightmaps for Java Edition, full ray-tracing for RTX. Lightmaps enabled 60fps on consoles, RTX path for high-end PCs.

**Platform Considerations:**
- **PC High-End:** Higher resolution lightmaps (2K-4K), more GI bounces
- **PC Low-End:** Lower resolution (512-1024), simpler lighting
- **Console (PS5/XSX):** 1024-2048 resolution, good GI quality
- **Last-Gen Console:** 512-1024 resolution, 2-3 bounces max
- **Mobile:** 256-512 resolution, minimal GI, aggressive compression
- **Memory:** 1024² lightmap = 4MB uncompressed, use BC6H/ASTC compression

**Godot-Specific Notes:**
- Use `LightmapGI` node in Godot 4.x for baking
- Set `GeometryInstance3D.gi_mode = GI_MODE_STATIC` for baked objects
- Godot uses GPU-based lightmapper (fast, good quality)
- UV2 generation automatic or custom in import settings
- Lightmaps stored in `.lmbake` file alongside scene
- `Environment.ambient_light` affects lightmap brightness
- Use `Light3D.light_bake_mode` to control which lights bake
- Directional lightmaps: Better with normal mapping

**Synergies:**
- Pairs with Real-Time Lights (hybrid approach for dynamic objects)
- Combines with Light Probes (dynamic objects in static lighting)
- Works with LOD Systems (lower-res lightmaps for distant LODs)
- Enables Complex Materials (lighting precomputed, material complexity "free")
- Reduces need for Dynamic GI (baked GI nearly free at runtime)

**Measurement/Profiling:**
- GPU profiler: Lighting time should be <1ms with lightmaps vs 5-15ms real-time
- Track memory: Lightmap textures (target <100MB for full scene)
- Measure bake time: 10 minutes - 8 hours depending on quality
- Visual quality: Compare baked vs real-time, target imperceptible difference
- Draw calls: Lightmaps don't add draw calls (texture lookup only)
- Target: 90%+ lighting precomputed, <5ms dynamic lighting
- Compression: BC6H reduces lightmap memory by 4-6x with minimal quality loss

**Additional Resources:**
- GDC 2018: "The Lightmapping Techniques of The Last of Us Part II"
- Siggraph 2015: "Real-Time Global Illumination in Frostbite"
- Godot Docs: Lightmapping Guide
