### 31. Mipmapping Optimization

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Texture sampling performance and quality. Sampling 4K texture for distant 4-pixel object reads random 4K² cache locations = 60-100 cycles per sample × 4 pixels = 240-400 cycles wasted. Mipmaps provide pre-filtered lower resolutions, sampling 4×4 mip reads contiguous cache = 4-8 cycles per sample, saving 98% of texture bandwidth (critical performance multiplier).

**Technical Explanation:**
Mipmaps are pre-calculated texture LODs at powers-of-2 resolutions (4096→2048→1024→512→...→1). GPU automatically selects mip level based on screen-space derivative (how much texture coord changes per pixel). Eliminates aliasing (Moiré patterns), improves cache coherency, reduces memory bandwidth by 30-70%. Storage cost: +33% (1 + 1/4 + 1/16 + ... = 4/3). Modern GPUs require mipmaps for performance; without them, texture sampling can be 10-100x slower. Anisotropic filtering samples multiple mip levels for quality.

**Algorithmic Complexity:**
- No mipmaps: O(n) random texture reads, poor cache utilization
- With mipmaps: O(log n) smaller texture, sequential reads, excellent cache hit rate
- Bandwidth savings: 50-70% typical (distant objects use tiny mips)
- Storage overhead: +33% texture memory (well worth it)
- Quality improvement: Eliminates temporal aliasing/flickering

