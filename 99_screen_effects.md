### 99. Screen Effects (Chromatic Aberration, Vignette)

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Flat, lifeless visuals that lack intensity and drama. Players can't immediately identify critical moments (damage, low health, power-ups). Uniform screen appearance fails to guide player attention. Without screen effects, intense moments feel the same as calm moments, reducing emotional impact and gameplay clarity.

**Technical Explanation:**
Post-processing shader effects applied to entire screen. Chromatic aberration separates RGB channels creating "lens distortion" (shift red/blue channels by 2-10 pixels). Vignette darkens screen edges using radial gradient. Effects controlled by intensity parameters (0-1) that spike on events then decay. Implemented as full-screen quad with fragment shader. GPU samples screen texture 3x for chromatic aberration (R, G, B at different UV offsets), blends vignette in screen-space using distance from center.

**Algorithmic Complexity:**
- O(1) per pixel: 3-5 texture samples + math
- O(w×h) total: Every screen pixel processed
- GPU: 0.1-0.5ms at 1080p, 0.3-1.5ms at 4K
- CPU: Negligible, shader parameters only
- Memory: 1 full-screen buffer (~8MB at 1080p RGBA)

**Implementation Pattern:**
```gdscript
# Godot implementation - Screen Effects Manager
class_name ScreenEffectsManager
extends Node

@onready var chromatic_aberration: ColorRect = $ChromaticAberration
@onready var vignette: ColorRect = $Vignette

# Chromatic aberration intensity
var ca_intensity: float = 0.0
var ca_target: float = 0.0
var ca_decay_rate: float = 2.0

# Vignette intensity
var vignette_intensity: float = 0.2  # Base vignette
var vignette_target: float = 0.2
var vignette_decay_rate: float = 1.5

func _ready():
	_setup_shaders()

func _setup_shaders():
	# Chromatic Aberration Shader
	var ca_shader = preload("res://shaders/chromatic_aberration.gdshader")
	chromatic_aberration.material = ShaderMaterial.new()
	chromatic_aberration.material.shader = ca_shader

	# Vignette Shader
	var vignette_shader = preload("res://shaders/vignette.gdshader")
	vignette.material = ShaderMaterial.new()
	vignette.material.shader = vignette_shader

func _process(delta):
	# Decay chromatic aberration
	ca_intensity = lerp(ca_intensity, ca_target, ca_decay_rate * delta)
	chromatic_aberration.material.set_shader_parameter("intensity", ca_intensity)

	# Decay vignette
	vignette_intensity = lerp(vignette_intensity, vignette_target, vignette_decay_rate * delta)
	vignette.material.set_shader_parameter("intensity", vignette_intensity)

# Trigger effects on events
func flash_chromatic_aberration(intensity: float = 0.5, duration: float = 0.2):
	ca_intensity = intensity
	ca_target = 0.0
	# Optional: Reset decay after duration
	await get_tree().create_timer(duration).timeout
	ca_target = 0.0

func set_vignette_intensity(intensity: float):
	vignette_target = intensity

# Health-based vignette (low health = darker edges)
func update_health_vignette(health_percent: float):
	# 100% health = 0.2 vignette, 0% health = 0.8 vignette
	vignette_target = lerp(0.8, 0.2, health_percent)

# Damage flash (combined effect)
func on_player_damaged(damage: float):
	var intensity = clamp(damage / 50.0, 0.0, 1.0)

	# Flash chromatic aberration
	flash_chromatic_aberration(intensity * 0.8)

	# Pulse vignette
	vignette_intensity = vignette_target + intensity * 0.3
```

**Chromatic Aberration Shader (chromatic_aberration.gdshader):**
```glsl
shader_type canvas_item;

uniform float intensity : hint_range(0.0, 1.0) = 0.0;
uniform vec2 direction = vec2(1.0, 0.0);  // Aberration direction
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;

void fragment() {
	vec2 uv = SCREEN_UV;

	// Calculate offset based on distance from center
	vec2 center_offset = uv - vec2(0.5, 0.5);
	float dist = length(center_offset);

	// Offset amount (increases toward edges)
	vec2 offset = normalize(center_offset) * intensity * 0.01 * (1.0 + dist);

	// Sample RGB channels at different offsets
	float r = texture(screen_texture, uv + offset).r;
	float g = texture(screen_texture, uv).g;
	float b = texture(screen_texture, uv - offset).b;

	COLOR = vec4(r, g, b, 1.0);
}
```

