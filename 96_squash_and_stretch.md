### 96. Squash & Stretch

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Stiff, lifeless animation that feels mechanical and unengaging. Objects appear rigid and lack weight, energy, and personality. Without squash and stretch, movements feel floaty and disconnected from physics. Players subconsciously perceive animated objects as "fake" or "game-y" rather than believable.

**Technical Explanation:**
One of Disney's 12 Principles of Animation. Deform object shape during motion to convey mass, velocity, and flexibility. Stretch along direction of motion (fast movement), squash perpendicular to impact (collision/landing). Key rule: preserve volume - if width scales by 1.5x, height scales by ~0.67x (1/1.5). Implementation uses scale transformation on sprite/mesh: `scale = Vector2(width_scale, height_scale)`. The technique creates visual momentum and anticipation, making motion readable and satisfying.

**Algorithmic Complexity:**
- O(1) per frame: Single scale transformation
- O(1) memory: 2 floats for scale vector
- GPU cost: Negligible - standard vertex transformation
- ~0.001ms CPU per object

**Implementation Pattern:**
```gdscript
# Godot implementation - SquashAndStretch component
class_name SquashAndStretch
extends Node2D

@export var sprite: Node2D  # The node to deform
@export var max_stretch: float = 1.3  # Maximum stretch multiplier
@export var max_squash: float = 0.7   # Maximum squash multiplier
@export var spring_stiffness: float = 20.0  # Return to normal speed
@export var velocity_scale: float = 0.02  # Velocity to stretch conversion
@export var preserve_volume: bool = true  # Maintain area

var base_scale: Vector2 = Vector2.ONE
var current_scale: Vector2 = Vector2.ONE
var target_scale: Vector2 = Vector2.ONE
var velocity: Vector2 = Vector2.ZERO

func _ready():
	if sprite:
		base_scale = sprite.scale

func _process(delta):
	# Interpolate toward target scale (spring back)
	current_scale = current_scale.lerp(target_scale, spring_stiffness * delta)

	# Apply scale
	if sprite:
		sprite.scale = base_scale * current_scale

# Call this when velocity changes
func update_from_velocity(vel: Vector2):
	velocity = vel
	var speed = vel.length()

	if speed < 1.0:
		target_scale = Vector2.ONE
		return

	# Stretch along direction of movement
	var direction = vel.normalized()
	var stretch_amount = min(speed * velocity_scale, max_stretch - 1.0)

	# Calculate stretch along movement direction
	var angle = direction.angle()
	var stretch_x = 1.0 + stretch_amount * abs(cos(angle))
	var stretch_y = 1.0 + stretch_amount * abs(sin(angle))

	if preserve_volume:
		# Preserve area: if x scales up, y scales down
		var volume = stretch_x * stretch_y
		stretch_y = 1.0 / stretch_x if stretch_x > 1.0 else 1.0
		stretch_x = 1.0 / stretch_y if stretch_y > 1.0 else stretch_x

	target_scale = Vector2(stretch_x, stretch_y)

# Call on landing/impact
func squash(intensity: float = 1.0):
	var squash_y = max_squash + (1.0 - max_squash) * (1.0 - intensity)
	var squash_x = 1.0 / squash_y if preserve_volume else 1.0

	current_scale = Vector2(squash_x, squash_y)
	target_scale = Vector2.ONE

	# Optional: Trigger with hit pause for extra impact
	await get_tree().create_timer(0.05).timeout
	target_scale = Vector2.ONE * 1.1  # Slight overshoot

# Jump anticipation
func anticipate_jump():
	squash(0.8)  # Pre-squash before launch
	await get_tree().create_timer(0.1).timeout
	target_scale = Vector2(0.8, 1.4)  # Stretch upward

# Usage example in character controller
extends CharacterBody2D

@onready var squash: SquashAndStretch = $SquashAndStretch

func _physics_process(delta):
	# Update squash based on movement
	squash.update_from_velocity(velocity)

	# Squash on landing
	if is_on_floor() and velocity.y > 100:
		var intensity = clamp(velocity.y / 1000.0, 0.0, 1.0)
		squash.squash(intensity)

	move_and_slide()
```

**Key Parameters:**
- **Max stretch:** 1.2-1.5x (subtle), 1.5-2.0x (cartoony)
- **Max squash:** 0.5-0.7x (realistic), 0.3-0.5x (exaggerated)
- **Spring back speed:** 10-30 (lower = bouncier, higher = snappier)
- **Velocity threshold:** Start stretching at 50-200 units/sec
- **Impact squash duration:** 0.05-0.15 seconds (3-9 frames at 60fps)
- **Overshoot amount:** 1.05-1.15x (subtle bounce back)

**Edge Cases:**
- **Rotating sprites:** Apply squash in local space, not world space
- **Scaling hierarchies:** Store original scale, multiply don't add
- **Multiple simultaneous effects:** Blend or priority system (landing > running)
- **Network sync:** Squash is client-side only, don't sync over network
- **Pixel art:** Quantize scale to whole pixels to avoid artifacts
- **Circular objects:** Equal squash/stretch on both axes can look wrong
- **Zero velocity:** Avoid division by zero when normalizing direction

**When NOT to Use:**
- **Rigid mechanical objects:** Robots, stone blocks, metal projectiles
- **Realistic military shooters:** Breaks immersion
- **Precise hitbox games:** Fighting games where hitbox must match visual exactly
- **UI elements:** Usually too subtle to notice, wastes CPU
- **Far distance objects:** Invisible at distance, skip for LOD
- **High object counts:** 1000+ stretched bullets causes visual noise

