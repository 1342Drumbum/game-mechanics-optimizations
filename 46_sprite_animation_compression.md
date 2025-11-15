### 46. Sprite Animation Compression

**Category:** Performance - Memory Management & Rendering

**Problem It Solves:** Animation frames consume excessive memory and bandwidth. A character with 20 animations × 12 frames × 128×128 RGBA = 157MB per character. With 5 unique characters = 785MB VRAM. Compression reduces this by 70-90% to 78-157MB total, critical for memory-constrained platforms and faster loading.

**Technical Explanation:**
Sprite animation compression uses multiple techniques: texture atlas packing (combine frames), palette reduction (256 colors vs 16M), delta encoding (store only changed pixels between frames), sprite sheet optimization (remove duplicate frames), and texture compression (DXT/ETC2/ASTC). For static pixels across frames, store once and reuse. For cyclical animations, loop detection eliminates redundant storage.

Advanced approach: separate character into body parts (head, torso, legs), animate independently and composite at runtime. A walk cycle might have 8 leg frames, 4 torso frames, 2 head frames = 14 total vs 8 full-body frames. Combine with bone-based animation for extreme efficiency. Use shader tricks to tint/swap palettes rather than storing color variations.

**Algorithmic Complexity:**
- Uncompressed: O(n × f × w × h × 4) bytes where n=animations, f=frames, w×h=dimensions, 4=RGBA
- Atlas Packing: O((n × f × w × h × 4) / packing_efficiency) - typically 50-70% size
- Delta Encoding: O(changed_pixels_per_frame) - 30-60% reduction for similar frames
- Palette: O(n × f × w × h × 1) + 1KB palette - 75% reduction
- Texture Compression: 4:1 to 8:1 ratio (DXT5/ETC2)

