### 22. Z-Prepass & Early-Z Rejection

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Pixel shader overdraw waste. Dense scenes render same pixel 5-20 times (high overdraw). At 1920×1080 with 10x overdraw = 20.7M fragment shader invocations. Complex shader at 100 cycles = 2.07 billion cycles wasted. Z-prepass reduces overdraw to ~1.5x, saving 85% fragment shader cost (15-25ms on mid-range GPU).

**Technical Explanation:**
Z-prepass renders scene geometry with minimal shader (depth only) before main rendering pass. Populates depth buffer with nearest surface per pixel. Main pass then uses Early-Z hardware test: pixel shader only runs if fragment passes depth test (depth < buffer value). Modern GPUs have hierarchical-Z: tests blocks of pixels against depth buffer pyramid. Enables Early-Z rejection before expensive fragment shader. Trade-off: extra draw calls vs reduced fragment shader load. Beneficial when fragment shader cost > vertex shader cost × 2.

**Algorithmic Complexity:**
- No prepass: O(n × p × s) where n=objects, p=pixels, s=shader cost
- With prepass: O(n × v) + O(k × p × s) where k=visible fragments (1-2 instead of 5-20)
- Prepass cost: Simple shader (~5 cycles) vs complex shader (~100+ cycles)
- Savings: 70-90% fragment shader invocations eliminated
- Break-even: When fragment shader >2x vertex shader cost