**Examples from Shipped Games:**

1. **Celeste:** Madeline squashes 20% on landing (heavier landings squash more), stretches 30% during dash. Hair particles have independent squash/stretch creating drag effect. Jump has anticipation squash. One of the best implementations of this principle.

2. **Dead Cells:** Character squashes heavily on landing (0.6x height), weapons stretch during swing (1.4x), enemies squash on hit. Each weapon type has unique stretch parameters - daggers 1.2x, greatswords 1.6x. Creates distinct weapon feel.

3. **Shovel Knight:** Shovel Knight squashes on ground pound, bounces on enemy heads with exaggerated stretch. Enemies squash when hit. Classic NES aesthetic but uses squash/stretch for modern juice.

4. **Cuphead:** Extreme squash/stretch (0.4x - 2.0x range) matching 1930s rubber hose animation. Every frame of animation incorporates the principle. Projectiles stretch heavily, characters are in constant deformation.

5. **Hollow Knight:** Subtle squash on nail swing (1.1x), squash on landing (0.85x), enemies squash/stretch on damage. More reserved than other examples but still present and important to the feel.

**Platform Considerations:**
- **Pixel art games:** Use integer scale increments or shader-based squash
- **3D games:** Apply to mesh vertices or use bone scaling
- **Mobile:** Very cheap effect, use liberally
- **High DPI:** Scale values need adjustment for different resolutions
- **Performance:** <0.01ms per object, negligible impact
- **Retro aesthetic:** Limit to 0.8x-1.2x range for authenticity

**Godot-Specific Notes:**
- Apply to `Node2D.scale` for sprites
- Use `Skeleton2D` bone scaling for advanced character rigs
- `AnimationPlayer` can keyframe scale for hand-animated squash
- Shader alternative: `uniform vec2 squash_scale` in vertex shader
- `Tween` for smooth interpolation: `tween.tween_property(sprite, "scale", target, duration)`
- Godot 4.x: Same as 3.x, consider using `PhysicsBody2D.constant_linear_velocity` for auto-stretch
- Particles: Set `scale_amount_curve` for projectile stretch

**Synergies:**
- **Hit Pause (Technique 95):** Hold squashed frame during pause for emphasis
- **Anticipation Frames (Technique 100):** Squash before action, stretch during
- **Follow-Through (Technique 101):** Overshoot stretch, bounce back to normal
- **Particle Effects (Technique 98):** Particles inherit parent's squash
- **Camera Shake (Technique 97):** Combined squash + shake = maximum impact
- **Animation Blending (Technique 103):** Blend between squashed states smoothly

**Measurement/Profiling:**
- **Visual comparison:** Side-by-side video with/without effect
- **Playtest surveys:** "Does movement feel weighty?" rating 1-10
- **Performance:** Godot profiler should show <0.01ms in `_process`
- **Exaggeration testing:** Gradually increase max values until it "feels wrong"
- **Frame-by-frame analysis:** Record at 60fps, verify squash timing
- **Accessibility:** Some players with motion sensitivity may prefer less extreme values

**Advanced Techniques:**
```gdscript
# Rotation-aware squash (stretch along local axes)
func update_from_velocity_rotated(vel: Vector2, rotation_rad: float):
	var local_velocity = vel.rotated(-rotation_rad)
	var speed = local_velocity.length()

	if speed < 1.0:
		target_scale = Vector2.ONE
		return

	var stretch = 1.0 + min(speed * velocity_scale, max_stretch - 1.0)
	var squash = 1.0 / stretch if preserve_volume else 1.0

	target_scale = Vector2(squash, stretch)

# Squash along impact normal (for collisions)
func squash_from_collision(collision_normal: Vector2, impact_speed: float):
	var intensity = clamp(impact_speed / 500.0, 0.0, 1.0)

	# Squash perpendicular to collision surface
	var angle = collision_normal.angle()
	var squash_amount = max_squash + (1.0 - max_squash) * (1.0 - intensity)

	# More complex: rotate squash to match surface
	var squash_x = 1.0 if abs(cos(angle)) < 0.5 else squash_amount
	var squash_y = 1.0 if abs(sin(angle)) < 0.5 else squash_amount

	current_scale = Vector2(squash_x, squash_y)

# Ease curves for more natural motion
const EASE_SQUASH = Tween.EASE_OUT
const EASE_STRETCH = Tween.EASE_IN

func squash_with_easing(intensity: float):
	var tween = create_tween()
	tween.set_ease(EASE_SQUASH)
	var squash_scale = Vector2(1.2, 0.8) * intensity
	tween.tween_property(sprite, "scale", base_scale * squash_scale, 0.05)
	tween.tween_property(sprite, "scale", base_scale, 0.1)
```

**Volume Preservation Math:**
```gdscript
# Maintain constant area during deformation
# Area = width * height
# If width *= x, height *= 1/x

func calculate_volume_preserving_scale(primary_scale: float) -> float:
	# If stretching width by 1.5x, height becomes 0.666x
	return 1.0 / primary_scale

# For 2D: Area = x * y (constant)
# For 3D: Volume = x * y * z (constant)
# Example: stretch x by 2.0, compress y and z by sqrt(2) = 1.414
```