**Implementation Pattern:**
```gdscript
# Godot Mipmapping Configuration
class_name MipmapOptimizer
extends Node

func configure_texture_import(texture_path: String, options: Dictionary):
    # In Godot, mipmaps configured in import settings

    # Example configuration
    var import_options = {
        "mipmaps/generate": true,  # Always enable for 3D textures
        "mipmaps/limit": -1,  # -1 = no limit, or set max mip level
        "roughness/mode": 0,  # Roughness affects mip generation
        "compress/mode": 2,  # Compression mode (affects mip quality)
    }

    # Apply settings
    # This would be done in .import file or import dock

func set_material_mipmap_bias(material: Material, bias: float):
    # Mipmap bias: negative = sharper (use higher res mips), positive = blurrier (use lower res mips)

    if material is StandardMaterial3D:
        # Godot doesn't expose mipmap bias directly on materials
        # Would need custom shader parameter
        pass

# Custom shader with mipmap control
const MIPMAP_SHADER = """
shader_type spatial;

uniform sampler2D albedo_texture : source_color, filter_linear_mipmap;
uniform float mipmap_bias = 0.0;  // -2.0 to 2.0
uniform float mipmap_scale = 1.0;

void fragment() {
    // Explicit mipmap level selection
    vec4 color = textureLod(albedo_texture, UV, mipmap_bias);

    // Or use automatic with bias
    // vec4 color = texture(albedo_texture, UV, mipmap_bias);

    ALBEDO = color.rgb;
}
"""

# Streaming mipmaps (load high-res on demand)
class MipmapStreamer:
    var base_textures: Dictionary = {}  # Low-res always loaded
    var highres_cache: Dictionary = {}  # High-res loaded on demand
    var cache_size_mb: float = 100.0
    var current_cache_mb: float = 0.0

    func request_texture(texture_name: String, required_mip: int) -> Texture2D:
        # Check if high-res version needed
        if required_mip <= 2:  # Need detailed texture
            return _get_highres_texture(texture_name)
        else:  # Low-res sufficient
            return _get_lowres_texture(texture_name)

    func _get_highres_texture(name: String) -> Texture2D:
        if name in highres_cache:
            return highres_cache[name]

        # Load high-res version
        var texture = load("res://textures/highres/" + name)

        # Check cache size
        var texture_mb = _estimate_texture_size(texture)

        if current_cache_mb + texture_mb > cache_size_mb:
            _evict_least_used()

        highres_cache[name] = texture
        current_cache_mb += texture_mb

        return texture

    func _get_lowres_texture(name: String) -> Texture2D:
        if name in base_textures:
            return base_textures[name]

        var texture = load("res://textures/lowres/" + name)
        base_textures[name] = texture
        return texture

    func _estimate_texture_size(texture: Texture2D) -> float:
        var image = texture.get_image()
        var width = image.get_width()
        var height = image.get_height()
        var format_size = 4  # Assume RGBA8

        # Include mipmaps (+33%)
        var size_bytes = width * height * format_size * 1.33

        return size_bytes / (1024.0 * 1024.0)  # Convert to MB

    func _evict_least_used():
        # Simple: evict first entry
        if not highres_cache.is_empty():
            var first_key = highres_cache.keys()[0]
            var texture = highres_cache[first_key]
            current_cache_mb -= _estimate_texture_size(texture)
            highres_cache.erase(first_key)

# Dynamic mipmap bias based on performance
class AdaptiveMipmapBias:
    @export var target_fps: float = 60.0
    @export var min_bias: float = -1.0
    @export var max_bias: float = 2.0

    var current_bias: float = 0.0

    func update_bias(delta: float):
        var current_fps = 1.0 / delta

        if current_fps < target_fps * 0.9:
            # Running slow, reduce texture quality (increase bias)
            current_bias = min(max_bias, current_bias + 0.1)
        elif current_fps > target_fps * 1.1:
            # Running fast, can afford better quality
            current_bias = max(min_bias, current_bias - 0.05)

    func apply_bias_to_materials(materials: Array):
        for material in materials:
            if material is ShaderMaterial:
                material.set_shader_parameter("mipmap_bias", current_bias)

# Mipmap generation quality
class MipmapGenerator:
    enum FilterType {
        BOX,      # Simple averaging (fast)
        TENT,     # Triangular filter (good)
        LANCZOS,  # High-quality (slow)
        KAISER,   # Best quality (slowest)
    }

    func generate_mipmaps(image: Image, filter: FilterType = FilterType.TENT) -> Image:
        # Godot handles this internally, but conceptually:

        var mipmapped = Image.create(image.get_width(), image.get_height(), true, image.get_format())

        # Generate each mip level
        var current_level = image
        var mip_level = 0

        while current_level.get_width() > 1 or current_level.get_height() > 1:
            # Generate next mip (2x2 downsample)
            var next_width = max(1, current_level.get_width() / 2)
            var next_height = max(1, current_level.get_height() / 2)

            var next_level = _downsample(current_level, next_width, next_height, filter)

            # Would store this mip level in texture
            current_level = next_level
            mip_level += 1

        return mipmapped

    func _downsample(source: Image, new_width: int, new_height: int, filter: FilterType) -> Image:
        var result = Image.create(new_width, new_height, false, source.get_format())

        match filter:
            FilterType.BOX:
                result = _box_filter(source, new_width, new_height)
            FilterType.TENT:
                result = _tent_filter(source, new_width, new_height)
            # ... other filters

        return result

    func _box_filter(source: Image, width: int, height: int) -> Image:
        # Simple 2x2 average
        var result = Image.create(width, height, false, source.get_format())

        for y in range(height):
            for x in range(width):
                var sx = x * 2
                var sy = y * 2

                var c1 = source.get_pixel(sx, sy)
                var c2 = source.get_pixel(sx + 1, sy)
                var c3 = source.get_pixel(sx, sy + 1)
                var c4 = source.get_pixel(sx + 1, sy + 1)

                var avg = (c1 + c2 + c3 + c4) / 4.0
                result.set_pixel(x, y, avg)

        return result

    func _tent_filter(source: Image, width: int, height: int) -> Image:
        # Triangular filter (weighted average of 3x3 area)
        # Better quality than box filter
        return _box_filter(source, width, height)  # Simplified
```