**Implementation Pattern:**
```gdscript
# Animation sprite sheet manager with compression
class_name CompressedAnimationPlayer
extends AnimatedSprite2D

# Instead of separate textures, use sprite sheet
@export var sprite_sheet: Texture2D
@export var frame_data: Dictionary  # Animation name -> frame regions

const FRAME_WIDTH = 64
const FRAME_HEIGHT = 64

func _ready():
    # Build SpriteFrames from compressed sheet
    sprite_frames = SpriteFrames.new()

    for anim_name in frame_data.keys():
        sprite_frames.add_animation(anim_name)

        var frame_coords = frame_data[anim_name]
        for coord in frame_coords:
            var atlas = AtlasTexture.new()
            atlas.atlas = sprite_sheet
            atlas.region = Rect2(
                coord.x * FRAME_WIDTH,
                coord.y * FRAME_HEIGHT,
                FRAME_WIDTH,
                FRAME_HEIGHT
            )
            sprite_frames.add_frame(anim_name, atlas)

# Palette-based sprite animation (256 colors)
class_name PaletteAnimatedSprite
extends Sprite2D

@export var palette_texture: Texture2D  # 8-bit indexed texture
@export var palette_colors: Texture2D   # 256x1 color lookup

const PALETTE_SHADER = """
shader_type canvas_item;

uniform sampler2D indexed_texture : filter_nearest;
uniform sampler2D palette : filter_nearest;

void fragment() {
    // Sample index from texture (red channel)
    float index = texture(indexed_texture, UV).r;

    // Lookup color from palette
    vec2 palette_uv = vec2(index, 0.5);
    COLOR = texture(palette, palette_uv);
}
"""

func _ready():
    var shader_mat = ShaderMaterial.new()
    shader_mat.shader = preload("res://shaders/palette.gdshader")
    shader_mat.set_shader_parameter("indexed_texture", palette_texture)
    shader_mat.set_shader_parameter("palette_colors", palette_colors)
    material = shader_mat

# Runtime palette swapping (character color variations)
func swap_palette(new_palette: Texture2D):
    material.set_shader_parameter("palette_colors", new_palette)
    # Single character asset supports infinite color variations!

# Delta frame compression for similar frames
class_name DeltaFrameAnimation
extends Node2D

class DeltaFrame:
    var base_frame: int = 0  # Reference frame index
    var delta_data: PackedByteArray  # Only changed pixels
    var delta_positions: PackedVector2Array  # Where pixels changed

var base_frames: Array[Image] = []  # Key frames only
var delta_frames: Array[DeltaFrame] = []
var current_frame: int = 0

func compress_animation(frames: Array[Image]):
    # First frame is always a base frame
    base_frames.append(frames[0])

    for i in range(1, frames.size()):
        var delta = DeltaFrame.new()
        delta.base_frame = find_closest_base_frame(frames[i])

        # Calculate differences from base
        var base = base_frames[delta.base_frame]
        var changed_pixels = PackedByteArray()
        var positions = PackedVector2Array()

        for y in range(frames[i].get_height()):
            for x in range(frames[i].get_width()):
                var current_pixel = frames[i].get_pixel(x, y)
                var base_pixel = base.get_pixel(x, y)

                if not current_pixel.is_equal_approx(base_pixel):
                    positions.append(Vector2(x, y))
                    changed_pixels.append(int(current_pixel.r * 255))
                    changed_pixels.append(int(current_pixel.g * 255))
                    changed_pixels.append(int(current_pixel.b * 255))
                    changed_pixels.append(int(current_pixel.a * 255))

        delta.delta_data = changed_pixels
        delta.delta_positions = positions
        delta_frames.append(delta)

        # Add as new base frame if too many changes (>30% pixels)
        if positions.size() > (frames[i].get_width() * frames[i].get_height() * 0.3):
            base_frames.append(frames[i])

func decompress_frame(frame_index: int) -> Image:
    if frame_index == 0:
        return base_frames[0].duplicate()

    var delta = delta_frames[frame_index - 1]
    var result = base_frames[delta.base_frame].duplicate()

    # Apply delta changes
    for i in range(delta.delta_positions.size()):
        var pos = delta.delta_positions[i]
        var offset = i * 4
        var color = Color(
            delta.delta_data[offset] / 255.0,
            delta.delta_data[offset + 1] / 255.0,
            delta.delta_data[offset + 2] / 255.0,
            delta.delta_data[offset + 3] / 255.0
        )
        result.set_pixel(int(pos.x), int(pos.y), color)

    return result

# Modular character animation (separate body parts)
class_name ModularCharacterAnimator
extends Node2D

@export var head_sprites: SpriteFrames
@export var body_sprites: SpriteFrames
@export var legs_sprites: SpriteFrames

@onready var head: AnimatedSprite2D = $Head
@onready var body: AnimatedSprite2D = $Body
@onready var legs: AnimatedSprite2D = $Legs

# Instead of 10 animations × 8 frames = 80 full frames,
# Use 10 × (2 head + 3 body + 8 legs) = 130 part frames
# Memory saving: ~38% reduction

func play_animation(anim_name: String):
    match anim_name:
        "walk":
            head.play("walk_head")  # 2 frames
            body.play("walk_body")  # 3 frames
            legs.play("walk_legs")  # 8 frames
        "idle":
            head.play("idle_head")  # 2 frames
            body.play("idle_body")  # 4 frames (breathing)
            legs.play("idle_legs")  # 1 frame (static)
        "attack":
            head.play("attack_head")   # 3 frames
            body.play("attack_body")   # 6 frames (swinging)
            legs.play("idle_legs")     # 1 frame (reuse!)

# Each part animates independently, massive frame reuse

# Texture compression helper
class_name TextureCompressor
extends Node

enum CompressionFormat {
    UNCOMPRESSED,  # RGBA8
    DXT5,          # Desktop, 4:1 ratio
    ETC2,          # Mobile, 4:1 ratio
    ASTC_4x4,      # Modern mobile, 8:1 ratio
    BASIS_UNIVERSAL # Universal, 6:1 average
}

static func get_memory_usage(texture: Texture2D, format: CompressionFormat) -> int:
    var pixels = texture.get_width() * texture.get_height()

    match format:
        CompressionFormat.UNCOMPRESSED:
            return pixels * 4  # 4 bytes per pixel
        CompressionFormat.DXT5, CompressionFormat.ETC2:
            return pixels / 4  # 4:1 compression
        CompressionFormat.ASTC_4x4:
            return pixels / 8  # 8:1 compression
        CompressionFormat.BASIS_UNIVERSAL:
            return pixels / 6  # 6:1 average
    return pixels * 4

# Example: 1024x1024 sprite
# Uncompressed: 4,194,304 bytes (4MB)
# DXT5/ETC2: 1,048,576 bytes (1MB)
# ASTC 4x4: 524,288 bytes (512KB)
```