**Implementation Pattern:**
```gdscript
# Godot Z-Prepass Implementation
class_name ZPrepassRenderer
extends Node3D

var depth_prepass_material: ShaderMaterial
var main_camera: Camera3D
var render_queue: Array = []

func _ready():
    _setup_prepass_shader()
    main_camera = get_viewport().get_camera_3d()

func _setup_prepass_shader():
    # Create minimal depth-only shader
    var shader = Shader.new()
    shader.code = """
shader_type spatial;
render_mode unshaded, depth_draw_always, depth_prepass_alpha;

void vertex() {
    POSITION = PROJECTION_MATRIX * MODELVIEW_MATRIX * vec4(VERTEX, 1.0);
}

void fragment() {
    // No color output, only depth
    ALBEDO = vec3(0.0);
    ALPHA = 1.0;
}
"""
    depth_prepass_material = ShaderMaterial.new()
    depth_prepass_material.shader = shader

func render_with_prepass(objects: Array):
    # Pass 1: Depth prepass
    _render_depth_prepass(objects)

    # Pass 2: Main color pass (hardware Early-Z active)
    _render_main_pass(objects)

func _render_depth_prepass(objects: Array):
    # Override materials with depth-only shader
    var material_cache = {}

    for obj in objects:
        if not obj is GeometryInstance3D:
            continue

        # Store original material
        material_cache[obj] = obj.material_override

        # Apply depth-only material
        obj.material_override = depth_prepass_material

    # Render frame (depth buffer populated)
    await RenderingServer.frame_post_draw

    # Restore original materials
    for obj in material_cache:
        obj.material_override = material_cache[obj]

func _render_main_pass(objects: Array):
    # Render with original materials
    # Early-Z will reject occluded fragments automatically
    pass

# Advanced: Selective Z-Prepass
class SelectiveZPrepass:
    # Only render large occluders in prepass
    var occluder_threshold: float = 10.0  # Surface area in m²

    func should_render_in_prepass(obj: GeometryInstance3D) -> bool:
        if not obj.visible:
            return false

        # Only render large, opaque objects
        var aabb = obj.get_aabb()
        var surface_area = _estimate_surface_area(aabb)

        # Skip if too small or transparent
        if surface_area < occluder_threshold:
            return false

        # Check if material is opaque
        var material = _get_material(obj)
        if material and _is_transparent(material):
            return false

        return true

    func _estimate_surface_area(aabb: AABB) -> float:
        var size = aabb.size
        return 2.0 * (size.x * size.y + size.y * size.z + size.z * size.x)

    func _get_material(obj: GeometryInstance3D):
        if obj.material_override:
            return obj.material_override
        if obj is MeshInstance3D and obj.mesh:
            return obj.mesh.surface_get_material(0)
        return null

    func _is_transparent(material: Material) -> bool:
        if material is StandardMaterial3D:
            return material.transparency != BaseMaterial3D.TRANSPARENCY_DISABLED
        # Check shader for transparency flags
        return false

# Hierarchical-Z optimization
class HierarchicalZOptimizer:
    # Exploit hardware Hi-Z for better Early-Z performance

    func optimize_render_order(objects: Array, camera: Camera3D) -> Array:
        # Sort front-to-back for optimal Early-Z
        var camera_pos = camera.global_position

        objects.sort_custom(func(a, b):
            var dist_a = a.global_position.distance_squared_to(camera_pos)
            var dist_b = b.global_position.distance_squared_to(camera_pos)
            return dist_a < dist_b
        )

        return objects

    func render_with_hiz(objects: Array):
        # Render in front-to-back order
        # Hardware Hi-Z will reject occluded fragments efficiently
        for obj in objects:
            if obj is VisualInstance3D:
                obj.visible = true
                # Rendering happens automatically

# Material-aware prepass
class MaterialAwarePrepass:
    var prepass_enabled: Dictionary = {}  # Material -> bool

    func analyze_materials(objects: Array):
        # Determine which materials benefit from prepass
        for obj in objects:
            if not obj is GeometryInstance3D:
                continue

            var material = _get_material(obj)
            if not material or material in prepass_enabled:
                continue

            var complexity = _estimate_shader_complexity(material)
            var should_prepass = complexity > 50.0  # Threshold

            prepass_enabled[material] = should_prepass

    func _estimate_shader_complexity(material: Material) -> float:
        # Estimate shader cost based on features
        var complexity = 0.0

        if material is StandardMaterial3D:
            complexity += 10.0  # Base cost

            if material.albedo_texture:
                complexity += 5.0

            if material.normal_enabled:
                complexity += 10.0

            if material.roughness_texture:
                complexity += 5.0

            if material.metallic_texture:
                complexity += 5.0

            if material.emission_enabled:
                complexity += 8.0

            if material.rim_enabled:
                complexity += 12.0

            if material.clearcoat_enabled:
                complexity += 15.0

            if material.anisotropy_enabled:
                complexity += 15.0

            if material.subsurface_scattering_enabled:
                complexity += 25.0

        return complexity

    func _get_material(obj: GeometryInstance3D):
        if obj.material_override:
            return obj.material_override
        if obj is MeshInstance3D and obj.mesh:
            return obj.mesh.surface_get_material(0)
        return null

# GPU-driven Hi-Z culling (advanced)
class GPUHiZCuller:
    # Use compute shader to test objects against Hi-Z buffer

    var hiz_buffer: RID
    var hiz_levels: int = 8

    func generate_hiz_pyramid():
        # Downsample depth buffer to create mipmap pyramid
        # Level 0: Full resolution
        # Level 1: 1/2 resolution
        # ...
        # Level 7: 1/128 resolution

        var rd = RenderingServer.get_rendering_device()
        if not rd:
            return

        # Use compute shader to generate mipmap levels
        # Each level stores maximum depth of 2×2 region
        pass

    func cull_objects_gpu(objects: Array) -> Array:
        # Test each object's AABB against Hi-Z buffer
        # If all corners behind any mip level, object is occluded

        var visible = []

        for obj in objects:
            if _test_aabb_against_hiz(obj.get_aabb()):
                visible.append(obj)

        return visible

    func _test_aabb_against_hiz(aabb: AABB) -> bool:
        # Project AABB corners to screen space
        # Test against appropriate Hi-Z mip level
        # Conservative test: visible if any corner visible
        return true  # Simplified

# Deferred rendering with Z-Prepass
class DeferredZPrepass:
    # In deferred rendering, geometry pass is essentially a Z-prepass

    func render_deferred_with_prepass():
        # Pass 1: G-Buffer (position, normal, albedo, roughness, etc.)
        # This naturally populates depth buffer

        # Pass 2: Lighting pass
        # Only shades visible pixels (depth buffer ensures this)

        # Early-Z is automatic in deferred rendering
        pass
```