**Key Parameters:**
- Mipmap generation: Always enable for 3D textures
- Mipmap bias: 0.0 default, -0.5 to 0.5 for quality adjustment
- Max mip level: -1 (no limit) or limit to save memory
- Anisotropic level: 1x-16x (8x typical for good quality)
- Filter mode: Linear mipmap (trilinear) for smooth transitions
- Compression: BC7/ASTC with mipmaps for optimal size/quality

**Edge Cases:**
- Texture pop-in: Preload relevant mips before needed
- Mipmap shimmer: Use anisotropic filtering
- UI textures: Disable mipmaps (always at 1:1 size)
- Extremely detailed textures: May need negative bias
- Video memory constraints: Limit max mip levels

**When NOT to Use:**
- UI/HUD elements (fixed screen size)
- Fonts (use distance field rendering instead)
- Pixel art (requires point filtering, mipmaps blur)
- Extremely memory-constrained platforms (though rare)
- Fixed-distance textures (orthographic UI, skybox)

**Examples from Shipped Games:**

1. **id Software Games (DOOM, Rage)**: Mega-texture with aggressive mipmap streaming. 128K×128K textures, load only visible mips. Bandwidth savings: 90%+. Enabled highly detailed environments on limited RAM.

2. **GTA V**: Dynamic mipmap bias based on streaming speed. Distant objects use lower mips until higher res loads. Seamless open world with 4-8GB texture budget.

3. **Horizon Zero Dawn**: Mipmap streaming, 50GB textures on disk, 4GB in VRAM. Prioritizes high-res mips for nearby objects. Anisotropic 16x for terrain quality.

4. **Assassin's Creed Unity**: Aggressive mipmap bias for crowd scenes. Reduces texture bandwidth by 60% in dense Paris streets. Essential for maintaining 30fps with thousands of NPCs.

5. **Minecraft RTX**: Mipmap optimization for voxels, prevents shimmering. PBR textures with mipmaps, anisotropic 16x. Maintains 60fps with path tracing.

**Platform Considerations:**
- **PC High-End**: Anisotropic 16x, full mip chains, bias 0.0
- **PC Low-End**: Anisotropic 4x-8x, may limit max mips, bias +0.5
- **Console (PS5/XSX):** Anisotropic 16x, full mipmaps, SSD enables fast streaming
- **Last-Gen Console**: Anisotropic 8x, careful mip management, bias +0.2
- **Mobile**: Anisotropic 2x-4x, aggressive streaming, bias +0.5 to +1.0
- **Memory**: Mipmaps add 33%, but savings in bandwidth worth it

**Godot-Specific Notes:**
- Godot automatically generates mipmaps for 3D textures
- Import settings: `mipmaps/generate = true`
- Filter mode: `filter_linear_mipmap` in shaders
- Anisotropic: Set in texture import or project settings
- `Texture2D.get_image()` includes mipmaps if generated
- Compression formats (BC/ASTC) preserve mip quality
- Godot 4.x: Improved mipmap streaming for large textures

**Synergies:**
- Essential for **Texture Atlasing** (atlases need mipmaps)
- Pairs with **Anisotropic Filtering** (samples multiple mip levels)
- Combines with **Texture Compression** (compress all mip levels)
- Works with **LOD Systems** (texture LOD matches geometry LOD)
- Enables **Virtual Texturing** (streaming mipmaps on demand)

**Measurement/Profiling:**
- GPU profiler: Texture bandwidth (should decrease 50-70% with mipmaps)
- Visual: Check for Moiré patterns (indicates missing/poor mipmaps)
- Cache hit rate: Should be >90% with mipmaps vs <50% without
- Memory: +33% texture size (worthwhile trade-off)
- Performance: 2-10x faster texture sampling with mipmaps
- Target: 0% Moiré/shimmer, >80% bandwidth reduction distant objects

**Additional Resources:**
- GPU Gems 2: Chapter 26 "Mipmap-Level Measurement"
- Real-Time Rendering 4th Ed: Chapter 6.2.2 "Texture Magnification and Minification"
- id Software: Virtual Texture technology papers