**Key Parameters:**
- **Texture Compression:** DXT5 (desktop), ETC2 (mobile), ASTC 4x4 (modern)
- **Atlas Padding:** 2-4 pixels between frames (prevent bleeding)
- **Color Depth:** 32-bit (RGBA8), 16-bit (RGB565), 8-bit (palette)
- **Frame Optimization:** Remove duplicate frames, loop detection
- **Delta Threshold:** 30% changed pixels = new base frame
- **Part Animation Split:** 3-5 parts typical (head, body, arms, legs, weapon)

**Edge Cases:**
- **Compression Artifacts:** DXT5 shows banding on gradients, use ASTC for higher quality
- **Palette Limitations:** 256 color limit problematic for complex sprites
- **Atlas Bleeding:** Adjacent frames blend at edges, requires padding
- **Animation Speed:** Compressed textures slower to update (decompression cost)
- **Memory/Quality Trade-off:** Aggressive compression degrades visual quality
- **Platform Support:** Some compression formats unavailable on older hardware

**When NOT to Use:**
- Desktop-only games with abundant VRAM (8GB+) - quality over compression
- Few animations (1-2 per character) - overhead not worth it
- High-fidelity art style requiring uncompressed quality
- Development builds - harder to iterate with compressed assets
- Real-time animation editing - compression pipeline adds complexity

**Examples from Shipped Games:**

1. **Dead Cells:** Modular character system with 50+ body parts. Each part has 3-8 animation frames. Weapon sprites separate (100+ weapons share ~15 animation sets). Uses texture atlases with ETC2 compression. Total character memory: ~15MB vs 200MB+ if fully composed. Essential for Switch port.

2. **Celeste:** Character animations use delta encoding - most frames only differ by 20-40 pixels from previous frame. Madeline has 100+ animation frames compressed to ~2MB. Palette swapping used for costume variants (no additional memory). Uses BASIS Universal compression for cross-platform.

3. **Terraria:** 2500+ item sprites in single compressed atlas (16MB vs 180MB individual). Uses palette system for ore/gem color variants. Texture compression (DXT1 for opaque, DXT5 for transparency). NPCs use modular animation (head/body/legs separate) saving ~60% memory.

4. **Stardew Valley:** Character portraits use palette swapping - single 128×128 base with 15 color palettes = 16 variants from 1 texture. Animation frames packed in tight atlases with 1-pixel padding. Total character assets ~8MB for all 50+ NPCs.

5. **Enter the Gungeon:** 200+ gun animations share common firing/reloading frames. Delta compression for similar gun types. Uses modular system: gun base + barrel + magazine = thousands of variations from <100 part sprites. Texture atlases with DXT5, total weapons: ~30MB vs 400MB+ uncompressed.