**Vignette Shader (vignette.gdshader):**
```glsl
shader_type canvas_item;

uniform float intensity : hint_range(0.0, 1.0) = 0.3;
uniform float falloff : hint_range(0.1, 2.0) = 0.5;
uniform vec4 vignette_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;

void fragment() {
	vec2 uv = SCREEN_UV;

	// Distance from center
	vec2 center = uv - vec2(0.5, 0.5);
	float dist = length(center);

	// Vignette gradient
	float vignette = smoothstep(falloff, falloff * 2.0, dist);
	vignette = pow(vignette, 2.0);  // Exponential falloff

	// Blend with vignette color
	vec4 screen_color = texture(screen_texture, uv);
	COLOR = mix(screen_color, vignette_color, vignette * intensity);
}
```

**Key Parameters:**
- **Chromatic aberration intensity:** 0.0-0.5 (subtle), 0.5-1.0 (extreme)
- **CA offset distance:** 0.005-0.02 UV units (2-10 pixels at 1080p)
- **CA decay rate:** 1.5-3.0 (higher = faster recovery)
- **Vignette base intensity:** 0.1-0.3 (always visible)
- **Vignette max intensity:** 0.6-0.9 (low health/damage)
- **Vignette falloff:** 0.3-0.7 (lower = tighter circle)
- **Flash duration:** 0.1-0.3 seconds

**Edge Cases:**
- **Multiple simultaneous effects:** Use max intensity, don't stack
- **Colorblind modes:** Reduce chromatic aberration, increase vignette
- **High contrast modes:** Disable chromatic aberration entirely
- **Motion sickness:** Provide option to disable per-effect
- **Cutscenes:** Disable reactive effects, use authored values
- **UI rendering:** Effects should be below UI layer
- **Screenshot mode:** Disable effects for clean captures

**When NOT to Use:**
- **VR games:** Motion sickness risk - avoid reactive effects
- **Competitive multiplayer:** Visual clarity is paramount
- **Pixel art games:** Chromatic aberration breaks pixel aesthetic
- **Puzzle games:** Distraction from problem-solving
- **Accessibility issues:** Always provide disable options
- **Reading-heavy games:** Vignette reduces text readability

**Examples from Shipped Games:**

1. **Hyper Light Drifter:** Heavy chromatic aberration on dash (0.6 intensity), vignette intensifies at low health (0.3 → 0.8). Combined with pink/cyan color grading creates signature aesthetic. Effects are aggressive but contribute to style.

2. **Doom (2016):** Vignette pulses on damage (0.2 → 0.6), desaturates at low health. Glory kill triggers chromatic aberration flash (0.4 intensity, 0.15s). Effects communicate danger without UI health bar.

3. **Celeste:** Minimal chromatic aberration on dash (0.2 intensity), vignette darkens during anxiety sequences (narrative tool). Conservative use maintains visual clarity for precise platforming.

4. **Dead Cells:** Chromatic aberration on parry (0.5 intensity), vignette increases with curse stacks (gameplay mechanic). Red vignette indicates curse, separate from health-based vignette. Multi-layered vignette system.

5. **Hotline Miami:** Heavy chromatic aberration always active (0.3-0.5 base), pulses on kills. Vignette flashes on damage. Core to the game's psychedelic, disorienting aesthetic. Intentionally extreme.

**Platform Considerations:**
- **Desktop GPU:** Full effects at 1080p-4K, <1ms overhead
- **Mobile GPU:** Reduce shader complexity, use lookup textures
- **Switch:** Optimize shaders, consider dynamic resolution scaling
- **WebGL:** Keep shaders simple, avoid dependent texture reads
- **4K displays:** Increase offset distances proportionally
- **High refresh rate:** No special considerations, shader cost constant