**Key Parameters:**
- Fragment shader complexity threshold: >50 ALU ops = enable prepass
- Occluder size threshold: >10m² surface area for prepass
- Overdraw threshold: >3x = enable prepass
- Hi-Z pyramid levels: 6-8 levels (down to 4×4 pixels)
- Front-to-back sorting tolerance: ±10% distance (avoid constant resorting)
- Transparency handling: Skip transparent objects in prepass

**Edge Cases:**
- Alpha-tested materials: May need prepass (depend on shader cost)
- Transparent objects: Skip prepass entirely (order-dependent)
- Skinned meshes: Prepass may be expensive (double vertex processing)
- Tessellation: Prepass doubles tessellation cost
- Low overdraw: Prepass overhead exceeds benefit (<2x overdraw)
- Mobile: Tile-based renderers have different behavior

**When NOT to Use:**
- Simple shaders (<30 ALU ops)
- Low overdraw scenes (<2x)
- Tile-based GPUs (mobile) - built-in Early-Z more efficient
- Deferred rendering (G-buffer pass is already a prepass)
- Transparent-heavy scenes
- Vertex-bound scenarios

**Examples from Shipped Games:**

1. **Crysis 3:** Z-prepass for vegetation, reduced overdraw from 15x → 2x in forest scenes. Saved 18ms fragment shader time (25ms → 7ms). Enabled ultra-high quality foliage at 30fps on console.

2. **The Witcher 3:** Selective Z-prepass for large buildings/terrain. Average overdraw 8x → 1.8x in cities. Saved 12ms on PS4, essential for 30fps target. Skipped characters (skinned) to avoid double vertex cost.

3. **Battlefield 3:** Full Z-prepass in destruction scenarios. Debris overdraw 20x → 2.5x. Saved 20ms GPU time in worst-case. Critical for maintaining 30fps during building collapse.

4. **Uncharted 4:** Hi-Z culling for foliage, 85% fragments rejected before shader. Prepass cost 2ms, saved 15ms fragment shading. Enabled dense jungle environments at 30fps.

5. **DOOM (2016):** Forward+ with Z-prepass for large demons/geometry. Overdraw 6x → 1.5x. Prepass overhead 1.5ms, saved 10ms. Combined with clustered shading for complex lighting.

**Platform Considerations:**
- **PC Desktop:** Highly effective, hardware Hi-Z on all modern GPUs
- **Console (PS5/XSX):** Excellent Early-Z support, prepass recommended
- **Last-Gen Console:** Beneficial for complex shaders, 10-15ms savings typical
- **Mobile (Tile-based):** Less beneficial, on-chip Early-Z more efficient, avoid prepass
- **VR:** Critical for performance, reduces fragment load by 80%+
- **Bandwidth:** Prepass increases geometry bandwidth, but saves fragment bandwidth

**Godot-Specific Notes:**
- Godot doesn't expose explicit Z-prepass control
- Forward+ renderer: automatic depth prepass for opaque geometry
- Use `render_mode depth_draw_always` in shaders
- Depth prepass happens automatically for opaque materials
- Transparent materials rendered after depth prepass
- `depth_prepass_alpha` for alpha-tested materials in prepass
- Godot 4.x: Forward+ has built-in depth prepass optimization
- Manual control: Render scene twice with shader override

**Synergies:**
- Essential for Forward+ Rendering (enables efficient light culling)
- Pairs with Occlusion Culling (depth buffer for queries)
- Combines with Hi-Z Culling (mipmap pyramid for fast tests)
- Enables Screen-Space Effects (high-quality depth buffer)
- Works with Shadow Mapping (shared depth techniques)

**Measurement/Profiling:**
- GPU profiler: Measure fragment shader invocations (should decrease 70-90%)
- Track overdraw: Use GPU debugger overdraw visualization
- Measure prepass cost: Should be <20% of total frame time
- Monitor bandwidth: Depth buffer writes vs fragment shader reads
- Target: <2x overdraw after prepass, 80%+ fragment rejection
- Break-even: Prepass overhead <10% of fragment shader savings
- Visual debug: Color pixels by number of fragment shader invocations

**Additional Resources:**
- GDC 2011: "Battlefield 3 - Rendering Architecture"
- GPU Gems 2: Chapter 6 "Hardware Occlusion Queries Made Useful"
- Siggraph 2015: "The Rendering Technologies of Crysis 3"
