### 32. Anisotropic Filtering

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Texture blur on surfaces viewed at grazing angles. Bilinear/trilinear filtering samples textures isotropically (circular region), causing blur when surface stretches into distance (16:1 aspect ratio). Floors/walls at shallow angles become blurry mush. Anisotropic filtering samples elongated regions (elliptical footprint), maintaining clarity. Cost: 2-4x texture samples per pixel on affected surfaces, but GPU optimizations make practical cost 10-20% performance hit for massive visual improvement.

**Technical Explanation:**
Standard filtering samples square/circular texture region. Anisotropic filtering detects screen-space texture stretching, samples elongated footprint. Samples 2-16 texels along major axis (aniso level). Modern GPUs optimize via texture cache and parallel sampling hardware. Cost scales with aniso level: 1x (disabled) → 2x → 4x → 8x → 16x. Diminishing returns beyond 8x. GPU efficiently handles 8x-16x with minimal overhead due to cache coherency. Affects primarily surfaces at low angles (floors, walls); perpendicular surfaces unaffected. Critical for environmental quality in 3D games.

**Algorithmic Complexity:**
- Bilinear: O(4) texture samples per pixel
- Trilinear: O(8) samples (2 mip levels × 4)
- Anisotropic 4x: O(16) samples worst case (4 along major axis × 4 trilinear)
- Anisotropic 16x: O(64) samples theoretical, ~O(24) practical (caching)
- Real cost: 0-30% depending on camera angles, surface distribution

**Implementation:**
```gdscript
# Godot Anisotropic Filtering Configuration
class_name AnisotropicFilteringManager
extends Node

# Global anisotropic filtering settings
func set_global_anisotropic_level(level: int):
    # level: 0=off, 1=2x, 2=4x, 3=8x, 4=16x
    var aniso = clamp(level, 0, 4)

    # Set in project settings
    ProjectSettings.set_setting("rendering/textures/default_filters/anisotropic_filtering_level", aniso)

    print("Anisotropic filtering set to: ", _level_to_string(aniso))

func _level_to_string(level: int) -> String:
    match level:
        0: return "Disabled"
        1: return "2x"
        2: return "4x"
        3: return "8x"
        4: return "16x"
        _: return "Unknown"

# Per-texture anisotropic configuration
func configure_texture_anisotropic(texture_path: String, level: int):
    # Set in import settings
    var import_config = {
        "filtering/anisotropic": level
    }
    # Applied via .import file

# Adaptive anisotropic filtering based on performance
class AdaptiveAnisotropic:
    @export var target_fps: float = 60.0
    var current_level: int = 3  # Start at 8x

    func update_quality(delta: float):
        var current_fps = 1.0 / delta

        if current_fps < target_fps * 0.9:
            current_level = max(1, current_level - 1)  # Reduce quality
        elif current_fps > target_fps * 1.05:
            current_level = min(4, current_level + 1)  # Increase quality

        ProjectSettings.set_setting("rendering/textures/default_filters/anisotropic_filtering_level", current_level)
```

**Key Parameters:**
- Aniso level: 0 (off), 2x, 4x, 8x, 16x
- Typical setting: 8x (good quality/performance balance)
- High-end: 16x (minimal extra cost on modern GPUs)
- Mobile: 2x-4x (significant cost on older GPUs)
- Per-texture override: Critical textures 16x, background 4x

**Edge Cases:**
- Perpendicular surfaces: No cost (aniso doesn't trigger)
- UI textures: Disable (no benefit, wastes performance)
- Extremely high resolution: May hit bandwidth limits
- Tile-based renderers (mobile): Higher cost than desktop

**When NOT to Use:**
- UI/HUD elements
- Particle textures (too small/temporary)
- Very low-end hardware (pre-2010)
- Stylized games with intentional low-detail aesthetic

**Examples from Shipped Games:**
1. **Crysis 3**: Anisotropic 16x standard, 2-3ms cost. Essential for jungle floor detail.
2. **The Witcher 3**: 16x on PC, 4x-8x on console. Noticeable quality difference in Novigrad streets.
3. **Forza Horizon 5**: 16x, <1ms cost on modern hardware. Critical for road surface clarity.
4. **Red Dead Redemption 2**: 16x on PC ultra. Maintains ground texture quality at distance.
5. **Spider-Man**: 8x on PS4, 16x on PS5. Improved building texture clarity.

**Platform Considerations:**
- **PC High-End**: 16x standard, <1% performance cost
- **PC Low-End**: 4x-8x, 5-10% cost
- **Console (PS5/XSX)**: 16x, minimal cost
- **Last-Gen Console**: 4x-8x, 10-15% cost
- **Mobile High**: 4x, 10-20% cost
- **Mobile Low**: 2x or disabled, 20-30% cost if enabled

**Godot-Specific Notes:**
- Set globally: `rendering/textures/default_filters/anisotropic_filtering_level`
- Per-texture: Import settings
- Shader: Use `filter_linear_mipmap_anisotropic`
- Godot 4.x: Better anisotropic performance than 3.x
- Automatic for StandardMaterial3D when enabled globally

**Measurement:**
- Visual: Compare floor textures at distance (before/after)
- GPU profiler: Texture sampler time (10-20% increase typical)
- Framerate: 5-15% cost for 16x vs off
- Target: <2ms additional cost for massive visual improvement