**Platform Considerations:**
- **Mobile (iOS/Android):** Essential - VRAM limited to 512MB-2GB. Use ASTC or ETC2 compression (4:1 minimum). Palette mode for pixel art. Target <50MB for all character animations.
- **Desktop (Windows/Mac/Linux):** Helpful but not critical. Use DXT5/BC7 compression. Can afford higher quality. VRAM budget 100-500MB for sprites okay.
- **Console (Switch/PS/Xbox):** Switch critical (4GB shared RAM) - use ASTC, target <100MB sprites. PS5/Xbox less constrained but compression still beneficial for load times.
- **WebGL:** Absolutely necessary - download size matters. BASIS Universal recommended (works everywhere). Target <20MB total sprite data.
- **Storage:** Compressed textures also reduce disk footprint by 70-90%, faster loading.

**Godot-Specific Notes:**
- Godot supports DXT/ETC2/ASTC via import settings: Texture → Compress → Mode
- Use `Image.compress()` for runtime compression (rarely needed)
- AtlasTexture for sprite sheet regions - zero memory overhead
- SpriteFrames automatically shares textures between frames
- Godot 4.x supports BASIS Universal (.basis files) for universal compression
- Import tab: set VRAM Compression based on platform
- Use Lossy quality 0.7-0.9 for aggressive compression (smaller file, slight quality loss)

**Synergies:**
- **Sprite Batching:** Atlased animations batch together perfectly
- **Object Pooling:** Pooled objects share compressed sprite data
- **Texture Atlasing:** Required for maximum compression efficiency
- **Canvas Layer Management:** Compressed textures load faster between layer switches
- **Modular Animation:** Combines with part-based systems for extreme savings

**Measurement/Profiling:**
- **VRAM Usage:** `Performance.RENDER_TEXTURE_MEM_USED` - before/after compression
- **Loading Time:** Track `ResourceLoader.load()` time for sprite sheets
- **Compression Ratio:** Compare file sizes (compressed vs uncompressed)
- **Visual Quality:** A/B test compressed vs uncompressed in-game
- **Profiling Pattern:**
```gdscript
var before_mem = Performance.get_monitor(Performance.RENDER_TEXTURE_MEM_USED)
var texture = load("res://characters/hero_compressed.png")
var after_mem = Performance.get_monitor(Performance.RENDER_TEXTURE_MEM_USED)
var used = (after_mem - before_mem) / 1024.0 / 1024.0
print("Texture memory: ", used, " MB")

# Check actual texture format
var image = texture.get_image()
print("Format: ", image.get_format())  # Should be compressed format
```

**Advanced Compression Techniques:**
```gdscript
# Technique 1: Automatic duplicate frame detection
func remove_duplicate_frames(frames: Array[Image]) -> Array[Image]:
    var unique_frames = []
    var frame_hashes = {}

    for frame in frames:
        var hash_val = frame.get_data().hash()
        if not frame_hashes.has(hash_val):
            unique_frames.append(frame)
            frame_hashes[hash_val] = unique_frames.size() - 1

    print("Removed ", frames.size() - unique_frames.size(), " duplicate frames")
    return unique_frames

# Technique 2: Adaptive compression quality
func compress_with_quality_target(texture: Texture2D, max_error: float) -> Texture2D:
    var qualities = [1.0, 0.9, 0.8, 0.7, 0.5]
    for quality in qualities:
        var compressed = compress_texture(texture, quality)
        var error = calculate_mse(texture, compressed)
        if error <= max_error:
            return compressed
    return texture

# Technique 3: Animation frame interpolation (reduce stored frames)
func interpolate_frames(frame_a: Image, frame_b: Image, t: float) -> Image:
    var result = Image.create(frame_a.get_width(), frame_a.get_height(), false, Image.FORMAT_RGBA8)

    for y in range(frame_a.get_height()):
        for x in range(frame_a.get_width()):
            var color_a = frame_a.get_pixel(x, y)
            var color_b = frame_b.get_pixel(x, y)
            var interpolated = color_a.lerp(color_b, t)
            result.set_pixel(x, y, interpolated)

    return result

# Store keyframes only, interpolate between at runtime
# 12 frame animation -> 3 keyframes + interpolation = 75% memory saving
```