**Godot-Specific Notes:**
- Use `ColorRect` with `ShaderMaterial` for full-screen effects
- Access backbuffer via `hint_screen_texture` in Godot 4.x
- Stack effects using multiple `ColorRect` nodes in `CanvasLayer`
- Set `mouse_filter = MOUSE_FILTER_IGNORE` to prevent input blocking
- Use `BackBufferCopy` node in Godot 3.x for screen texture access
- Post-processing order: Chromatic Aberration → Vignette → Bloom
- Consider `RenderingServer` for more control in Godot 4.x

**Synergies:**
- **Hit Pause (Technique 95):** Flash chromatic aberration during pause
- **Screen Shake (Technique 97):** Combined shake + aberration = impact
- **Particle Effects (Technique 98):** Particles spawn when effect triggers
- **Damage Scaling (Technique 102):** Higher damage = stronger effect
- **Hit Confirmation (Technique 106):** Part of multi-sensory feedback
- **Health UI:** Vignette as non-intrusive health indicator

**Measurement/Profiling:**
- **GPU profiler:** Shader cost should be <1ms at target resolution
- **Frame timing:** Monitor for performance spikes during effect
- **Visual testing:** Check for color banding in vignette gradients
- **A/B testing:** Compare player performance with/without effects
- **Accessibility feedback:** Track complaints about motion sickness
- **Pixel count:** Effects scale with resolution (4K = 4x cost of 1080p)
- **Shader complexity:** Use Godot's shader profiler in debug builds

**Advanced Techniques:**
```gdscript
# Directional chromatic aberration (toward damage source)
func flash_directional_aberration(damage_position: Vector2, intensity: float):
	var player_pos = get_viewport().get_camera_2d().get_screen_center_position()
	var direction = (damage_position - player_pos).normalized()

	chromatic_aberration.material.set_shader_parameter("direction", direction)
	chromatic_aberration.material.set_shader_parameter("intensity", intensity)

# Colored vignette (poison = green, fire = red)
func set_vignette_color(color: Color, intensity: float):
	vignette.material.set_shader_parameter("vignette_color", color)
	vignette.material.set_shader_parameter("intensity", intensity)

# Pulse effect (rhythmic pulsing)
func pulse_vignette(frequency: float, amplitude: float):
	while is_pulsing:
		var pulse = sin(Time.get_ticks_msec() * 0.001 * frequency) * amplitude
		vignette_intensity = vignette_target + pulse
		await get_tree().process_frame

# Speed-based chromatic aberration
func update_speed_aberration(speed: float, max_speed: float):
	var speed_percent = clamp(speed / max_speed, 0.0, 1.0)
	ca_target = speed_percent * 0.3  # Max 0.3 at full speed

# Combined screen distortion shader
shader_type canvas_item;

uniform float ca_intensity : hint_range(0.0, 1.0) = 0.0;
uniform float vignette_intensity : hint_range(0.0, 1.0) = 0.3;
uniform float distortion_amount : hint_range(0.0, 0.1) = 0.0;
uniform sampler2D screen_texture : hint_screen_texture;

void fragment() {
	vec2 uv = SCREEN_UV;
	vec2 center_offset = uv - vec2(0.5);
	float dist = length(center_offset);

	// Barrel distortion
	vec2 distorted_uv = uv + center_offset * distortion_amount * dist;

	// Chromatic aberration
	vec2 ca_offset = center_offset * ca_intensity * 0.01;
	float r = texture(screen_texture, distorted_uv + ca_offset).r;
	float g = texture(screen_texture, distorted_uv).g;
	float b = texture(screen_texture, distorted_uv - ca_offset).b;

	vec3 color = vec3(r, g, b);

	// Vignette
	float vignette = smoothstep(0.5, 1.5, dist);
	vignette = pow(vignette, 2.0);
	color = mix(color, vec3(0.0), vignette * vignette_intensity);

	COLOR = vec4(color, 1.0);
}
```

**Accessibility Settings:**
```gdscript
# User preferences
@export var chromatic_aberration_enabled: bool = true
@export var vignette_enabled: bool = true
@export var effect_intensity_multiplier: float = 1.0  # 0.0-1.0

func flash_chromatic_aberration(intensity: float):
	if not chromatic_aberration_enabled:
		return

	var adjusted_intensity = intensity * effect_intensity_multiplier
	ca_intensity = adjusted_intensity
```
